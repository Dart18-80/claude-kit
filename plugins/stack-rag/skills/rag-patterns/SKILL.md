---
name: rag-patterns
description: Retrieval-Augmented Generation end to end — ingestion and parsing, chunking strategies, embeddings, vector stores (pgvector on Postgres/Supabase, dedicated vector DBs), hybrid search (vector + full-text), metadata filtering and multi-tenant isolation, reranking, context assembly with citations, freshness and re-indexing, and evaluating retrieval separately from generation. Use when building, debugging, or reviewing a RAG system, semantic search, or "chat with your documents" feature.
metadata:
  origin: claude-kit (original)
---

# RAG Patterns

RAG quality is mostly a **retrieval** problem. If the right passage never reaches the model, no prompt will fix the answer.

## When to Activate

- Building search or Q&A over company documents, tickets, code, or a knowledge base
- Choosing chunking, embeddings, or a vector store
- Answers are wrong, outdated, or uncited, and nobody knows whether retrieval or generation failed
- Adding multi-tenant document search

## First: Do You Need RAG?

- Small, stable corpus that fits comfortably in the context window → just put it in the prompt (with prompt caching).
- Structured data (orders, metrics) → query the database with tools/SQL, don't embed rows.
- Exact lookups (IDs, codes, names) → keyword search or filters beat embeddings.
- Large, changing, unstructured text → RAG.

## Pipeline Overview

```text
INGEST:  source → parse → clean → chunk → enrich metadata → embed → upsert (idempotent)
QUERY:   question → (rewrite) → hybrid retrieve (+ filters) → rerank → assemble context → generate with citations → validate
EVAL:    retrieval metrics (recall@k, MRR)  |  generation metrics (faithfulness, correctness)
```

## 1. Ingestion

- **Parsing quality matters most.** Garbage PDFs produce garbage chunks. Use layout-aware parsers for PDFs and tables; keep headings, lists, and table structure. Inspect parsed output by hand on a sample.
- Store a **document record** (id, source URL/path, title, owner/tenant, version or hash, updated_at, ACL) separately from chunks.
- Ingestion is **idempotent**: re-ingesting the same document (same content hash) is a no-op; a changed document replaces its old chunks atomically.
- Run ingestion as background jobs (queue), with retries and per-document status.

## 2. Chunking

- Start with **structure-aware chunks**: split by headings/sections, then by paragraphs, with a token cap (often ~300–800 tokens) and a small overlap (~10–15%).
- Never split mid-sentence, mid-table-row, or mid-code-block.
- Prepend context to each chunk before embedding: document title + section path (e.g. `Employee Handbook > Vacation > Carry-over`). This alone often improves retrieval a lot.
- Keep chunk ↔ document ↔ position links so you can show citations and expand to neighbors.
- Tune chunk size **with evals**, not intuition: too small loses context, too large dilutes the embedding.

## 3. Embeddings

- Pick a model by evaluating on **your** queries (multilingual matters: Spanish + English corpora need a multilingual model).
- Store the model name and dimension with each vector. **Changing the embedding model means re-embedding everything**: version your index.
- Use the same model and preprocessing for documents and queries (some models need different "query" vs "document" prefixes or input types; follow their docs).
- Batch embedding calls and cache by content hash.

## 4. Vector Store

- **Postgres + pgvector** (including Supabase) is a strong default when you already run Postgres: one database, transactions, joins with your data, row-level security for tenancy.
- Dedicated vector DBs make sense at very large scale or for specialized features. Don't start there by default.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE doc_chunks (
  id           bigserial PRIMARY KEY,
  document_id  uuid NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  tenant_id    uuid NOT NULL,
  content      text NOT NULL,
  heading_path text,
  position     int  NOT NULL,
  embedding    vector(1536) NOT NULL,          -- match your model's dimension
  fts          tsvector GENERATED ALWAYS AS (to_tsvector('simple', content)) STORED
);

CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON doc_chunks USING gin (fts);
CREATE INDEX ON doc_chunks (tenant_id);
```

- Use the distance operator that matches the index ops class (`<=>` for cosine with `vector_cosine_ops`).
- HNSW for most cases. Check recall after adding filters: very selective `WHERE` clauses can reduce how many results the approximate index returns. Consider iterative/filtered scan settings of your pgvector version or partitioning by tenant.
- Pick the text-search config (`'simple'`, `'spanish'`, `'english'`) for your corpus language.

## 5. Retrieval

- **Hybrid search** (vector + full-text/BM25, merged with Reciprocal Rank Fusion) beats pure vector search on names, codes, acronyms, and exact terms. Make it the default.
- **Always filter by tenant and permissions in the query itself**, never after retrieval in application code. The model must never see a chunk the user couldn't open.
- Retrieve generously (e.g. top 20–50), then **rerank** with a cross-encoder/reranker model down to the few (3–8) chunks that go into the prompt.
- Query rewriting helps for conversational follow-ups ("and for contractors?" → standalone question). Multi-query / HyDE are optional extras: add them only if evals show a gain.
- Metadata filters (date, product, doc type) from the UI or extracted from the question cut noise cheaply.

## 6. Context Assembly and Generation

- Put retrieved chunks in clearly delimited blocks with an ID and source: `<source id="3" title="..." url="...">...</source>`.
- Instruct: answer **only** from the sources, cite source IDs for each claim, and say "I don't know / not in the documents" when the answer isn't there. Make that option explicit.
- Treat retrieved text as **untrusted data**: documents can contain prompt-injection text. Don't give the generation step dangerous tools.
- Validate citations in code: every cited ID must exist in the provided context; strip or flag answers citing nothing.
- Order chunks by relevance (most relevant first). De-duplicate near-identical chunks.

## 7. Freshness and Operations

- Re-index on document change (webhooks, CDC, or scheduled sync). Delete chunks when documents are deleted or access is revoked.
- Track per document: last indexed time, embedding model version, chunk count, errors.
- Log per query: rewritten query, retrieved chunk IDs and scores, reranked IDs, final answer, citations. Without this you can't debug.

## 8. Evaluate Retrieval and Generation Separately

1. **Retrieval set:** questions with the chunk/document IDs that contain the answer. Metrics: recall@k, MRR, hit rate. Run this on every chunking, embedding, or retrieval change; it's cheap.
2. **Generation set:** questions with reference answers. Metrics: faithfulness (claims supported by context), answer correctness, citation accuracy, "I don't know" rate on unanswerable questions.
3. When an answer is wrong, check first whether the right chunk was retrieved. That tells you which half to fix.

See `llm-evals` (stack-llm plugin) for datasets, judges, and CI gates.

## Anti-Patterns

- Tuning prompts when retrieval recall is the real problem
- Fixed 512-character chunks that cut tables and sentences
- Filtering tenants/permissions after retrieval
- Dumping top-k raw chunks into the prompt without reranking or pruning
- Mixing vectors from different embedding models in one index
- No logging of which chunks produced an answer

## Review Checklist

- [ ] Parsing output was inspected; structure preserved
- [ ] Structure-aware chunks with title/section context; size tuned with evals
- [ ] Embedding model + version stored; re-index plan exists
- [ ] Hybrid retrieval + reranking; tenant/ACL filters inside the query
- [ ] Sources delimited, cited, and validated; "not found" is an allowed answer
- [ ] Ingestion idempotent; deletions and permission changes propagate
- [ ] Retrieval and generation evaluated separately

## Related

- Agent `rag-pipeline-reviewer` (this plugin) for reviewing an existing pipeline
- `postgres-patterns` (stack-postgres plugin) for indexing and RLS
- `llm-app-patterns`, `llm-evals` (stack-llm plugin)
