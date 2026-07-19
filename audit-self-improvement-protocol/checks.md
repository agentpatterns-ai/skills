# audit-self-improvement-protocol — detectors

Loaded on demand by `audit-self-improvement-protocol`. For the always-loaded base instruction file
(CLAUDE.md / AGENTS.md / equivalent), run SIP-1…SIP-7 plus the SIP-0 precision guard. Each item: ID /
**Flags** / **Why** (corpus; an external arXiv anchor appears only where a cited corpus page itself
carries it — the one corpus-external anchor is explicitly marked as such) / **Fix** (additive delta) +
remediation lesson. All detection is read-only; the fix applies only on confirmation (interactive), and is
filed to the backlog (never applied) in autonomous mode.

The unifying rule: a self-improvement loop compounds across sessions, so a single bad persisted change
ratchets. Every write must be gated on **objective evidence**, additive, bounded to a small always-loaded
section, and — when unattended — routed to the backlog rather than applied.

## Contents
- [SIP-1 — Protocol present and in always-loaded context](#sip-1--self-improvement-protocol-present-and-in-always-loaded-context-single-finding-when-missing-high-if-a-loop-already-runs-else-medium--no-sip-27-fan-out)
- [SIP-2 — Delta-update rule present](#sip-2--delta-update-rule-present-no-wholesale-rewrite-medium)
- [SIP-3 — Write-gate / approval boundary defined](#sip-3--write-gate--approval-boundary-defined-two-part-high-medium-sub-case)
- [SIP-4 — Protected source-of-truth core declared](#sip-4--protected-source-of-truth-core-declared-medium)
- [SIP-5 — No lossy compression](#sip-5--no-lossy-compression-rare-safety-critical-lines-protected-medium)
- [SIP-6 — Autonomous → backlog + audit trail](#sip-6--autonomous--backlog--audit-trail-no-autonomous-executable-scaffolding-self-modification-high)
- [SIP-7 — Growing content routed out of the always-loaded file](#sip-7--growing-content-routed-out-of-the-always-loaded-file-low)
- [SIP-0 — Precision / anti-theatre guard](#sip-0--precision--anti-theatre-guard-do-not-flag-a-well-formed-protocol)

---

## SIP-1 — Self-improvement protocol present and in always-loaded context *(single finding when missing: High if a loop already runs, else Medium — no SIP-2…7 fan-out)*
- **Flags:** no self-improvement / learning-loop section in the always-loaded base file; the loop exists only
  ad-hoc in chat, in a non-loaded doc, or not at all; or the section is present but in a file that is not
  actually loaded every session. When the load path cannot be confirmed from file inspection (a
  non-standard harness), report it as "unverified — confirm load path", not pass/fail.
- **Why:** improvement compounds only when the loop runs every session as a standing protocol on the
  system's own infrastructure — a closed loop "improving the system's own infrastructure, not individual
  task outputs" ([agentic-flywheel](https://agentpatterns.ai/agent-design/agentic-flywheel/)). A loop
  with no governing section in context is unguided.
- **Fix:** install the canonical protocol section (see `template.md`) into the always-loaded base file as an
  additive delta. Remediation: [eval-driven-harness-improvement](https://learn.agentpatterns.ai/harness-engineering/eval-driven-harness-improvement/).

## SIP-2 — Delta-update rule present (no wholesale rewrite) *(Medium)*
- **Flags:** the section instructs the agent to "rewrite / regenerate / rewrite the whole file/section" when
  it learns; no rule constraining edits to additive, surgical, individually-addressable deltas.
- **Why:** when an LLM rewrites a context it drops hard-won domain knowledge (brevity bias) and repeated full
  rewrites compound into context collapse — a measured run fell 18,282→122 tokens with a 9.6-point accuracy
  drop; structured delta entries that merge deterministically preserve knowledge
  ([evolving-playbooks](https://agentpatterns.ai/context-engineering/evolving-playbooks/); ACE / Zhang et al.
  arXiv:2510.04618).
- **Fix:** state "edits are additive delta entries — never a wholesale rewrite of the
  section or file." Remediation: [context-compression](https://learn.agentpatterns.ai/context-engineering/context-compression/).

## SIP-3 — Write-gate / approval boundary defined (two-part) *(High; Medium sub-case)*
- **The gate is two-part** everywhere this skill states it: **(a) approval boundary** — who may let a
  change land (interactive = explicit human confirmation; autonomous = file to backlog, never apply);
  **(b) evidence standard** — what justifies the change (an eval / test / objective observation, never the
  agent's self-assessment).
- **Flags (High):** a learning/skill/instruction persists with no gate at all, or the gate is the agent's
  own judgement ("if you think it's an improvement, save it") — self-assessment as the write authority.
- **Flags (Medium sub-case):** an approval boundary exists (human confirm interactive / backlog-only
  autonomous) but no evidence standard — nothing requires the proposed change to be justified by an eval /
  test / objective observation before it is put to the approver.
- **Why:** the safe loop is reflect→draft→**validate**→persist — a quality gate is a non-negotiable
  prerequisite, and a single agent reflecting on its own output rationalises rather than critiques, so the
  evidence standard must be objective, not self-assessment (reward hacking / sycophancy)
  ([self-rewriting-meta-prompt-loop](https://agentpatterns.ai/agent-design/self-rewriting-meta-prompt-loop/);
  [memory-synthesis-execution-logs](https://agentpatterns.ai/agent-design/memory-synthesis-execution-logs/),
  the corpus page carrying the Voyager self-verification arXiv:2305.16291 and SSGM Write-Validation-Gate
  arXiv:2603.11768 anchors).
- **Fix:** state both parts of the gate — (a) interactive = human confirm, autonomous = file to backlog
  (never apply); (b) every persisted change justified by evals / objective evidence, never the agent's
  say-so. Remediation:
  [verification-gates](https://learn.agentpatterns.ai/harness-engineering/verification-gates/).

## SIP-4 — Protected source-of-truth core declared *(Medium)*
- **Flags:** no declaration of a protected, human-authored core (e.g. CLAUDE.md + CONTEXT.md) that proposed
  learnings may never contradict; the loop is free to overwrite or supersede foundational rules.
- **Why:** intent must be preserved and unmodifiable through execution, and a consolidation gate must stop a
  memory write that contradicts the protected core — otherwise locally-reasonable writes accumulate into
  behaviour no one authorised (compositional drift)
  ([frozen-spec-file](https://agentpatterns.ai/instructions/frozen-spec-file/);
  [layered-mutability](https://agentpatterns.ai/agent-design/layered-mutability/); SSGM arXiv:2603.11768).
- **Fix:** name the protected core and state "proposed learnings may refine, never contradict, the core."
  Remediation: [instruction-files-and-altitude](https://learn.agentpatterns.ai/harness-engineering/instruction-files-and-altitude/).

## SIP-5 — No lossy compression; rare safety-critical lines protected *(Medium)*
- **Flags:** the protocol allows summarising/compacting the section or memory in a way that can silently drop
  low-frequency lines; no audit trail; learnings overwrite in place with no dated/reversible record.
- **Why:** premature summarisation strips the diagnostic signal reflection depends on, and knowledge degrades
  through iterative summarisation — low-frequency safety-critical instructions are exactly what a lossy pass
  drops, and a single bad persisted entry compounds over the agent's lifetime
  ([memory-synthesis-execution-logs](https://agentpatterns.ai/agent-design/memory-synthesis-execution-logs/);
  [layered-mutability](https://agentpatterns.ai/agent-design/layered-mutability/)).
- **Fix:** forbid lossy compression of the section; require dated, reversible, audit-logged
  entries and explicit protection of safety-critical lines. Remediation:
  [remember-dont-re-read](https://learn.agentpatterns.ai/context-engineering/remember-dont-re-read/).

## SIP-6 — Autonomous → backlog + audit trail; no autonomous executable-scaffolding self-modification *(High)*
- **Flags:** the protocol lets an autonomous/scheduled run apply changes unattended, or rewrite executable
  scaffolding (code, hooks, tools, commands); no rule that unattended runs file to the backlog and apply
  nothing; no structured audit trail (observation + run context + suggested action).
- **Why:** autonomous routines must externalise observations to a durable, deduped backlog — the signal dies
  in the transcript otherwise — and removing the human approval step from self-modification trades speed for
  unreviewed drift and adversarial exposure; safety-critical self-modification without human sign-off is a
  hard no, so propose-don't-apply is the boundary
  ([self-reporting-loops](https://agentpatterns.ai/agent-design/self-reporting-loops/);
  [self-rewriting-meta-prompt-loop](https://agentpatterns.ai/agent-design/self-rewriting-meta-prompt-loop/);
  corpus-external anchor, marked as such — STOP arXiv:2310.02304 reports proposed self-improvements
  attempting to disable their own sandbox flag).
- **Fix:** state "autonomous runs file to the backlog with an audit trail and apply nothing; never modify
  code/hooks/tools unattended." Remediation:
  [plan-mode-and-plan-first](https://learn.agentpatterns.ai/harness-engineering/plan-mode-and-plan-first/).

## SIP-7 — Growing content routed OUT of the always-loaded file *(Low)*
- **Flags:** the protocol accumulates learnings *in* the always-loaded section (a growing list that inflates
  the base file); no two-store split between the small stable protocol and a separate backlog / learnings store.
- **Why:** models reliably follow only ~150–200 instructions and an ever-growing base file collapses
  compliance — the always-loaded file is a table of contents, not an encyclopedia; durable content belongs in
  a second, on-demand store ([instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/);
  [agents-md-as-table-of-contents](https://agentpatterns.ai/instructions/agents-md-as-table-of-contents/);
  IFScale arXiv:2507.11538).
- **Fix:** keep the section a small stable protocol; route growing learnings to the
  backlog / learnings store (GitHub issues here). Remediation:
  [skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).

---

## SIP-0 — Precision / anti-theatre guard (do NOT flag a well-formed protocol)
- **Flags (suppress findings when present):** the section is present and always-loaded (SIP-1), states a
  delta-only rule (SIP-2), defines both parts of the write-gate — approval boundary AND objective-evidence
  standard (SIP-3), declares a protected core
  (SIP-4), forbids lossy compression and keeps an audit trail (SIP-5), routes autonomous findings to the
  backlog and bars executable self-mod (SIP-6), and keeps growing content out of the base file (SIP-7).
- **Why:** flagging an already-correct protocol is noise that trains reviewers to ignore the audit — the
  tiered-approval design is the *correct* shape, confirm it rather than duplicate it
  ([agentic-flywheel](https://agentpatterns.ai/agent-design/agentic-flywheel/)).
- **Fix:** report these in the **Clean** section with the element that holds them; raise a finding only on a
  real gap. Remediation: [eval-driven-harness-improvement](https://learn.agentpatterns.ai/harness-engineering/eval-driven-harness-improvement/).
