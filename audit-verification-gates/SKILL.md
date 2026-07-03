---
name: audit-verification-gates
description: Audit the completion-gating architecture of an agent harness — the hooks, ledgers, graders, checklists, and red-green constraints that decide when an agent may declare "done" — and flag every gate that trusts self-report, grades the execution path, or is gameable instead of being mechanically anchored to deterministic external state. Invoke when reviewing or hardening how a harness blocks completion (Stop / SubagentStop / PostToolUse completion hooks, verification ledgers, pre-completion checklists, TDD "do not change the tests" constraints, eval graders) — before trusting an agent's "all tests pass" / "done". Skip when the concern is whether you can SEE or diagnose the run at all (tracing, logs, metrics, exit-code plumbing) — that is audit-observability-setup; or when auditing security / exfiltration architecture — use audit-lethal-trifecta.
user-invocable: true
version: "0.2.0"
usage: /audit-verification-gates [path-to-agent-config-or-repo]
---

# Audit Verification Gates

Agents optimize for task **completion, not correctness** — without a gate, an agent declares success
after partial work, an uninvestigated failing test, or code that compiles but misses the requirement
([pre-completion-checklists](https://agentpatterns.ai/verification/pre-completion-checklists/)).
One question of a harness: **is "done" blocked until verification has actually run and passed against
deterministic external state — and can that gate be gamed?** A gate that trusts the agent's narration,
grades the path it took, or that the agent can rewrite is not a gate.

**Stance — detect and recommend; read-only; apply nothing without confirmation.**
- **The costliest gap first: a prose gate is not a gate.** "Be thorough / check your work" in a prompt
  gets ~70–90% compliance; a Stop/PostToolUse hook or middleware that runs *outside* the model's turn
  gets ~100% because context pressure can't skip it ([pre-completion-checklists, *Implementation
  Options*](https://agentpatterns.ai/verification/pre-completion-checklists/)). Recommend the mechanical
  control, not stronger wording.
- **Anchor to observable external state.** Tie completion to test/build exit codes, `git diff`, schema
  state — not a transcript regex or "status: done". Self-reported verification is unfalsifiable
  ([verification-ledger](https://agentpatterns.ai/verification/verification-ledger/)).
- **Respect the floor — don't manufacture ceremony.** Ledgers, golden journeys, and macro-evals are
  theater below documented floors (throwaway/stateless work, <~1,000-trace clustering, <~70% judge
  precision). A finding that demands a heavy gate where the floor isn't met is itself a false positive
  (VG-11). A natural-language guardrail documents intent; it does not enforce.

## Input
- `path` (optional): agent-config dir or repo. Default scan: `.claude/settings.json` hooks (`Stop`,
  `SubagentStop`, `PostToolUse`); `CLAUDE.md` / system-prompt "completion gate" sections; agent /
  sub-agent defs; CI / pre-commit / Makefile (deterministic checks + what completion is wired to);
  verification ledger (SQL / `verification.json` / structured-output schema); eval & grader code;
  `RELIABILITY.md` golden journeys + `feature_list.json`; red-green / TDD workflow defs.

## Scope
Audits the **completion-gating setup / architecture** — statically. **Out of scope:** run
*visible / diagnosable* (tracing, logs, metrics) → `audit-observability-setup`; security / exfiltration
architecture → `audit-lethal-trifecta`; live runtime monitoring (this is a static config audit). **Shared
seam (golden-journey failure signals, semantic exit codes):** the *same* artifact appears in both audits
— split by what the finding is about: concern = signal *visible/diagnosable* → observability; concern =
signal *gating completion* → here.

## Procedure
1. **Map the completion path.** For each agent / sub-agent, find what produces the "done" signal and what
   (if anything) intercepts it. Audit each path separately — sub-agents inherit no gate unless wired.
   Done when every path has its done-signal producer and interceptor recorded.
2. **Classify each gate** with the detectors in [`checks.md`](checks.md) (VG-1…VG-11): mechanical
   (hook/middleware/CI) or prose? Anchored to external state or narration? Specific or vague? Structured
   ledger or prose? Execution separated from recording? Verifier ≥ generator? Checkpointed or end-only?
   Outcome- or path-graded? Tests locked? Restart-clean? — then run the **floor guard (VG-11)**.
   Done when every gate has a verdict per detector VG-1…VG-11.
3. **Flag each weak/absent/gameable gate**, with severity (below). Mark partial states (a hook that fires
   but reads self-report) — do not collapse an ambiguous gate to a false "pass".
   Done when every flagged gate carries a severity and partial states are marked.
4. **Recommend the cheapest deterministic fix** per finding — a hook over a prompt, a ledger query over a
   prose summary, an outcome grader over a path grader — and only where the floor is met.
   Done when every finding names one deterministic fix.
5. **Report** with the template below. Read-only: apply nothing without confirmation.
   Done when the filled template covers every finding.

## Checks
The eleven detectors (VG-1…VG-11) — each with Flags / Why (cited corpus page) / Fix (lesson) — live in
[`checks.md`](checks.md); load it when auditing. Index: VG-1 prose-only gate · VG-2 not state-anchored ·
VG-3 vague items · VG-4 prose not ledger · VG-5 gameable / non-separated · VG-6 weak verifier / LLM-judge
· VG-7 end-only / no baseline · VG-8 path-based grading · VG-9 mutable tests · VG-10 no restart-clean /
non-grep-able signal · VG-11 below-floor ceremony (false-positive guard).

## Output template
```
# Verification-gate audit — <target>

## Completion gates
| Path / artifact | Mechanism | State-anchored? | Gameable? | Verdict |
|---|---|:--:|:--:|:--:|
| main agent (CLAUDE.md "Completion Gate") | prose only | No (self-report) | — | FAIL (VG-1, VG-2) |
| grader.py | path assert `tool_calls==[…]` | No | — | FAIL (VG-8) |
| red-green workflow | no "don't change tests" | n/a | Yes | FAIL (VG-9) |

## Findings
| Severity | ID | Path | What's wrong | Cheapest deterministic fix |
|---|---|---|---|---|
| High | VG-1/2 | main agent | "Be thorough; check your work" + self-reported "all tests pass" — no hook, not state-anchored | Stop/PostToolUse hook gating on exit code + a baseline/after ledger (the-pre-completion-checklist, the-verification-ledger) |
| High | VG-8 | grader.py | asserts the tool-call sequence, not the end-state | grade outcome: pytest exit code / schema state (grade-the-outcome) |
| High | VG-9 | red-green | agent may edit tests to pass them | lock tests in green; confirm red first (red-green-for-agents) |

**Passing gates:** <which, and the mechanism that makes them hold.>
**Floor note (VG-11):** <any gate NOT demanded because its floor isn't met — e.g. throwaway/stateless.>
**Smallest high-impact change:** <the one gate to make mechanical first.>
```
Severity: **High** = completion can be declared without deterministic proof, or the gate is gameable
(VG-1/2/4/5/8/9); **Medium** = gate exists but weak/partial (VG-3/6/7/10); **Low** = hardening where no
live gap exists. A VG-11 over-demand is **not** a finding — drop it.

## Related
- Sibling to **`audit-lethal-trifecta`** (security architecture) and **`audit-instruction-file`**
  (instruction prose) — same audit family, different target: this owns **correctness gating**.
- Reciprocal skip with **`audit-observability-setup`**: *can we SEE the run?* → observability; *is
  completion BLOCKED until correctness is proven?* → here. Corpus citations per check are in `checks.md`.

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue — interactive: recommend and confirm first (never auto-file); autonomous: self-file — shaped like `track-backlog` (title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table). Each finding, in both the report **and** the filed issue, carries its **Fix → lesson** link — resolved by check ID via [`checks.md`](checks.md), which stays the link's canonical home (citation standard).

## Critical rules (read last)
- A gate that trusts narration, grades the path, or the agent can rewrite is **not** a gate — anchor it
  to deterministic external state and make recording independent of the producer.
- Prefer the **mechanical** control (hook / CI / ledger query / outcome grader) over stronger prompt wording.
- **Respect the floor (VG-11)** — don't demand ledgers, journeys, or macro-evals on throwaway/stateless
  work or sub-floor evals. Read-only; recommend, apply nothing without confirmation.
