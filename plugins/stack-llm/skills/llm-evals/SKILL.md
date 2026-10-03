---
name: llm-evals
description: Evaluation-driven development for LLM features, RAG systems, and agents — building golden datasets, choosing graders (exact match, code checks, LLM-as-judge with rubrics, human review), offline regression suites in CI, online metrics, and comparing prompts or models. Use when changing a prompt or model, deciding whether an LLM feature is good enough to ship, or when outputs regress and nobody can measure by how much.
metadata:
  origin: claude-kit (original)
---

# LLM Evals

If you can't measure it, every prompt change is a guess. Evals are the test suite for non-deterministic code.

## When to Activate

- Before shipping an LLM, RAG, or agent feature
- Before changing a prompt, model, temperature, retrieval setting, or tool definition
- When users report "it got worse" and there's no baseline
- When choosing between models or providers on quality vs cost

## 1. Define "Good" First

Write down, per task, before writing prompts:

- **What a correct output looks like** (fields, facts, format, tone)
- **Unacceptable failures** (hallucinated numbers, wrong customer data, unsafe content, leaking the system prompt)
- **The metric and the bar** to ship (e.g. "≥ 95% field accuracy on the golden set, 0 critical failures, p95 < 4 s, cost < $0.01/call")

## 2. Build the Dataset

- Start with **20–50 real examples**, not 1,000 synthetic ones. Real user inputs (anonymized), support tickets, and documents.
- Cover: typical cases, edge cases (empty, very long, multilingual, malformed), adversarial inputs (prompt injection, off-topic), and every **bug that ever reached production**.
- Each case: `id`, `input`, `expected` (exact value, reference answer, or rubric notes), `tags` (slice: language, customer type, difficulty).
- Store it in the repo (`evals/datasets/*.jsonl`) and version it. Never tune prompts on the same cases you report final numbers on: keep a held-out split.
- Grow it continuously from production feedback: thumbs-down, user edits, and escalations become new cases.

## 3. Choose Graders (Cheapest That Works)

| Grader | Use for | Notes |
|---|---|---|
| Exact / normalized match | Classification, extraction of IDs, enums, numbers | Deterministic, free. Prefer whenever possible |
| Code checks | Schema validity, JSON parses, length limits, regex, "contains citation", "no PII" | Deterministic; run on every case |
| Reference-based similarity | Short factual answers | Use carefully; surface similarity ≠ correctness |
| LLM-as-judge with rubric | Open-ended quality: faithfulness, helpfulness, tone, completeness | Calibrate against humans; see below |
| Human review | Calibrating judges, high-stakes outputs, final ship decision | Slow; sample strategically |

### LLM-as-judge rules

- Judge **one criterion per call** with a concrete rubric and a small scale (pass/fail or 1–3), not a vague 1–10 "quality" score.
- Ask for a short reasoning *before* the verdict, and return structured output.
- Use pairwise comparison (A vs B) when comparing two prompts or models; randomize order to avoid position bias.
- Prefer a different, strong model as the judge than the one being evaluated.
- **Calibrate:** label ~30–50 cases by hand, measure agreement with the judge, and fix the rubric until agreement is high. An uncalibrated judge is a random number generator with opinions.

## 4. Task-Specific Metrics

- **Extraction / classification:** per-field accuracy, precision/recall per class, confusion matrix.
- **RAG:** retrieval recall@k and MRR (did the right chunk come back?), then faithfulness (is every claim supported by the context?) and answer correctness. Evaluate retrieval and generation **separately** so you know which one broke. See `rag-patterns`.
- **Agents:** task success rate, steps/tool calls per task, cost per task, unsafe-action rate. Run each task several times: agents are high-variance, so report pass rate over N runs (pass@k / pass^k), not a single run.
- **All:** latency p50/p95, cost per call, validation-failure rate, refusal rate.

## 5. Run Evals Like Tests

```text
evals/
  datasets/support_triage.jsonl
  graders/                 # code graders + judge prompts
  run_eval.py | run-eval.ts
  results/                 # gitignored, or summarized in the PR
```

- One command runs a suite and prints a table: overall score, per-slice scores, failures with diffs, cost of the run.
- **Regression gate in CI** for prompt/model/retrieval changes: block merges when the score drops below the baseline or any critical case fails. Keep CI suites small and fast. Run the full suite nightly or before releases.
- Pin model versions in eval runs and record them with results.
- Set temperature to 0 (or fixed seeds) where the provider supports it to reduce noise. Still, re-run borderline comparisons.
- Compare against the **baseline**, not against perfection. Report deltas: `+3.1% field accuracy, -12% cost, 2 new failures (ids…)`.

## 6. Online Evaluation

Offline evals catch regressions. Production tells you what you missed.

- Log inputs/outputs (respecting PII rules) with prompt version and model.
- Track user signals: acceptance, edits, retries, thumbs, escalation to human.
- Sample production traffic for periodic judge or human review.
- For risky changes, ship behind a flag, shadow-run or A/B test, and compare metrics before full rollout.

## 7. Error Analysis Loop

1. Read 20–50 failures by hand before changing anything.
2. Cluster them (retrieval miss, wrong format, outdated info, instruction ignored, injection…).
3. Fix the **largest cluster** first. The fix may not be the prompt: it can be data, retrieval, a tool, or a code check.
4. Add the failures to the dataset, re-run, and compare against the baseline.

## Anti-Patterns

- "Vibe checking" a few examples in a playground and shipping
- Only synthetic data generated by the same model you're testing
- A single aggregate score that hides a collapsing slice
- Optimizing the judge's score instead of the user outcome (Goodhart)
- Evals that never run because they're slow or manual

## Review Checklist

- [ ] Success criteria and ship bar written down per task
- [ ] Versioned dataset with real, edge, adversarial, and past-bug cases
- [ ] Deterministic graders first; LLM judges have rubrics and were calibrated
- [ ] Retrieval and generation evaluated separately (RAG)
- [ ] One-command runner with per-slice results and cost
- [ ] CI regression gate on prompt/model/retrieval changes
- [ ] Production feedback flows back into the dataset

## Related

- `llm-app-patterns` (this plugin)
- `rag-patterns` (stack-rag plugin)
- `loop-design-check` (stack-agents plugin) for machine-decidable goals in agent loops
