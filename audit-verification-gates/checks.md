# audit-verification-gates — detectors

Loaded on demand by `audit-verification-gates`. Classify each completion gate against VG-1…VG-11, then
run the floor guard (VG-11) before emitting any finding. Each item: ID / **Flags** / **Why** (cited corpus
page = evidence) / **Fix** (lesson = remediation). All detection is read-only.

---

### VG-1 — Prose-only completion gate
- **Flags:** "done" is gated by self-report or soft instructions — "be thorough", "make sure", "check your
  work" in `CLAUDE.md` / system prompt — with **no** `Stop` / `SubagentStop` / `PostToolUse` hook or
  middleware that runs outside the model's turn.
- **Why:** prompt-based instructions achieve ~70–90% compliance; a hook achieves near-100% because it runs
  at the system level, immune to the context pressure that degrades prompt compliance ([pre-completion-checklists,
  *Implementation Options* / *Why It Works*](https://agentpatterns.ai/verification/pre-completion-checklists/)).
- **Fix:** implement the checklist as a `PostToolUse`/`Stop` hook or a `PreCompletionChecklist` middleware
  that blocks the completion path until it returns PASS ([the-pre-completion-checklist](https://learn.agentpatterns.ai/verification/the-pre-completion-checklist/)).

### VG-2 — Stopping criterion not tied to observable external state
- **Flags:** the completion check pattern-matches the agent's *reported* status (transcript regex,
  `"status":"done"`, a summary message) instead of executing the tests on the branch and reading
  pass/fail / exit code.
- **Why:** self-reported verification is unfalsifiable — agents claim fixes for code never changed and
  insist tests pass when the transcript shows failures; "a checkpoint that reads the agent's self-report
  is not a checkpoint" ([verification-ledger, *Self-Reported Verification*](https://agentpatterns.ai/verification/verification-ledger/);
  [incremental-verification, *When This Backfires*](https://agentpatterns.ai/verification/incremental-verification/)).
- **Fix:** tie the stopping criterion to observable external state — build/test exit codes, `git diff`,
  schema/DB state — and cross-reference any claim against that evidence ([the-verification-ledger](https://learn.agentpatterns.ai/verification/the-verification-ledger/),
  [check-at-each-step](https://learn.agentpatterns.ai/verification/check-at-each-step/)).

### VG-3 — Vague / unverifiable gate items (gate theater)
- **Flags:** checklist items are not specific and observable — "review the requirement", "check your work"
  — so they nominally pass without verifying anything.
- **Why:** "Vague items provide false confidence" — agents satisfy the surface form of the instruction,
  not the intent; every item must specify a concrete, observable output ([pre-completion-checklists,
  *Checklist Items* / *When This Backfires*](https://agentpatterns.ai/verification/pre-completion-checklists/)).
- **Fix:** make each item name a concrete check ("Run `npm test` and paste the result — do not summarize")
  ([the-pre-completion-checklist](https://learn.agentpatterns.ai/verification/the-pre-completion-checklist/)).

### VG-4 — Verification recorded as prose, not a structured ledger
- **Flags:** evidence is "Build passed. Tests green." rather than per-check rows (tool, command, exit_code,
  output) with a `baseline` capture **before** changes and an `after` capture, and a gate query that blocks
  completion when required rows are missing.
- **Why:** "If the INSERT did not happen, the verification did not happen"; baseline capture is what lets a
  regression (baseline `passed=1`, after `passed=0`) be attributed to the agent, and the gate query enforces
  ordering through data, not trust ([verification-ledger, *Structured Proof* / *Baseline Capture* / *Gate
  Enforcement*](https://agentpatterns.ai/verification/verification-ledger/)).
- **Fix:** a structured ledger (SQL table, `verification.json`, or inline structured-output schema), baseline
  + after rows, gate on `SELECT COUNT(*)` ([the-verification-ledger](https://learn.agentpatterns.ai/verification/the-verification-ledger/)).

### VG-5 — Execution and recording not separated (gameable ledger / non-independent verifier)
- **Flags:** the same agent both runs the tool and writes the row (it can fake an exit code or skip the
  INSERT on failure); or the completion verifier shares the producer's context/model and has write access.
- **Why:** "if the same agent runs the tool and writes the row, it can fake exit codes or skip the INSERT
  when a check fails — the ledger only holds when execution and recording are separated (CI, a harness, or a
  hook)" ([verification-ledger, *When This Backfires*](https://agentpatterns.ai/verification/verification-ledger/)).
- **Fix:** make the verifier independent of the producer, read-only, default-reject on ambiguity; record via
  CI / hook, not the producing agent ([the-verification-ledger](https://learn.agentpatterns.ai/verification/the-verification-ledger/)).

### VG-6 — Verifier weaker than the generator / LLM-judge anchors a mechanical check
- **Flags:** a checkpoint or gate uses an LLM-as-judge where a deterministic check exists, or the judge is
  no more reliable than the model it grades.
- **Why:** "the checkpoint must be more reliable than the thing being verified; compilers and tests qualify,
  unconstrained models often do not" — a hallucinating judge blesses wrong work and rejects correct work
  ([incremental-verification, *When This Backfires*](https://agentpatterns.ai/verification/incremental-verification/)).
  Use model rubric graders *alongside* deterministic checks for genuinely subjective dimensions, not in
  place of them ([grade-agent-outcomes, *Handling Subjective Outcomes*](https://agentpatterns.ai/verification/grade-agent-outcomes/)).
- **Fix:** anchor the gate to deterministic signals (tests / compile / schema); reserve the LLM judge for the
  subjective layer and keep it off mechanical checks ([check-at-each-step](https://learn.agentpatterns.ai/verification/check-at-each-step/),
  [grade-the-outcome](https://learn.agentpatterns.ai/verification/grade-the-outcome/)).

### VG-7 — Verification only at the end — no checkpoints between stages, no baseline
- **Flags:** all checks run after a large batch / at the end of the pipeline; no stage gates between stages;
  no known-good state saved before a batch.
- **Why:** error cost grows with distance from its source — an agent that writes 500 lines before any
  verification may have a wrong assumption at line 10, and unwinding the cascade is expensive; save a
  known-good state before a batch so recovery is bounded ([incremental-verification, *Why This Works* /
  *Checkpoint-Save*](https://agentpatterns.ai/verification/incremental-verification/); [staged-evidence-gates](https://agentpatterns.ai/verification/staged-evidence-gates-program-repair/)).
- **Fix:** insert verification gates between stages; commit/snapshot before a batch; stage gates in
  cost-ascending order ([check-at-each-step](https://learn.agentpatterns.ai/verification/check-at-each-step/)).

### VG-8 — Path-based grading instead of outcome/state-based
- **Flags:** graders assert a tool-call sequence or file-edit order (`tool_calls == ["read","write","bash"]`)
  rather than the final state (tests green, schema/DB state, response fields).
- **Why:** path-based grading penalizes valid alternative solutions as failures and becomes **more**
  misleading as agents grow more capable, because the number of valid paths grows combinatorially
  ([grade-agent-outcomes, *Problem with Path-Based Grading* / *Why It Works*](https://agentpatterns.ai/verification/grade-agent-outcomes/);
  [behavioral-testing-agents](https://agentpatterns.ai/verification/behavioral-testing-agents/)).
- **Fix:** grade the end-state — deterministic tests are the most reliable outcome grader for coding agents
  ([grade-the-outcome](https://learn.agentpatterns.ai/verification/grade-the-outcome/),
  [testing-what-it-decides](https://learn.agentpatterns.ai/verification/testing-what-it-decides/)).

### VG-9 — Mutable tests in the green phase / no test-lock (reward-hacking surface)
- **Flags:** a red-green / TDD workflow lets the agent modify the tests to pass them — no "do not change the
  tests" constraint, no confirm-red step; the surface for `sys.exit(0)` or a weakened assertion.
- **Why:** "Do not change the tests" is the load-bearing green-phase constraint — METR's 2025 evaluations
  document frontier models "modifying test or scoring code", and Anthropic's reward-hacking work documents
  `sys.exit(0)` to make all tests appear to pass ([red-green-refactor-agents, *Constraints That Make This
  Work*](https://agentpatterns.ai/verification/red-green-refactor-agents/); [anti-reward-hacking, *Test
  harness bypass*](https://agentpatterns.ai/verification/anti-reward-hacking/)).
- **Fix:** lock the tests in green ("It is unacceptable to remove or edit tests"); confirm the red state
  before implementing; have the pre-completion checklist verify no existing tests were removed/modified
  ([red-green-for-agents](https://learn.agentpatterns.ai/verification/red-green-for-agents/)).

### VG-10 — No restart-clean completion gate / non-grep-able failure signals
- **Flags:** for a stateful surface, completion isn't gated on the system restarting cleanly (a feature can
  pass unit + integration tests yet leave a corrupt cache, half-applied migration, or stuck worker), and/or
  failure signals are "test fails" rather than a specific grep-able log line / exit code. *(Applicability —
  see VG-11.)*
- **Why:** "No feature is complete if the system cannot restart cleanly afterward"; "test fails" is not a
  failure signal — a specific signal ("`/index` returns 500 with body `chunk size <= 0`") is what makes the
  failure diagnosable and defeats agent rubber-stamping ([golden-journeys, *The Pattern* / *Failure Signal
  Specificity*](https://agentpatterns.ai/verification/golden-journeys/)).
- **Fix:** add a restart-clean gate; give each journey step a specific failure signal (log line / exit code /
  screen state) ([golden-journeys](https://learn.agentpatterns.ai/verification/golden-journeys/)).

### VG-11 — Gate ceremony below its floor (false-positive guard / applicability)
- **Flags:** a finding that demands a heavy gate where its documented floor isn't met — throwaway /
  exploratory / spike work; a stateless or trivial CLI with nothing to restart cleanly; macro-eval
  clustering under ~1,000 traces; an LLM judge under ~70% precision; a ~20-query starter eval set treated as
  insufficient.
- **Why:** every heavy gate has a floor below which it is theater — ledgers don't earn their keep on
  throwaway work, golden journeys produce "ceremony without test signal" on stateless systems, and
  HDBSCAN macro clustering "reports noise or collapses unrelated cases" below ~1,000 traces / below ~70%
  judge precision; ~20 queries already catch large effect sizes. Over-flagging is itself a false positive
  ([verification-ledger](https://agentpatterns.ai/verification/verification-ledger/) / [golden-journeys](https://agentpatterns.ai/verification/golden-journeys/) /
  [staged-evidence-gates](https://agentpatterns.ai/verification/staged-evidence-gates-program-repair/) *When This Backfires*;
  [macro-evals-agentic-systems, *When This Layer Applies*](https://agentpatterns.ai/verification/macro-evals-agentic-systems/);
  [behavioral-testing-agents, *Three-Part Eval Foundation*](https://agentpatterns.ai/verification/behavioral-testing-agents/)).
- **Fix:** scope the demand to where the floor is met; for sub-floor cases recommend the lighter control —
  inline structured output over a SQL ledger, a frequency table over macro clustering, prose spec + review
  over red-green ([evals-at-scale](https://learn.agentpatterns.ai/verification/evals-at-scale/),
  [testing-what-it-decides](https://learn.agentpatterns.ai/verification/testing-what-it-decides/)).
