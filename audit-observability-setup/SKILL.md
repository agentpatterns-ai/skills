---
name: audit-observability-setup
description: Audit an agent harness's observability setup — is a long/autonomous/multi-agent run legible enough to confirm it worked and catch it when it fails — OTel exporters, maxTurns + loop-detection circuit breakers, agent_id trace propagation, per-source context attribution, a wired regression gate, and a trajectory trail. Invoke when reviewing or building a harness that runs long, autonomous, or multi-agent work (settings.json hooks/telemetry, subagent instrumentation, CI eval gates, progress files). Skip when a single-shot/stateless prompt has nothing to instrument, or when judging whether a gate's grading is correct (use audit-verification-gates) rather than whether observability is wired.
user-invocable: true
version: "0.4.0"
usage: /audit-observability-setup [path-to-agent-config-or-repo]
---

# Audit Observability Setup

A long, autonomous, or multi-agent run is only safe if it is **legible** — instrumented enough to
confirm it worked and catch it when it fails. This audit checks the *plumbing of legibility*:
telemetry export, runtime circuit breakers, trace-identity propagation, context attribution, a wired
regression gate, a trajectory trail. Missing plumbing is silent — a cost spike, doom-loop, or false
"done" leaves no trace to trace ([observability-legible-to-agents](https://agentpatterns.ai/observability/observability-legible-to-agents/)).

**Stance — detect and recommend; read-only; apply nothing without confirmation.**
- **Runtime enforcement, not an instruction, stops a runaway.** A safety-critical ceiling
  (iterations, cost) must be `maxTurns` / a hook — runtime-enforced, unoverridable by model reasoning;
  the same ceiling as a natural-language rule the model is asked to obey stops nothing
  ([circuit-breakers](https://agentpatterns.ai/observability/circuit-breakers/)). Flag every
  instruction-level safety stop; recommend the deterministic equivalent.
- **Audit by run shape, not feature checklist.** Identity-propagation and per-source attribution are
  overhead on a single-agent / minimal-source run — the corpus calls them so. **Suppress exactly per
  the run-shape table in [`checks.md`](checks.md)** — the single source for what to skip; do not
  false-positive (see Scope).
- **Stay on the six pillars.** Observability has 25 auditor clusters; auditing all trips the
  compliance ceiling. Anchor on [`checks.md`](checks.md); push the long tail (dashboards, redaction,
  spend alerting, event-sourcing) to a future sibling.

## Input
- `path` (optional): an agent-config dir or repo. Default: scan the harness — `settings.json` /
  `managed-settings.json` (hooks, `maxTurns`, OTel env keys), agent/subagent defs + instrumentation
  code, MCP wiring, CI config (eval/regression gate), trail files (`claude-progress.txt`,
  `feature-state.json`, `init.sh`).

## Scope
Audits observability **wiring / legibility**: telemetry exported, breaker present, trace identity
propagated, gate runs and joins the trace, trail present. **Out of scope:** whether
a gate's *grading is correct* — outcome-vs-path grading, judge precision, generator≠grader (use
`audit-verification-gates`); application source-code review; live runtime monitoring (static config
audit). **Run-shape suppression:** classify the shape in
step 1, then suppress exactly per the run-shape table in [`checks.md`](checks.md) — the single
source for what to skip (a one-shot/stateless target is out of dispatch scope entirely — see Skip).
Every check the table does not suppress applies to every in-scope run.

## Procedure
1. **Classify the run shape.** Long/autonomous? Multi-agent (fan-out)? How many context sources?
   Sets which checks fire vs. suppress (Scope).
   Done when the run shape and each check's fire/suppress mark are recorded.
2. **Run the detectors** in [`checks.md`](checks.md) — OBS-1…OBS-10, each with sub-checks.
   Record present / partial / absent; do not collapse an ambiguous partial to a false "No".
   Done when every OBS check and sub-check has a recorded state — no blanks.
3. **Flag each gap** with severity (below), the corpus "why", and the lesson "how" (remediation).
   Done when every gap carries a severity, a why, and a lesson.
4. **Name the cheapest deterministic fix** per finding — prefer a runtime/hook control over a
   prompt; for OBS-4, name the runtime primitive that replaces the instruction.
   Done when every finding names its fix.
5. **Report** with the template below.
   Done when the Coverage table, Findings, Smallest high-impact change, and Suppressed sections are filled.

## Detectors
The ten detectors (OTel export, circuit breaker, loop hook, runtime-vs-instruction, dual-surface
`agent_id`, off-protocol egress, context attribution, regression-gate wiring, trajectory trail,
agent-legible signals) plus sub-checks and severity live in [`checks.md`](checks.md) — load when
auditing.

## Output template
```
# Observability audit — <target>   (run shape: long / multi-agent / N sources)

## Coverage
| Pillar | Check | State | Severity |
|---|---|:--:|:--:|
| Telemetry export        | OBS-1 | Absent  | High |
| Runtime circuit breaker | OBS-2 | Absent  | High |
| Loop detection          | OBS-3 | Absent  | High |
| Trajectory trail        | OBS-9 | Partial (feature-state flipped pre-verify) | High |
| Regression gate wired   | OBS-8 | Absent  | Med  |
| Trace propagation       | OBS-5 | n/a — single-agent (suppressed) | — |

## Findings
| Sev | Check | What's missing (why it bites) | Deterministic fix | Lesson |
|---|---|---|---|---|
| High | OBS-2 | No `maxTurns` on the open-ended coder — a stuck loop burns the whole budget | add `maxTurns` to the agent frontmatter (runtime-enforced) | breaking-the-loop |
| High | OBS-9 | `feature-state.json` set `passes:true` at code-write time, before any test ran | flip the flag only post-verification; write a progress file read-at-start | leaving-a-trail |

**Smallest high-impact change:** <the one wire to add first.>
**Suppressed (run shape):** <which checks, and why they don't apply.>
```
Severity: **High** = missing runtime safety control (breaker/loop), telemetry blackout, or a trail
that lies (premature "done"); **Med** = wired-but-incomplete pillar (gate runs but verdict not
trace-coupled; attribution present but lying denominator); **Low** = legibility tightening.

## Related / pairing
- Seam with **`audit-verification-gates`**: this checks the gate is *wired and trace-coupled*;
  that checks the gate's *grading is correct*. Reciprocal Skip clauses keep dispatch unambiguous.
- Corpus: [agent-observability-otel](https://agentpatterns.ai/observability/agent-observability-otel/),
  [loop-detection](https://agentpatterns.ai/observability/loop-detection/),
  [subagent-otel-trace-correlation](https://agentpatterns.ai/observability/subagent-otel-trace-correlation/),
  [context-usage-attribution](https://agentpatterns.ai/observability/context-usage-attribution/),
  [trajectory-logging-progress-files](https://agentpatterns.ai/observability/trajectory-logging-progress-files/),
  [traces-need-feedback-to-power-learning](https://agentpatterns.ai/observability/traces-need-feedback-to-power-learning/).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- A safety-critical stop must be **runtime-enforced** (`maxTurns` / hook) — an instruction asking the
  model to stop is not a control.
- **Audit by run shape:** suppress only per the run-shape table in [`checks.md`](checks.md) — never
  improvise suppression; do not manufacture findings.
- Scope to wiring; defer gate-*correctness* to `audit-verification-gates`. Read-only — recommend,
  apply nothing without confirmation.
