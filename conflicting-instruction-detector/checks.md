# conflicting-instruction-detector — conflict classes & human-review triage

Loaded on demand by `conflicting-instruction-detector`. Each class is self-contained: ID / **Flags**
/ **Why** (cited) / **Question** (the human-review prompt it emits). **Every class is read-only and
emits a question, never a fix.** Two rules only conflict if their *trigger conditions overlap* — they
apply to the same action/input. **The overlap gate (mandatory, before ranking): construct one
concrete input that triggers BOTH rules; if none exists, the pair does not conflict** — surface it in
the "likely-not-a-conflict" list so the false-positive surface is visible, not hidden.

> *Worked example (the canonical false positive):* `never force-push` (doc A) vs `force-push is fine`
> (doc B) reads like a direct CI-1 contradiction — but A scopes to *shared/pulled* history and B to
> *your own un-pulled feature branch*. No single push triggers both rules, so it is **not** a
> conflict; it belongs in likely-not-a-conflict, never at Rank 1. The lexical opposition is a lure;
> the overlap gate is what catches it.

> Posture: detection of cross-document **semantic** conflict is unreliable — Claude-3.5-Sonnet scores
> 92.7% on same-type (intra-constraint) but only 74.6% on cross-type (inter-constraint) conflicts,
> bottoming at 60.3% F1 ([ConInstruct, arXiv:2511.14342](https://arxiv.org/abs/2511.14342)). So
> CI-2/CI-4 (the semantic classes) carry the most false positives — rank them by *cost*, present them
> as candidates, and never let the skill resolve them.

---

## CI-1 — Direct cross-document contradiction (lexically alignable)
- **Flags:** two docs assert opposite values for the same knob — `always X` in one, `never X` /
  `not X` in another (force-push, test-before-commit, indent style, a model id, a branch name).
  Lexically alignable, so this is the *higher-precision* class.
- **Why:** independently-authored docs in one library drift into opposition; nothing reconciles them
  at load time, so behavior becomes position-dependent (the cross-document case of
  `audit-instruction-file` CE-5, lifted out of a single file) ([instruction-file-ecosystem](https://agentpatterns.ai/instructions/instruction-file-ecosystem/)).
- **Question:** "`A` says `<X>`, `B` says `<¬X>`, both apply to `<action>`. Which governs — or is one
  scoped to a subtree/condition the other isn't?"
- **Resolve (human, off-skill):** pick the canonical rule and make the other point to it or scope it.
  Remediation: [learn — the-compliance-stack](https://learn.agentpatterns.ai/prompt-engineering/the-compliance-stack/).

## CI-2 — Mutually-unsatisfiable constraints (semantic — low precision)
- **Flags:** two rules that share *no wording* but cannot both hold for the same input — "respond in
  under 50 words" vs "always include a worked example and a citation"; "never touch prod config" vs
  "keep `deploy.yaml` in sync." The contradiction is in *effect*, inferred, not matched.
- **Why:** this is the cross-type conflict models detect worst (~60% F1, ConInstruct) — the lane that
  *needs* a human and the lane most likely to be a false positive. Expect to be wrong often here
  ([instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/)).
- **Question:** "Can both hold for `<input>`? They look mutually exclusive because `<inferred
  reason>` — is that real, or do they apply under different conditions?"
- **Resolve (human, off-skill):** if genuinely unsatisfiable, one rule has to give — decide which,
  then relax or re-scope it. Remediation: [learn — where-prompting-ends](https://learn.agentpatterns.ai/prompt-engineering/where-prompting-ends/).

## CI-3 — Unresolved cross-document precedence
- **Flags:** two applicable rules in *different* docs/layers prescribe different actions and **no
  precedence is stated** between those docs (e.g. project `CLAUDE.md` vs a `SKILL.md`, two sibling
  rule files). Not necessarily a contradiction in content — an *unresolved ordering*.
- **Why:** precedence follows specificity as a *tendency*, not an enforced rule, so an unstated
  winner makes behavior unpredictable (CE-5 precedence, across documents rather than within a stack)
  ([layered-instruction-scopes](https://agentpatterns.ai/instructions/layered-instruction-scopes/)).
- **Question:** "When `<action>` triggers both `A` and `B`, which wins? Should one declare precedence,
  or is one meant to be scoped narrower?"
- **Resolve (human, off-skill):** state the precedence explicitly, or narrow the broader rule's scope
  so the two stop overlapping. Remediation: [learn — most-specific-wins](https://learn.agentpatterns.ai/prompt-engineering/most-specific-wins/).

## CI-4 — Scope / applicability collision (semantic — low precision)
- **Flags:** two rules whose *trigger conditions overlap* on some inputs and prescribe different
  actions there, while each looks correct in isolation (a role/audience/format rule that contradicts
  a global rule only on the overlap — ConInstruct's constraint-type families: content, format, style,
  length, keyword/phrase).
- **Why:** the conflict exists only on the intersection of two conditions, so it is easy for both
  authors and models to miss — and easy to flag falsely when the conditions don't actually overlap
  ([layered-instruction-scopes](https://agentpatterns.ai/instructions/layered-instruction-scopes/)).
- **Gate (mandatory):** name the *exact input* that lies in both conditions' intersection. If you
  cannot produce one, the conditions are disjoint → this is **not** a conflict; route to
  likely-not-a-conflict. This is the class flagged falsely most often, so the named-input test is
  required before a CI-4 pair may be ranked.
- **Question:** "On inputs matching both `<cond-A>` and `<cond-B>`, `A` wants `<x>` and `B` wants
  `<y>`. Does that intersection occur in practice, and which rule should win there?"
- **Resolve (human, off-skill):** decide the winner on the overlap and encode it as the more specific
  rule. Remediation: [learn — most-specific-wins](https://learn.agentpatterns.ai/prompt-engineering/most-specific-wins/).

## CI-5 — Cross-document drift duplication
- **Flags:** the *same* convention restated in two docs with **divergent wording**, so the two copies
  have begun to mean different things (the cross-document case of CE-6's contradictory restatement;
  distinct from intentional identical edge-repetition, which is **not** a conflict).
- **Why:** duplicated guidance drifts; once two copies disagree, the model reads them as two rules.
  One source of truth per convention; the others should point, not restate
  ([agents-md-as-table-of-contents](https://agentpatterns.ai/instructions/agents-md-as-table-of-contents/)).
- **Question:** "`A` and `B` look like two copies of the same rule but now differ (`<diff>`). Which is
  canonical — collapse to it and have the other point?"
- **Resolve (human, off-skill):** collapse to one canonical statement and replace the copies with a
  pointer to it. Remediation: [learn — point-at-the-spec](https://learn.agentpatterns.ai/prompt-engineering/point-at-the-spec/).

---

## Triage — ranking the review queue
Rank candidates by **review-worthiness = confidence-it's-real × cost-if-followed-wrong**, not by
confidence alone. A middling-confidence conflict over a destructive action (force-push, prod deploy,
secret handling) outranks a high-confidence conflict over formatting. The queue is ordered for a
human's attention, so the costliest ambiguities come first ([WIRE: diagnostics are *conditional, not
deployment-frequency* estimates](https://arxiv.org/abs/2605.27784) — rank by stakes, don't treat a
flag as a confirmed defect).

## What this skill must never do
- **Never resolve** a conflict, pick a precedence winner, or edit a file — it emits a question.
- **Never present a candidate as a finding** — the ~60% F1 floor means false positives are expected;
  always include the likely-not-a-conflict pairs so the reader calibrates trust
  ([ConInstruct](https://arxiv.org/abs/2511.14342)).
- **Never offer a `--fix` flag.** The apply step belongs to the human, or — for a *within-file* fix
  they choose — to `audit-instruction-file`.
