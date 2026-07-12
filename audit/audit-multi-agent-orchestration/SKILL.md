---
name: audit-multi-agent-orchestration
description: Audit an existing multi-agent orchestration's coordination mechanics — typed vs prose handoffs, shared mid-run state, verifier/producer role separation, and loop bounds — and flag the structural smells that make multi-agent systems fail. Invoke when reviewing or hardening a harness that already runs more than one agent (orchestrator-worker, fan-out/synthesis, lead-and-teammates, generator-critic loops), including whether the verifier role is independent of the producer. Skip when deciding whether to go multi-agent at all or which topology (use architecture-committee), auditing the general completion-gating architecture — hooks, ledgers, graders, red-green anchoring (use audit-verification-gates), auditing security legs / prompt-injection (use audit-lethal-trifecta), or a single instruction file's prose (use audit-instruction-file).
user-invocable: true
version: "0.2.1"
usage: /audit-multi-agent-orchestration [path-to-harness-or-repo]
---

# Audit Multi-Agent Orchestration

Multi-agent failures come from **missing structure at the wiring between agents, not from model
capability** ([typed-schemas-at-agent-boundaries](https://agentpatterns.ai/multi-agent/typed-schemas-at-agent-boundaries/)).
Audits an *already-built* orchestration — sub-agent definitions plus the glue that spawns, hands off,
gates, and merges — for the four structural smells that defeat its parallelism/correctness:
**untyped/prose handoffs, shared mid-run state, no independent verifier gate on "done", unbounded loop.**

**Stance — detect and recommend; read-only static-config audit; apply nothing without confirmation.**
- **Audit the wiring between agents, per boundary** — handoff schemas, shared surfaces, gate position,
  loop caps, spawn count — not each agent's prose (that is `audit-instruction-file`).
- **Enforcement beats instruction.** A "please get approval first" or "never declare done early" prompt
  line is *not* a gate — prompt-discipline gates are routinely violated under load; prefer a control
  enforced by the permission model / structure ([lead-teammate-plan-approval-handshake](https://agentpatterns.ai/multi-agent/lead-teammate-plan-approval-handshake/)).
- **"Done" is not the producer's to declare.** LLMs prefer their own output when grading it; self-refinement
  amplifies the bias ([Xu et al. 2024, arXiv:2402.11436](https://arxiv.org/abs/2402.11436)) — put a separate
  read-only verifier on the critical path ([verify-gated-completion](https://agentpatterns.ai/multi-agent/verify-gated-completion-admission-control/)).
- **Ceremony is a non-finding.** Typed schemas/gates pay off only when work *genuinely crosses agent
  boundaries*; on a solo, single-stage, low-stakes task they add friction — note it, do not demand more
  structure ([agent-handoff-protocols, *When This Backfires*](https://agentpatterns.ai/multi-agent/agent-handoff-protocols/)).

## Input
- `path` (optional): harness dir or repo. Default scan: `.claude/agents/*.md` (sub-agent `tools`,
  `model`, any `## Returns`/`## Output`/`returns:`/`output_schema:` handoff contract); orchestration
  scripts that spawn workers and merge results; any verifier/critic step and its position vs "done";
  any refinement/debate loop and its termination condition; shared state surfaces workers touch
  mid-run (scratch files, shared doc, message bus).

## Scope
Audits **coordination correctness** of an orchestration that already exists. Verifier checks test one
property only — "done" is decided by a verifier **independent of the producer** and on the inter-agent
critical path. **Out of scope:** the *decision* whether to go multi-agent or which topology (use
`architecture-committee`); general completion-gating architecture — hooks, ledgers, graders, red-green
anchoring (use `audit-verification-gates`); *security* legs — private data + untrusted input + egress
(use `audit-lethal-trifecta`); a single instruction file's prose attention/density (use
`audit-instruction-file`); live runtime monitoring (static audit only).

## Procedure
1. **Map the topology.** Identify the pattern (orchestrator-worker, fan-out/synthesis,
   lead-and-teammates, generator-critic); enumerate each agent boundary, shared surface, gate, loop.
   Done when every boundary, surface, gate, and loop is listed — none omitted.
2. **Decide if structure is warranted.** Solo/single-stage/low-stakes work → heavy schemas/gates are
   ceremony; record a non-finding, skip the structure checks for that path.
   Done when each path carries a warranted or non-finding mark.
3. **Run the detectors** in [`checks.md`](checks.md) per boundary/surface/gate/loop. Classify a verifier
   carefully: present-but-mis-wired (MA-V2) is *not* "has a gate."
   Done when every boundary/surface/gate/loop × check has a recorded verdict.
4. **Recommend the cheapest enforced control** — permission-model / structural over prompt — per
   finding; for a removed/added control, name the new target it shifts attention to (e.g. token
   resolver, synthesis step). Done when every finding names an enforced control.
5. **Report** with the template below.
   Done when the Topology, Findings, Ceremony, and Smallest high-impact change sections are filled.

## Detectors
Ten detectors — handoffs (MA-H*), state/topology (MA-S*), gates/verification (MA-V*), loops (MA-L*),
cost/scaling (MA-C*) — each with Flags / Why (cited) / Fix, plus the ceremony backfire guard, live in
[`checks.md`](checks.md). Load it when auditing.

## Output template
```
# Multi-agent orchestration audit — <target>

## Topology
Pattern: orchestrator-worker. Boundaries: orchestrator→worker (x3), worker→synthesis.
Gates: none on "done". Loops: refine-loop (uncapped). Shared surfaces: scratch.md (shared).

## Findings
| ID | Severity | Where | Smell | Why it fails (source) | Fix (enforced, not prompt) |
|---|---|---|---|---|---|
| MA-H1 | High | worker→synthesis | prose handoff, no schema | field extraction non-deterministic; ambiguity compounds | declare `## Returns` JSON: done/found/needs-attention/unresolved |
| MA-S1 | High | workers↔scratch.md | shared mid-run state | "any state sharing between workers during execution is a design smell" | workers return to orchestrator only; non-overlapping file sets |
| MA-S2 | High | synthesis | `"\n".join(outputs)` | pays the ~15x multi-agent token premium for zero quality gain | reasoning merge: reconcile conflicts, weight reliability |
| MA-V1 | High | "done" | producer self-declares | self-judgement bias amplified by self-refine | separate read-only verifier on path; fail-closed; packetize |
| MA-L1 | High | refine-loop | `while True:` | runs to budget exhaustion; reference loop is unbounded | hard `max_rounds` (start 3); skip on strong baseline |
| MA-C1 | Med  | orchestrator | fixed worker count in code | over-spawns on simple queries | move effort-scaling (1 / 2-4 / 10+) into the prompt |

**Ceremony (non-finding):** <paths where structure exceeds the boundary-crossing — note, don't flag.>
**Smallest high-impact change:** <the one enforced control to add first.>
```
Severity: **High** = a smell that loses information, races, or admits unverified "done"; **Medium** =
cost/scaling or advisory-only gaps; **Low** = tightening with no live failure.

## Related / pairing
- Runs *after* the build; sibling **`architecture-committee`** runs *before* it to decide whether to
  go multi-agent and which topology. Same family (`audit-lethal-trifecta`, `audit-instruction-file`),
  different target.
- Corpus: [orchestrator-worker](https://agentpatterns.ai/multi-agent/orchestrator-worker/),
  [agent-handoff-protocols](https://agentpatterns.ai/multi-agent/agent-handoff-protocols/),
  [emergent-behavior-sensitivity](https://agentpatterns.ai/multi-agent/emergent-behavior-sensitivity/),
  [evaluator-optimizer](https://agentpatterns.ai/agent-design/evaluator-optimizer/),
  [fan-out-synthesis](https://agentpatterns.ai/multi-agent/fan-out-synthesis/).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- **Untyped/prose handoffs, shared mid-run state, no independent verifier on "done", unbounded loops**
  are the four costliest smells — flag every one.
- **A prompt instruction is not a gate.** Recommend permission-model / structural enforcement.
- **Ceremony is a non-finding** — don't demand schemas/gates where work doesn't cross boundaries.
- Read-only: recommend, apply nothing without confirmation.
