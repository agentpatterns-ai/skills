# audit-prompt-hook-placement — detectors

Loaded on demand. Score each rule on the **three-property test** (non-negotiable + binary +
opposed-by-prior, expressible at the tool-call boundary), then run the hook-soundness and durability
checks. Each item: ID / **Flags** / **Why** (cited) / **Fix** (→ remediation lesson). All read-only.

> Core distinction: prompts *request*, hooks *require*. A hook runs outside the context window — the
> model cannot forget or overrule it ([hooks-vs-prompts, Core Distinction](https://agentpatterns.ai/instructions/hooks-vs-prompts/)).

## Contents
- [Placement detectors (HP-1…HP-4)](#placement-detectors--should-this-rule-be-prose)
- [Hook-soundness detectors (HP-5…HP-7)](#hook-soundness-detectors--is-the-placed-hook-right)
- [Durability detectors (HP-8…HP-9)](#durability-detectors--does-a-correctly-prose-rule-survive-the-session)
- [Escalation ladder](#escalation-ladder-for-recommendations)

---

## Placement detectors — should this rule be prose?

### HP-1 — Prohibition pile-up *(first two triggers deterministic/countable; the third needs history)*
- **Flags:** an instruction file with **>5 negation rules** (`never`/`do not`/`avoid`), or `IMPORTANT:`
  appearing **>1×** — both countable from the file alone; or the **same error class recurring**
  despite prompt edits ("attempt 4") — requires history evidence: run logs, `git log`, **or the
  file's own change-log/notes recording the repeats**; the current rule text alone can't show it.
- **Why:** prompts aren't contracts — IFScale finds compliance drops to 68% at high instruction
  density; ManyIFEval's "curse of instructions" compounds per-rule failure; each new prohibition
  dilutes attention for the rest ([prompt-tinkerer, Symptoms](https://agentpatterns.ai/anti-patterns/prompt-tinkerer/);
  arXiv:[2507.11538](https://arxiv.org/abs/2507.11538), [2509.21051](https://arxiv.org/abs/2509.21051)).
- **Required in finding:** state the argument itself, not just the counts — more prose
  prohibitions do **not** increase compliance; attention is finite and caps around **68%** at high
  instruction density. Cite the IFScale figure and/or `instruction-compliance-ceiling` /
  `prompt-tinkerer` (pair with HP-9). Negation/`IMPORTANT:` counts with no "more prose ≠ more
  compliance" framing is an incomplete finding.
- **Fix:** relocate the **binary subset** to a hook; stop tinkering with prose.
  → [where-prompting-ends](https://learn.agentpatterns.ai/prompt-engineering/where-prompting-ends/).

### HP-2 — Misplaced absolute rule *(core check)*
- **Flags:** a prose rule that is **non-negotiable + binary + opposed by a training prior** and
  expressible at the tool-call boundary — e.g. "never `git push --force` to main", "use pnpm not npm",
  "no writes to secrets files". Miss any of the three → it correctly stays prose.
- **Counter-examples — don't blanket-hook a rule set.** A matcher existing is not sufficient; score
  all three properties per rule. "Avoid absolute paths when creating files" — `avoid`, not `never`,
  and legitimate exceptions exist — fails **non-negotiable** even though a path-pattern check is easy
  to write. "Avoid editing `.github/workflows/` without approval" — the path glob is checkable, but
  the condition it gates ("without approval") is a judgment call no hook can verify — fails
  **binary**. Both stay prose; see HP-4.
- **Why:** these three properties are exactly when a hook beats a prompt; a prose rule the model can
  forget or override mid-task is not enforcement ([hooks-vs-prompts, Decision Rule + What Hooks Can
  Enforce](https://agentpatterns.ai/instructions/hooks-vs-prompts/)).
- **Fix:** move to a `PreToolUse` hook (exit 2 blocks, stderr feeds back to the model; for a
  `Write`/`Edit`-boundary rule use JSON decision output — see HP-6 for the exit-2 coverage gap).
  → [hooks-and-deterministic-enforcement](https://learn.agentpatterns.ai/tool-engineering/hooks-and-deterministic-enforcement/);
  [where-prompting-ends](https://learn.agentpatterns.ai/prompt-engineering/where-prompting-ends/).

### HP-3 — Security/injection-critical rule as prose
- **Flags:** a rule whose failure is a security boundary (block exfiltration / secret write /
  destructive command) stated **only** as prose.
- **Why:** prose is bypassed by prompt injection — an injected instruction and `CLAUDE.md` compete in
  the *same* reasoning loop. A hook fires before execution and is unreachable from that loop; injected
  text can influence what the agent *tries*, not what a hook *allows* ([hooks-vs-prompts, Injection
  Resistance](https://agentpatterns.ai/instructions/hooks-vs-prompts/);
  [prompt-injection-threat-model](https://agentpatterns.ai/security/prompt-injection-threat-model/)).
- **Fix:** a hook (or OS-level control); if it is the *trifecta* leg model, hand to `audit-lethal-trifecta`.
  → [where-prompting-ends](https://learn.agentpatterns.ai/prompt-engineering/where-prompting-ends/).

### HP-4 — Over-blocking / inverse error
- **Flags:** a hook (existing or proposed) enforcing a **judgment / contextual** rule — "prefer
  composition", "raise a concern before editing auth", tone/style — where false positives cost more
  than rare non-compliance.
- **Why:** hooks see parameters, not intent; judgment rules regress when compressed into a regex
  ([hooks-vs-prompts, What Prompts Do](https://agentpatterns.ai/instructions/hooks-vs-prompts/);
  [enforcing-agent-behavior, When This Backfires](https://agentpatterns.ai/instructions/enforcing-agent-behavior-with-hooks/)).
- **Fix:** keep it in prose; hand wording/density to `audit-instruction-file`.
  → [hooks-and-deterministic-enforcement](https://learn.agentpatterns.ai/tool-engineering/hooks-and-deterministic-enforcement/).

---

## Hook-soundness detectors — is the placed hook right?

### HP-5 — Substitution-incomplete hook
- **Flags:** a hook denies one path (e.g. `Bash(rm *)`) but sibling effect-paths are unhooked —
  `/bin/rm`, a truncating `Write`, `perl -e unlink`, sub-agents, MCP calls, pipe/bare mode.
- **Why:** determinism at the tool-call boundary ≠ determinism everywhere; the deny must cover *every*
  tool achieving the same effect ([hooks-vs-prompts, When Hooks Cannot Enforce — substitution +
  execution-path gaps](https://agentpatterns.ai/instructions/hooks-vs-prompts/)).
- **Fix:** cover every effect-path; pair with a CI / git / OS-level gate.
  → [hooks-and-deterministic-enforcement §3](https://learn.agentpatterns.ai/tool-engineering/hooks-and-deterministic-enforcement/).

### HP-6 — Fail-open hook *(deterministic, exit-code inspectable)*
- **Flags:** a block hook exiting with **code 1 (or anything but 2)**, or with no dependency /
  availability guard; **or** a hook guarding a `Write`/`Edit` boundary that relies on **bare exit 2**
  with no JSON decision output — a second, distinct fail-open mode.
- **Why:** any exit code other than 0 or 2 is treated as a hook *error* and does **not** block — it
  fails open silently; a teammate with a missing dependency gets no enforcement and no warning
  ([enforcing-agent-behavior, Block: Exit Code 2 + Hooks fail open silently](https://agentpatterns.ai/instructions/enforcing-agent-behavior-with-hooks/)).
  And exit 2 itself has coverage gaps: `PreToolUse` exit 2 has failed to block `Write` and `Edit`
  while still blocking `Bash` ([anthropics/claude-code #13744](https://github.com/anthropics/claude-code/issues/13744)),
  and has halted the agent idle rather than acting on stderr ([#24327](https://github.com/anthropics/claude-code/issues/24327))
  ([enforcing-agent-behavior, When This Backfires](https://agentpatterns.ai/instructions/enforcing-agent-behavior-with-hooks/)).
- **Fix:** block with **exit 2**, redirect the message to stderr, guard dependencies — and where
  tool-level nuance matters (`Write`/`Edit` boundaries), prefer **JSON stdout with explicit
  `decision`/`permissionDecision` fields** over bare exit codes.
  → [hooks-deterministic-guardrails](https://learn.agentpatterns.ai/harness-engineering/hooks-deterministic-guardrails/).

### HP-7 — Hook-source trust
- **Flags:** project-scope hooks in `.claude/settings.json` carried by an untrusted / unfamiliar repo,
  unreviewed before load.
- **Why:** a malicious project-scope hook is RCE + API-key exfiltration on repo load
  ([CVE-2025-59536](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/));
  the determinism that makes a trusted hook reliable makes a hostile one unconditional
  ([hooks-vs-prompts, When Hooks Cannot Enforce](https://agentpatterns.ai/instructions/hooks-vs-prompts/);
  [enforcing-agent-behavior, Hook Scoping Hierarchy](https://agentpatterns.ai/instructions/enforcing-agent-behavior-with-hooks/)).
- **Fix:** review hook configs before opening unfamiliar repos; put the hard boundary in OS-level /
  managed (MDM) scope, which a project cannot override.
  → [where-prompting-ends §3](https://learn.agentpatterns.ai/prompt-engineering/where-prompting-ends/).

---

## Durability detectors — does a correctly-prose rule survive the session?

### HP-8 — Fade / compaction-fragile critical rule
- **Flags:** a must-hold rule placed *once* at the top of a long-session instruction file, with no
  event-driven reminder and no post-compaction re-read wiring.
- **Why:** static rules lose attention as history grows even while in context, and are *paraphrased*
  (lower fidelity) on compaction with no error signal ([event-driven-system-reminders](https://agentpatterns.ai/instructions/event-driven-system-reminders/);
  [post-compaction-reread-protocol](https://agentpatterns.ai/instructions/post-compaction-reread-protocol/);
  [instruction-compliance-ceiling, Architectural Response](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/)).
- **Fix:** relocate to a hook if binary (preferred); else add an event-driven reminder on the safety
  detector, or a `SessionStart` `matcher:"compact"` re-read-and-confirm hook (not `PostCompact`,
  which can't inject context).
  → [when-the-prompt-fades](https://learn.agentpatterns.ai/prompt-engineering/when-the-prompt-fades/);
  [the-reread-protocol](https://learn.agentpatterns.ai/prompt-engineering/the-reread-protocol/).

### HP-9 — Compliance-ceiling relocation *(supporting)*
- **Flags:** an instruction file whose total rule count buries the must-never-fail subset below the
  reliable-attention window.
- **Why:** even frontier models hold only 68% accuracy at high instruction density; rules that must
  never fail belong in a linter / pre-commit hook / CI gate, not attention-degrading prose
  ([instruction-compliance-ceiling, Architectural Response](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/);
  arXiv:[2507.11538](https://arxiv.org/abs/2507.11538)).
- **Fix:** count and cut; move the absolute subset to hooks to free attention budget for the rest.
  → [hooks-deterministic-guardrails](https://learn.agentpatterns.ai/harness-engineering/hooks-deterministic-guardrails/).

---

## Escalation ladder (for recommendations)
Stop at the step that eliminates the error ([prompt-tinkerer, Escalation Ladder](https://agentpatterns.ai/anti-patterns/prompt-tinkerer/)):

| # | Step | When |
|---|---|---|
| 1 | **Prompt** | guidance, tone, judgment, valid range of interpretation |
| 2 | **Skill** | encode a reusable correct behavior the agent invokes |
| 3 | **Hook** | absolute + binary + opposed-by-prior, at the tool-call boundary (exit 2) |
| 4 | **Tool restriction** | remove the ability entirely when even a hook is leaky |
| 5 | **Accept & verify** | none is cost-effective → human verification gate |
