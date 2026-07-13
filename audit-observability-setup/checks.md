# audit-observability-setup — detectors

Loaded on demand by `audit-observability-setup`. Each item: ID / **Flags** / **Why** (cited corpus =
the "why") / **Fix** (cited lesson = the "how"). All detection is read-only. Before flagging, apply the
**run-shape suppression table at the end of this file** — it is the single source for what to skip.

---

## OBS-1 — No telemetry exporter enabled
- **Flags:** `CLAUDE_CODE_ENABLE_TELEMETRY` unset, or set with no `OTEL_METRICS_EXPORTER` /
  `OTEL_LOGS_EXPORTER` — cost / token-usage / trajectory data never leaves the host, so a cost spike
  or 429 storm cannot be traced post-hoc.
- **Why:** the built-in OTel exporter is the standing record; `prompt.id` correlates a prompt's many
  calls and cost values are surfaced as approximations ([agent-observability-otel](https://agentpatterns.ai/observability/agent-observability-otel/);
  [opentelemetry-agent-observability](https://agentpatterns.ai/standards/opentelemetry-agent-observability/)).
- **Fix:** enable telemetry + an exporter and an export interval ([leaving-a-trail](https://learn.agentpatterns.ai/observability/leaving-a-trail/)).

## OBS-2 — No runtime circuit breaker (`maxTurns` absent on open-ended work)
- **Flags:** an agent doing open-ended work with no runtime iteration cap (`maxTurns`) and no session
  cost budget — a stuck loop can burn the whole window / budget before anything halts it.
- **Why:** five signals warrant a break; the iteration limit is enforced by `maxTurns` at the runtime
  level ([circuit-breakers](https://agentpatterns.ai/observability/circuit-breakers/)).
- **Fix:** add `maxTurns` to the agent frontmatter (and a cost budget) ([breaking-the-loop](https://learn.agentpatterns.ai/observability/breaking-the-loop/)).

## OBS-3 — No loop detection wired
- **Flags:** no `PostToolUse` hook on `Edit|Write` tracking per-file edit count and injecting a nudge,
  and no doom-loop detector for identical tool-call/error pairs. The three layers — edit-count nudge →
  doom-loop stop → iteration cap — are absent.
- **Why:** three complementary layers protect against unproductive execution; the nudge states the
  observation factually and matches only on edit/write tools ([loop-detection](https://agentpatterns.ai/observability/loop-detection/)).
- **Fix:** add the `PostToolUse(Edit|Write)` edit-count hook + doom-loop detector ([breaking-the-loop](https://learn.agentpatterns.ai/observability/breaking-the-loop/)).

## OBS-4 — Instruction-level stop where runtime enforcement is required
- **Flags:** a safety-critical ceiling (iteration / cost stop) encoded as a natural-language rule the
  model is asked to obey, instead of `maxTurns` / a hook.
- **Why:** runtime enforcement cannot be overridden by model reasoning; instruction-level enforcement
  depends on the model policing itself — unreliable for safety-critical stops ([circuit-breakers,
  *runtime vs instruction*](https://agentpatterns.ai/observability/circuit-breakers/)).
- **Fix:** move the ceiling to `maxTurns` / a deterministic hook; keep the instruction only as a
  secondary error-rate signal the runtime can't detect ([breaking-the-loop](https://learn.agentpatterns.ai/observability/breaking-the-loop/)).
  *Seam:* OBS-4 covers **loop/cost safety ceilings** only; the general prose-vs-hook placement
  decision for any other rule is `audit-prompt-hook-placement`'s lane — hand off when the finding
  generalizes beyond a stop/budget ceiling.

## OBS-5 — Multi-agent harness without dual-surface `agent_id` propagation *(suppress if single-agent)*
- **Flags:** fan-out exists but `agent_id` is not emitted on **both** the outgoing HTTP header
  (`x-claude-code-agent-id`) and the OTel span attribute, and/or `parent_agent_id` is missing — so
  "all work by this subagent" and "which dispatch chain caused this 429" need an expensive span-tree
  walk.
  - **Sub-check (cardinality):** `agent_id` (per-instance, high-cardinality) used as a **metric
    label** → cardinality explosion. `agent.name` (bounded subagent *type*) is the metric-safe partner.
- **Why:** the header+span pair is load-bearing; the propagated attribute answers "what work this
  identity caused regardless of dispatch path" ([subagent-otel-trace-correlation](https://agentpatterns.ai/observability/subagent-otel-trace-correlation/)).
- **Fix:** emit `agent_id`/`parent_agent_id` on header and span; slice metrics by `agent.name`, drill
  by `agent_id` ([one-id-across-the-trace](https://learn.agentpatterns.ai/observability/one-id-across-the-trace/)).

## OBS-6 — Off-protocol egress drops agent identity *(suppress if single-agent)*
- **Flags:** raw `curl` / subprocess / fire-and-forget queue calls that produce a tool span but carry
  no `x-claude-code-agent-id` header — the downstream service sees an anonymous request. Note:
  `TRACEPARENT` is auto-inherited but carries the trace, **not** the identity.
- **Why:** documented propagation break — a Bash shell-out emits a tool span with no agent header;
  subprocess spans inherit the trace, not the agent id ([subagent-otel-trace-correlation, *Where the
  Propagation Breaks*](https://agentpatterns.ai/observability/subagent-otel-trace-correlation/)).
- **Fix:** lift the shell-out into an instrumented MCP tool, or wrap it with an explicit
  `x-claude-code-agent-id` header ([one-id-across-the-trace](https://learn.agentpatterns.ai/observability/one-id-across-the-trace/)).

## OBS-7 — No per-source context attribution / untrustworthy denominator *(suppress if minimal-source)*
- **Flags:** the harness exposes only a single "X% full" number (or a per-tool cut only) — an operator
  can't tell *which source* (rules / skills / MCP / subagents / history / cache) filled the window,
  the cut that maps to a remediation primitive.
  - **Sub-check (lying denominator):** a breakdown counting **input tokens only** (not input + output
    + cache) undercounts the budget — Claude Code showed ~20% full while at its limit.
- **Why:** categories must match remediation primitives, and a slice is trustworthy only when it sums
  every token `type` ([context-usage-attribution](https://agentpatterns.ai/observability/context-usage-attribution/)).
- **Fix:** break context down per source; confirm the denominator covers input + output + cache
  ([attributing-the-context](https://learn.agentpatterns.ai/observability/attributing-the-context/)).

## OBS-8 — No regression gate wired, or feedback not coupled to the trace
- **Flags:** no eval/regression gate runs in CI; **or** a gate runs but its verdict is not attached to
  the run/trace (no trace ID on the feedback record). *Presence + wiring only — grading correctness is
  `audit-verification-gates`' job; do not assess judge precision here.*
- **Why:** observability is descriptive; a trace becomes a learning corpus only when each one carries
  a verdict coupled at write time (e.g. `gen_ai.evaluation.result`) ([traces-need-feedback-to-power-learning](https://agentpatterns.ai/observability/traces-need-feedback-to-power-learning/)).
- **Fix:** wire a gate into CI and store its verdict on the run with the trace ID ([gates-that-catch-regressions](https://learn.agentpatterns.ai/observability/gates-that-catch-regressions/)).

## OBS-9 — No trajectory trail / premature-completion guard
- **Flags:** long/multi-session work with no progress file (read-at-start, write-at-end), no
  git-commit-per-task trail, and/or `feature-state.json` flipped to `passes:true` **before**
  verification — so a fresh window is blind to prior work and the model can declare "done" unconfirmed.
- **Why:** the trail is what carries state across windows; the state flag must flip only after a real
  check ([trajectory-logging-progress-files](https://agentpatterns.ai/observability/trajectory-logging-progress-files/)).
- **Fix:** add a progress file read-at-start + commit-per-task; flip `feature-state` only
  post-verification ([leaving-a-trail](https://learn.agentpatterns.ai/observability/leaving-a-trail/)).

## OBS-10 — Signals not legible to the agent (write-and-hope) + blind-agent trap
- **Flags:** the agent has code + test output but no log / metric / visual (MCP) signal to confirm the
  *system* behaved; and no handling for the failure mode where a down MCP server returns empty and the
  agent reads "no results" as "no errors."
- **Why:** without a legible signal the agent is running write-and-hope, and an MCP outage that returns
  empty reads as all-clear ([observability-legible-to-agents, *When This Backfires*](https://agentpatterns.ai/observability/observability-legible-to-agents/)).
- **Fix:** wire a log/metric/browser MCP signal the agent can read; treat empty MCP results as
  unknown, not clear ([write-and-hope](https://learn.agentpatterns.ai/observability/write-and-hope/)).

---

## Severity
- **High:** missing runtime safety control (OBS-2, OBS-3, OBS-4) or telemetry blackout (OBS-1), or a
  trail that lies — premature `passes:true` (OBS-9).
- **Med:** a wired-but-incomplete pillar — gate runs but verdict not trace-coupled (OBS-8);
  attribution present but lying denominator (OBS-7 sub-check); identity on span but not header (OBS-5/6).
- **Low:** legibility tightening with no live blind spot.

## Run-shape suppression (do not false-positive)
| Run shape | Suppress | Why |
|---|---|---|
| Single-agent, no fan-out | OBS-5, OBS-6 | nothing to correlate across agents — propagation is overhead ([subagent-otel-trace-correlation](https://agentpatterns.ai/observability/subagent-otel-trace-correlation/)) |
| Minimal context sources | OBS-7 | per-source attribution adds little when there is one source |
| One-shot / stateless | OBS-2, OBS-3, OBS-9 | no loop / trajectory to guard (the skill should not fire at all — see Skip) |

Every check not named in a matching row above (OBS-1, OBS-4, OBS-8, OBS-10 for all shapes;
OBS-2/3/9 for any in-scope shape) applies — this table is the **single source** for suppression.
