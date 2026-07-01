# audit-self-improvement-protocol — canonical section template

The section the fix **installs or repairs** in the always-loaded base instruction file (CLAUDE.md /
AGENTS.md / equivalent). Install it **additively as a delta** (never by rewriting the file). It is a small,
stable protocol — it does **not** grow; learnings route to the backlog. Tailor the bracketed names to the
project (e.g. the protected core files, the backlog location) but keep every rule.

Each line below maps to a detector: presence+always-loaded (SIP-1), delta-only (SIP-2), write-gate
(SIP-3), protected core (SIP-4), no-lossy/audit-trail (SIP-5), autonomous→backlog + no executable self-mod
(SIP-6), content routed out (SIP-7).

---

```markdown
## Self-improvement protocol

Continuously look for ways to improve this harness — instructions, skills, subagents, hooks, commands,
tools, context capture. When you spot one, surface it; don't silently act on it.

- **Spot → surface → track.** Name the opportunity, then OFFER to file it to the backlog
  ([<backlog location>]). Growing learnings live in the backlog / learnings store, **not** in this file —
  this section stays a small, stable protocol.            <!-- SIP-1, SIP-7 -->
- **Deltas, never rewrites.** Any edit to this section or an instruction file is an additive, surgical
  delta — never regenerate or wholesale-rewrite the file.  <!-- SIP-2 -->
- **Gate every persisted change on objective evidence.** Before any learning/skill/instruction persists,
  it must pass a write-gate: **interactive** = explicit human confirmation; the gate is an eval / test /
  objective check, **never my own say-so** (no self-certified "this is better").   <!-- SIP-3 -->
- **Protect the source-of-truth core.** [<CLAUDE.md + CONTEXT.md, etc.>] is the protected core. A proposed
  learning may refine it, but may **never contradict** it.  <!-- SIP-4 -->
- **No lossy compression; keep an audit trail.** Never summarise this section in a way that could drop a
  rare safety-critical line. Persisted entries are dated and reversible.   <!-- SIP-5 -->
- **Autonomous runs FILE, they never APPLY.** In any scheduled / unattended run, file every opportunity to
  the backlog with an audit trail (what was observed · run context · suggested action) and **apply
  nothing**. Never modify code, hooks, or tools unattended. Propose — don't apply.   <!-- SIP-6 -->
```

---

## Notes for the installer

- **Placement.** Put the section in the always-loaded base file; if the file is at its attention/length
  budget, hand the *file-level* restructuring to `audit-instruction-file` first, then install this delta.
- **Backlog wiring.** "OFFER to file" / "file" is the `track-backlog` hand-off — one issue per opportunity,
  deduped (search before create). Interactive OFFERS; autonomous FILES.
- **What this section is NOT.** It is not the sandbox/permission boundary (that is harness config, audited
  by `audit-harness-safety`) and not a deterministic hook (that is `audit-prompt-hook-placement`). This is
  the *written protocol*; the runtime controls enforce it.
