---
name: audit-prompt-hook-placement
description: Audit the enforcement-medium decision in an agent harness — does each stated rule belong in prose (judgment) or a deterministic gate (PreToolUse hook / CI / tool-restriction)? — and flag misplaced absolute rules and unsound hooks. Invoke when deciding where a rule should live — completion rules included (e.g. whether a tests-must-pass-before-finishing prose rule ought to become a Stop hook), when piled-up prompt prohibitions need relocating to a gate, before trusting a CLAUDE.md prohibition for a must-never-fail constraint, or when adding/reviewing a hook. Skip when judging prose wording/attention/density, or diagnosing why a rule isn't followed with no request to relocate it (use audit-instruction-file), bounding the blast radius of a granted action — sandbox scope, deny floors, reversibility (use audit-harness-safety), the private-data/untrusted-input/egress exfiltration model (use audit-lethal-trifecta), or auditing whether an existing completion gate's anchoring is sound (use audit-verification-gates).
user-invocable: true
version: "0.5.0"
usage: /audit-prompt-hook-placement [path-to-harness-or-instruction-file]
---

# Audit Prompt vs Hook Placement

Every rule in a harness lives in one of two regimes: **prose** (probabilistic — the model *may*
comply) or a **deterministic gate** (hook / CI / tool-restriction — the model *cannot* overrule). A
rule in the wrong regime is the failure this audit detects. Reciprocal of `audit-instruction-file`:
that asks "is this prose good?"; this asks "should it be prose **at all**?"

**Stance — detect and recommend; read-only; apply nothing without confirmation.**
- **Costliest miss: a must-never-fail rule living as prose is not enforcement.** Prompts request,
  hooks require; a hook runs outside the context window — the model can't forget or argue with it
  ([hooks-vs-prompts, Core Distinction](https://agentpatterns.ai/instructions/hooks-vs-prompts/)).
  Relocate the binary subset to a `PreToolUse` hook (exit 2 blocks — but see HP-6 for its
  `Write`/`Edit` coverage gap).
- **Decision rule (per rule).** Hook **only** when all three hold: non-negotiable, binary,
  opposed-by-prior — and expressible at the tool-call boundary. Miss any one → stays prose
  ([hooks-vs-prompts, Decision Rule](https://agentpatterns.ai/instructions/hooks-vs-prompts/)).
- **Inverse error is real.** Forcing a *judgment* rule ("prefer composition", "raise a concern
  before editing auth") into a binary hook regresses it — false positives cost more than rare
  non-compliance ([enforcing-agent-behavior, When This Backfires](https://agentpatterns.ai/instructions/enforcing-agent-behavior-with-hooks/)).
  Keep it in prose; hand wording/density to `audit-instruction-file`.
- **A sound hook with a narrow coverage gap is not a formal finding.** HP-2/4/5/6/7 flag a wrong
  *placement* or *soundness* decision (wrong regime, fails open, unreviewed source). A hook that is
  correctly placed, exit-2, first-party, and blocks the stated case — but whose matcher could in
  principle be tightened further (e.g. an alternate command-chaining phrasing) — is at most a brief
  advisory note, never a tabled Medium/High finding with a required Fix. No regex enumeration is
  ever complete; don't manufacture a finding to demonstrate thoroughness on an already-sound control.

## Input
- `path` (optional): a harness dir or single instruction file. Default: scan `CLAUDE.md` /
  `AGENTS.md` / `.cursor/rules` / sub-dir instruction files / `SKILL.md` bodies **and** the `hooks`
  block of `.claude/settings.json` (+ `settings.local.json`, managed/MDM scope), git hooks, CI gates.

## Scope
Audits the **placement decision** (prose vs deterministic gate) for every rule, and whether placed
hooks are *sound*. **Out of scope:** wording / attention curve / density of rules that legitimately
stay prose → `audit-instruction-file`; loop/cost safety ceilings inside an observability wiring
audit → `audit-observability-setup` (OBS-4; this skill owns the general enforcement-medium decision);
the private-data + untrusted-input + egress trifecta and
egress-removal → `audit-lethal-trifecta` (does **not** enumerate execution paths); application
source-code vulnerabilities → `review-code`.

## Procedure
1. **Inventory every rule** across instruction files and the hook/CI config — what it requires, where
   it lives now.
   Done when every rule has a recorded current location — no blanks.
2. **Score each prose rule on the three-property test** (non-negotiable? binary? opposed-by-prior?).
   All three + tool-call-expressible → *misplaced prose*; recommend a hook. Otherwise stays prose.
   Run the detectors in [`checks.md`](checks.md).
   Done when every prose rule has a recorded three-property verdict.
3. **Scan for pile-up signals** (HP-1): >5 negation rules or `IMPORTANT:` >1× (deterministic,
   countable), or the same error class recurring across prompt edits (needs history evidence) —
   any one is the symptom that a binary subset needs to move structurally.
   Done when both pile-up counts and a history-evidence record (recurrence found / no history available) exist.
4. **Audit each existing hook for soundness** — fail-open exit code (HP-6), substitution gaps (HP-5),
   source trust (HP-7), over-blocking of judgment (HP-4).
   Done when every existing hook has a verdict per soundness check.
5. **Check durability** — must-hold rules placed once in a long-session file with no reminder /
   re-read wiring fade and are paraphrased on compaction (HP-8).
   Done when every must-hold rule has a recorded durability verdict.
6. **Recommend the cheapest correct medium** via the escalation ladder (prompt → skill → hook →
   tool-restriction → verify); stop at the step that eliminates the error. Report with the template.
   Done when every finding names its recommended medium.

## Detectors
Nine detectors (HP-1…HP-9: pile-up, misplaced-absolute, injection-critical-prose, over-blocking,
substitution-gap, fail-open, hook-source-trust, fade/compaction, ceiling-relocation) with Flags / Why
(cited) / Fix live in [`checks.md`](checks.md) — load it when auditing.

## Output template
```
# Prompt-vs-hook placement audit — <target>

## Rule placement
| Rule | Now | Non-neg | Binary | Opp-prior | Correct medium | Check |
|---|---|:--:|:--:|:--:|---|---|
| never `git push --force` to main | CLAUDE.md prose | Y | Y | Y | PreToolUse hook (exit 2) | HP-2 |
| use pnpm, not npm                | CLAUDE.md prose | Y | Y | Y | PreToolUse hook (exit 2) | HP-2 |
| prefer composition in payments   | AGENTS.md prose | N | N | — | stays prose (judgment)   | HP-4 |

## Findings
| Sev | Check | Location | Issue | Fix (deterministic) |
|---|---|---|---|---|
| High | HP-2 | CLAUDE.md | force-push ban is prose for a must-never-fail rule | move to PreToolUse hook, exit 2 |
| High | HP-6 | settings.json | block hook exits 1 → fails open silently | change to exit 2 |
| Med  | HP-1 | CLAUDE.md | 7 negations, IMPORTANT:×3, recurring write-outside error | relocate binary subset to a hook |

**Correctly-placed (no change):** <which prose rules are judgment and stay prose, and why.>
**Smallest high-impact change:** <the one rule to relocate first.>
```
Severity: **High** = must-never-fail rule as prose, a fail-open / source-untrusted hook, **or a
security-critical rule behind a substitution-incomplete hook** (an unhooked `/bin/rm`/MCP/sub-agent
path is materially prose-only exposure); **Medium** = pile-up / fade / over-blocked judgment rule
(HP-4) / non-security substitution gap; **Low** = ceiling-relocation tidy-up.

## Related / pairing
- Reciprocal sibling to **`audit-instruction-file`** — it audits prose *quality*; this audits the
  *placement decision upstream of it*. AIF hands "wrong medium" here; this hands wording/density there.
- Distinct from **`audit-lethal-trifecta`** — that owns the exfiltration legs + egress removal; a
  generic "this npm/secrets/force-push rule is prose that should be a hook" stays here.
- Corpus: [enforcing-agent-behavior-with-hooks](https://agentpatterns.ai/instructions/enforcing-agent-behavior-with-hooks/),
  [prompt-tinkerer](https://agentpatterns.ai/anti-patterns/prompt-tinkerer/),
  [instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- A must-never-fail rule as **prose is not enforcement** — relocate it to a hook the model can't overrule.
- Hook **only** when non-negotiable + binary + opposed-by-prior; forcing a *judgment* rule binary regresses it.
- A block hook must **exit 2** (never 1) or it fails open silently — and where exit-2 coverage is
  gapped (`Write`/`Edit`), prefer JSON stdout with an explicit decision field (HP-6); cover every
  substitution path or pair with CI.
- Read-only — recommend; apply nothing without confirmation.
