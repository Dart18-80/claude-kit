---
name: llm-app-patterns
description: Production patterns for integrating LLMs into applications — provider abstraction, prompts as versioned code, structured outputs validated with schemas, tool/function calling, streaming, timeouts and retries, prompt caching, cost and token tracking, prompt-injection defenses, PII handling, and observability. Use when adding an LLM feature to a backend or frontend, calling OpenAI/Anthropic/Gemini/local-model APIs, or reviewing code that does.
metadata:
  origin: claude-kit (original)
---

# LLM Application Patterns

How to put an LLM call inside a real product without it becoming the least reliable part of the system.

## When to Activate

- Adding a feature that calls an LLM API (chat, summarization, extraction, classification, generation)
- Designing the module or service that wraps model calls
- Reviewing code with raw SDK calls scattered through controllers or components
- Debugging flaky outputs, timeouts, runaway cost, or malformed JSON from a model

## Core Principle

An LLM call is a **slow, expensive, non-deterministic, remote dependency that returns untrusted text**. Treat it like a payment gateway, not like a pure function:

- isolate it behind an interface,
- validate everything it returns,
- bound its time and cost,
- log enough to replay any call.

## 1. One Gateway Module

All model calls go through one module (`llm/` or `ai/`). Controllers, services, and UI never import a provider SDK directly.

```text
src/ai/
  client.ts          # provider SDK init, timeouts, retries, tracing
  prompts/           # prompt templates + versions
  schemas/           # output schemas (zod / pydantic)
  tasks/             # one function per use case: summarizeTicket(), extractInvoice()
```

- Each **task function** has a typed input, a typed output, and owns its prompt, model choice, and schema.
- Model IDs come from config, never hardcoded in task code, so they can be swapped per environment.
- If you need several providers, abstract at the *task* level (input → typed output), not at the raw chat-message level. Provider-specific features (prompt caching, tool formats) leak through low-level abstractions anyway.
- Check the provider's current docs and SDK version before writing calls. Model names, parameters, and SDK methods change often.

## 2. Prompts Are Code

- Keep prompts in files next to the task, under version control, with a `PROMPT_VERSION` constant that is logged with every call.
- Structure: system prompt (role, rules, output contract) → stable context (docs, examples) → variable input last. Stable content first also maximizes prompt-cache hits.
- Put user/third-party content inside clear delimiters (XML-style tags work well) and tell the model it is data, not instructions.
- Few-shot examples should match the real input distribution, including edge cases, not only happy paths.
- Changing a prompt is a behavior change: it needs an eval run (see `llm-evals`) like any code change needs tests.

## 3. Structured Output, Always Validated

When code consumes the output, ask for structured data and **validate it with a schema**.

```python
class InvoiceFields(BaseModel):
    vendor: str
    total: Decimal
    currency: Literal["USD", "GTQ", "EUR"]
    due_date: date | None

result = llm.extract(task="invoice", text=doc, schema=InvoiceFields)  # returns InvoiceFields or raises
```

- Prefer the provider's native structured-output / JSON-schema / tool-calling mode over "please answer in JSON" prose.
- Validate anyway (Zod / Pydantic). Native modes reduce but do not eliminate bad output.
- On validation failure: retry **once** with the validation error appended, then fail with a typed error. No infinite repair loops.
- Use enums and constrained fields instead of free text whenever the downstream code branches on the value.
- Make "unknown" a valid answer (`null`, `"unknown"`) so the model is not forced to hallucinate a value.

## 4. Tool / Function Calling

- Tool names and descriptions are prompts: say *when* to use the tool, not just what it does.
- Validate tool arguments with the same schema library before executing.
- Tools that change state (send email, charge card, delete rows) require: authorization checks on the **end user's** permissions, idempotency keys, and, for high-impact actions, human confirmation.
- Return compact, model-readable results (IDs, short summaries, explicit errors), not raw 5k-line payloads.
- Cap the tool-call loop (max iterations / max tokens) and log every step. For full agents, see the `stack-agents` plugin.

## 5. Latency, Timeouts, Retries

- Set an explicit timeout on every call. SDK defaults are often minutes.
- Retry only retryable errors (429, 5xx, network) with exponential backoff and jitter, honoring `retry-after`. Never retry 400-class validation errors.
- Stream responses for anything user-facing that takes more than ~1–2 s; show partial output or progress.
- Long or batchable jobs (bulk classification, document processing) go to a queue / background worker, not the request path. Use provider batch APIs when latency doesn't matter; they are usually cheaper.
- Have a degraded mode: cached answer, simpler model, or a clear "AI unavailable" state. The feature must not take the whole page down.

## 6. Cost and Tokens

- Log `input_tokens`, `output_tokens`, cached tokens, model, and computed cost for every call, tagged by task and tenant/user.
- Set `max_tokens` (output cap) on every call.
- Enable prompt caching for long, stable prefixes (system prompt, documents, tool definitions) where the provider supports it.
- Route by difficulty: small/fast model for classification and extraction, large model only where evals show it's needed.
- Per-user and per-tenant budgets / rate limits for any endpoint that end users can trigger.
- See `cost-aware-llm-pipeline` (this plugin) for routing and budget patterns.

## 7. Security

- **Prompt injection is unsolved — design for containment.** Any text from users, emails, web pages, files, or tool results can contain instructions. Limit what the model can *do* (least-privilege tools, no secrets in context, confirmation for side effects) rather than relying on "ignore malicious instructions" in the prompt.
- Never put API keys, internal URLs, or other tenants' data into prompts.
- Treat model output as untrusted: escape before rendering as HTML/Markdown, never `eval` it, parameterize any SQL built from it, validate URLs before fetching.
- PII: minimize what you send; redact where possible; check the provider's data-retention terms; don't log full prompts containing PII in plaintext.
- Keys live server-side only. A browser or mobile app never calls a provider directly with a secret key.

## 8. Observability

Log for every call: request ID, task name, prompt version, model, latency, token usage, cost, finish/stop reason, validation result, and (subject to PII rules) inputs and outputs or a pointer to them.

- Trace multi-step flows (retrieve → generate → validate) as one trace with spans.
- Collect user feedback (thumbs, edits, rejections) linked to the call ID. It is your best source of eval cases.
- Alert on: error rate, validation-failure rate, p95 latency, cost per day, and sudden shifts in output length.

## 9. Testing

- **Unit tests** never hit the real API: mock the gateway and assert on prompt construction, schema handling, retries, and fallbacks.
- **Contract tests** with recorded responses (fixtures) for parsing and edge cases (empty, refusal, truncated by `max_tokens`).
- **Evals** for quality, run on prompt or model changes — see `llm-evals`.

## Review Checklist

- [ ] No provider SDK imports outside the AI gateway module
- [ ] Model ID and parameters come from config
- [ ] Prompt is versioned; version is logged
- [ ] Untrusted content is delimited and treated as data
- [ ] Output validated with a schema; one bounded retry on failure
- [ ] Timeout, `max_tokens`, and retry policy set explicitly
- [ ] State-changing tools check end-user authorization and are idempotent
- [ ] Output escaped / sanitized before rendering or executing
- [ ] Tokens, cost, latency logged per task; budgets for user-triggered endpoints
- [ ] Degraded mode when the provider fails
- [ ] Tests mock the gateway; evals exist for quality-critical tasks

## Related

- `llm-evals`, `cost-aware-llm-pipeline`, `regex-vs-llm-structured-text` (this plugin)
- `rag-patterns` (stack-rag plugin) when answers must be grounded in your own data
- `agent-development` (stack-agents plugin) when the model plans and runs multi-step tool loops
