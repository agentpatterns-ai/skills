---
name: audit-self-improvement-protocol
description: Audit whether an agent's always-loaded base instruction file (CLAUDE.md / AGENTS.md / equivalent) carries a well-formed self-improvement protocol — the standing section telling the agent to spot harness-improvement opportunities (instructions, skills, subagents, hooks, commands, tools), gate every persisted change behind approval, route learnings to a backlog, protect the source-of-truth core, and never autonomously rewrite its own executable scaffolding — then install or repair the canonical protocol on confirmation. Invoke when adding, reviewing, or repairing a continuous self-improvement / learning-loop section in an agent's base instruction file, or when wiring an agent to capture and track its own improvement ideas. Skip when auditing the whole instruction file's attention / length / density structure (use audit-instruction-file), auditing harness permissions / sandbox / reversibility / cost boundaries (use audit-harness-safety), placing a must-never-fail rule into a deterministic hook (use audit-prompt-hook-placement), or simply filing one backlog item (use track-backlog).
user-invocable: true
version: "0.1.0"
usage: /audit-self-improvement-protocol [path-to-base-instruction-file-or-repo]
---

# Audit Self-Improvement Protocol

A self-improving agent runs a learning loop every session — spotting harness-improvement opportunities,
reflecting, and persisting what it learns. That loop is only as safe as the **protocol** governing it,
and that protocol must live as a small standing section in the **always-loaded** base instruction file
(CLAUDE.md / AGENTS.md / equivalent). This skill audits whether that section exists and is well-formed,
and on confirmation **installs or repairs** the canonical protocol — mirroring `audit-instruction-file`
(detect → recommend → apply on confirmation), but for one specific section and its running rules.

**Stance — detect, recommend, and (only on explicit confirmation) apply additive fixes; never auto-apply.**
- **The loop must improve through evidence, not self-assertion.** Every persisted change passes a write-gate
  on **objective evidence / evals — never the agent's own say-so**; sycophantic self-assessment and reward
  hacking are the named failure modes (SIP-3). A natural-language protocol rule *documents* the boundary;
  the runtime gate / human confirm is what *stops* the write.
- **Autonomous mode files, it never applies.** In autonomous/scheduled runs the agent routes every
  opportunity to the **backlog** with an audit trail and modifies nothing; only interactive mode applies,
  on confirmation (SIP-6). Autonomous rewrite of executable scaffolding (code, hooks, tools) is out of bounds.
- **Keep the always-loaded section small.** Edits are surgical **deltas, never wholesale rewrites** (SIP-2);
  growing learnings route OUT to the backlog / learnings store (SIP-7); the protected human-authored core is
  never contradicted (SIP-4) and never lossily compressed (SIP-5).

## Input
- `path` (optional): the base instruction file (CLAUDE.md / AGENTS.md / GEMINI.md / `.cursorrules`), a dir,
  or a repo. Default — locate the always-loaded base file(s) at the repo root and read the
  self-improvement / learning-loop section (or detect its absence).

## Scope
Audits **one section + its running protocol** inside the always-loaded base file, and the wiring of the
loop's autonomous/interactive paths. **Out of scope:** the whole file's attention/length/layering/density
(→ `audit-instruction-file`); harness permissions / sandbox / reversibility / cost — where the runtime
actually stops an action (→ `audit-harness-safety`); converting a must-never-fail rule into a deterministic
hook (→ `audit-prompt-hook-placement`); the act of filing one issue (→ `track-backlog`, the mechanism this
skill hands off to). It flags the *protocol text* for propose-don't-apply; it does not audit the runtime
sandbox that backs it.

## Procedure
1. **Locate** the always-loaded base file(s) and extract the self-improvement / learning-loop section (or
   record its absence — SIP-1).
2. **Run the detectors** in [`checks.md`](checks.md) (SIP-1…SIP-7) plus the SIP-0 precision guard, one pass.
3. **Classify severity.** Critical = a missing/ungated write boundary that lets the loop self-modify
   unilaterally (SIP-3, SIP-6). High = no delta-rule / no protected core / lossy compression (SIP-2, SIP-4,
   SIP-5). Medium = section absent-but-harmless or growing content not yet routed out (SIP-1, SIP-7).
4. **Recommend the fix per finding** (the `checks.md` Fix column). When the section is missing or malformed,
   offer to install the canonical protocol from [`template.md`](template.md) — additively, as a delta.
5. **Apply only on explicit confirmation (interactive).** In autonomous mode, do **not** apply — **file the
   findings to the backlog** via `track-backlog` with the audit trail and stop (SIP-6).
6. **Apply the SIP-0 precision guard.** Do not flag a section already well-formed; confirm it in **Clean**.

## Checks
Seven detectors + the SIP-0 precision guard, each with Flags / Why→corpus / Fix→lesson (two-link), in
[`checks.md`](checks.md), loaded when auditing: SIP-1 protocol present & always-loaded · SIP-2 delta-update
(no wholesale rewrite) · SIP-3 write-gate on objective evidence · SIP-4 protected source-of-truth core ·
SIP-5 no lossy compression / safety-line protection · SIP-6 autonomous→backlog + no executable self-mod ·
SIP-7 growing content routed out. The canonical section the fix installs is in [`template.md`](template.md).

## Worked example (SIP-6, the autonomous boundary)
**Before:** a CLAUDE.md self-improvement section reads "When you discover an improvement, update the
relevant instruction file or skill so it persists." Run in a nightly autonomous routine, the agent rewrites
its own hooks. **Finding:** Critical, SIP-6 — no autonomous boundary; permits unilateral executable
self-modification. **After (the fix, a delta):** add "**Autonomous runs file to the backlog and apply
nothing**; never modify code/hooks/tools unattended. Interactive runs may apply a learning only after a
write-gate passes on objective evidence." The boundary is now stated; the runtime sandbox (→
`audit-harness-safety`) enforces it.

## Output template
```
# Self-improvement protocol audit — <repo / base file>
Base file: <path> (always-loaded) · Section: <present | MISSING>

## Findings
| Sev | Check | File:line | Flag | Fix (additive delta) |
|---|---|---|---|---|
| Critical | SIP-6 | CLAUDE.md:88 | autonomous run may self-edit hooks | "autonomous → file to backlog; apply nothing" |
| Critical | SIP-3 | CLAUDE.md:84 | learning persists on the agent's say-so | gate on evals/objective evidence before persist |
| High | SIP-2 | CLAUDE.md:90 | "rewrite the section" instruction | additive delta entries only; no full rewrite |
| Medium | SIP-7 | CLAUDE.md:92 | learnings accumulate in the file | route growing content to backlog/learnings store |

**Clean (SIP-0):** <well-formed elements that conform, and why.>
**Highest-impact fix:** <the one change to make first.>
**Mode:** interactive → offered to apply on confirmation | autonomous → filed to backlog (nothing applied).
```

## Related
- **`audit-instruction-file`** — audits the whole base file's structure (CE-1…CE-10); this skill owns the
  one self-improvement section + its protocol.
- **`audit-harness-safety`** — permissions/sandbox/reversibility/cost; route sandbox/blast-radius there.
- **`track-backlog`** — the filing mechanism; interactive OFFERS to file, autonomous FILES (never applies).

## Critical rules (read last)
- **Never auto-apply.** Autonomous mode files to the backlog with an audit trail and modifies nothing;
  interactive applies only on explicit confirmation, and only after a write-gate on objective evidence.
- **Deltas, not rewrites; keep the section small.** Additive/surgical edits; route growing content out;
  never contradict or lossily compress the protected source-of-truth core.
- **Load [`checks.md`](checks.md) before auditing** and install the section from [`template.md`](template.md).
