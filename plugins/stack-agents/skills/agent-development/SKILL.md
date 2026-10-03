---
name: agent-development
description: Designing and building AI agents and agentic workflows — when to use a fixed workflow vs an autonomous agent, core loop architecture, orchestration patterns (routing, orchestrator-workers, evaluator-optimizer, multi-agent), state and memory, human-in-the-loop, permissions and sandboxing, framework choice (Claude Agent SDK, OpenAI Agents SDK, LangGraph, Vercel AI SDK, Pydantic AI…), MCP for tools, tracing, and cost/step limits. Use when building an agent, a multi-step LLM pipeline, or reviewing one.
metadata:
  origin: claude-kit (original)
---

# Agent Development

An agent is an LLM in a loop that chooses actions (tool calls) based on what happened so far, until a goal is met. Most reliability problems come from too much autonomy, vague goals, and weak tools, not from the model.

## When to Activate

- Building an assistant that uses tools, browses, writes code, or operates on business systems
- Turning a multi-step LLM process into a pipeline or agent
- Choosing an agent framework or SDK
- Reviewing an agent that loops, stalls, overspends, or takes unsafe actions

## 1. Workflow or Agent?

Use the **least autonomy that solves the problem**:

| Shape | Use when | Example |
|---|---|---|
| Single LLM call | One step, clear input/output | Summarize a ticket |
| **Workflow** (code decides the steps) | Steps are known in advance | Classify → extract → validate → store |
| Router | Input type decides which workflow runs | Billing vs technical vs sales question |
| **Agent** (model decides the steps) | Steps depend on what is discovered along the way | Investigate a bug, research a topic, multi-system support case |

Workflows are cheaper, faster, and testable. Promote to an agent only when the path really can't be scripted.

## 2. The Core Loop

```text
goal + context
  └─► model → (tool calls?) ──yes──► execute tools (validated, authorized) → append results ─┐
                     │                                                                         │
                     no → final answer → verify against goal ◄─────────────────────────────────┘
stop when: goal verified | max steps | max cost/tokens | max time | unrecoverable error | needs human
```

Non-negotiables:

- **Hard limits:** max iterations, max tokens/cost, wall-clock timeout. Every loop has them.
- **Machine-checkable "done"** where possible (tests pass, record created, schema valid). See `loop-design-check`.
- **Explicit failure path:** the agent can stop and report "blocked because X" instead of improvising.

## 3. Tools Are the Interface

The quality of an agent is mostly the quality of its tools (see `agent-harness-construction`).

- Few, well-named tools with precise descriptions of *when* to use them and clear argument schemas.
- Tools return concise, structured results and **actionable errors** ("customer_id not found; search by email with find_customer").
- Prefer high-level tools (`refund_order(order_id, reason)`) over low-level ones (`http_request`) when the action is common and risky.
- Expose tools through **MCP** when they should be reusable across agents and clients (see `mcp-server-patterns`).
- Validate arguments and **authorize against the end user's permissions** inside the tool, never only in the prompt.

## 4. Orchestration Patterns

- **Prompt chaining:** fixed sequence with checks between steps.
- **Routing:** a classifier picks a specialized prompt/model/workflow.
- **Parallelization:** independent subtasks run concurrently; results are merged (or voted on for reliability).
- **Orchestrator–workers:** a lead agent splits work dynamically and delegates to workers with **isolated, minimal context**, then synthesizes.
- **Evaluator–optimizer:** one step generates, another critiques against explicit criteria, and they loop with a cap.
- **Multi-agent:** only when subtasks are truly separable and context isolation helps. It multiplies cost and failure modes: one good agent with good tools is usually better.

## 5. State and Memory

- **Conversation state:** the message/tool history for the current run. Trim or summarize old tool results, keep the goal and constraints pinned.
- **Run state:** persist checkpoints (step, partial results, pending approvals) in a DB so a run can resume after a crash or deploy. Long runs belong in a worker/queue, not an HTTP request.
- **Long-term memory:** store explicit facts/preferences (structured, with source and timestamp) and retrieve them selectively. Don't append everything forever. Memory is a data store with privacy and correctness obligations.
- Each run has an ID; all tool calls, model calls, and state transitions are logged under it.

## 6. Human in the Loop

- Classify actions by risk: **read** (auto), **reversible write** (auto + log), **irreversible / external / costly** (require approval).
- Approval requests show exactly what will happen (diff, recipient, amount) and resume the run from a checkpoint.
- Make it easy for a human to take over or correct mid-run.

## 7. Safety and Security

- **Prompt injection:** content from web pages, emails, documents, and tool results can carry instructions. Contain the blast radius: least-privilege tools, no secrets in context, approvals for side effects, and separate "reading untrusted content" from "taking privileged actions".
- **Sandbox** code execution and file access (containers, restricted FS, network allowlists).
- Credentials stay in the tool layer, scoped per user/tenant. The model never sees raw secrets.
- Rate-limit and budget per user; alert on runaway loops.

## 8. Framework Choice

Pick by need, and **check current docs**: these SDKs evolve quickly.

- **Provider agent SDKs** (e.g. Claude Agent SDK, OpenAI Agents SDK): fastest path to a capable tool-using agent with built-in loop, tools, and tracing; tied to one provider's strengths.
- **Graph/state-machine frameworks** (e.g. LangGraph): explicit states, checkpoints, human-in-the-loop, and complex branching; more code, more control.
- **App-level SDKs** (e.g. Vercel AI SDK for TypeScript, Pydantic AI for Python): typed tool calling and streaming inside web/backend apps; good for workflows and lighter agents.
- **Plain SDK + your own loop:** fine for simple agents. A loop with limits and logging is ~100 lines; don't add a framework you can't debug.

Whatever you choose: keep business logic in your own tools/services so you can switch frameworks without rewriting the domain.

## 9. Observability and Evaluation

- Trace every run: each model call (prompt version, model, tokens, latency) and tool call (args, result summary, duration, error) as spans.
- Metrics: task success rate, steps per task, cost per task, tool error rate, approval rate, human-takeover rate.
- Evaluate on a task suite run multiple times per case, since agents are high-variance. See `llm-evals` (stack-llm plugin).
- When an agent fails, find the **first wrong step** in the trace (bad tool choice, bad tool result, misread result, premature stop) and fix that layer. `agent-architecture-audit` gives a structured diagnosis.

## Review Checklist

- [ ] A fixed workflow was considered first; autonomy is justified
- [ ] Max steps, max cost/tokens, and timeout enforced in code
- [ ] "Done" is verifiable; there is an explicit blocked/failed exit
- [ ] Tools: few, well-described, validated, authorized per end user, with actionable errors
- [ ] Risky actions require approval; runs can pause and resume
- [ ] Untrusted content can't trigger privileged actions without a check
- [ ] Runs are traced end to end; success/cost metrics tracked
- [ ] Task-level evals exist and run on prompt/model/tool changes

## Related

- `agent-harness-construction`, `loop-design-check`, `agent-architecture-audit`, `mcp-server-patterns` (this plugin)
- `llm-app-patterns`, `llm-evals`, `cost-aware-llm-pipeline` (stack-llm plugin)
- Official plugins `agent-sdk-dev` and `mcp-server-dev` from `claude-plugins-official` for Claude Agent SDK projects and MCP servers
