# audit-skill-quality — detector catalog

Loaded on demand by `audit-skill-quality`. Run each detector against every `SKILL.md` in scope.
Each item: ID / **Flags** / **Why** (cited corpus) / **Fix** (remediation lesson). All detection is
read-only — flag and recommend, never rewrite (that is `write-skill`).

## Contents
- [Dispatch / frontmatter (AQ-1…AQ-4)](#dispatch--frontmatter-first-class)
- [Body / governance (AQ-5…AQ-11)](#body--governance-first-class)
- [Tail — advisory (AQ-12)](#tail--advisory)

---

## Dispatch / frontmatter (first-class)

### AQ-1 — Description doesn't state what + when + capabilities
- **Flags:** a `description` that names the domain but not a trigger condition ("Helps with X"),
  has no negative/skip clause where the domain plausibly overlaps a sibling, or reads in first/second
  person instead of third.
- **Why:** vague, generic descriptions cause arbitrary or missed selection; specific trigger phrases
  and a named skip boundary are what let an agent choose correctly among many skills
  ([skill-authoring-patterns §Description craft](https://agentpatterns.ai/tool-engineering/skill-authoring-patterns/)).
- **Fix:** rewrite as `[what] + [when] + [capabilities]` with a concrete skip clause — hand to
  `write-skill`. Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### AQ-2 — Name doesn't match directory or isn't unambiguous
- **Flags:** a `name` field diverging from its directory, a noun-only/abstract name (`helper`,
  `utils`), or a name colliding in meaning with a sibling skill's name.
- **Why:** verb-noun naming keeps a skill discoverable and its selection intent unambiguous; an
  abstract or mismatched name is a mis-load waiting to happen
  ([skill-library-evolution §Discoverability](https://agentpatterns.ai/tool-engineering/skill-library-evolution/)).
- **Fix:** rename to a verb-noun form matching the directory. Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### AQ-3 — Required frontmatter fields missing or malformed
- **Flags:** a missing `name`/`description`/invocation-mode field, an unset version-equivalent, or a
  `description` that is empty, truncated, or carries markup the frontmatter contract forbids.
- **Why:** the frontmatter is a contract with fixed required fields; a skill missing one is
  malformed at the surface an agent reads first
  ([contractual-skill-files](https://agentpatterns.ai/instructions/contractual-skill-files/),
  [skill-frontmatter-reference](https://agentpatterns.ai/tool-engineering/skill-frontmatter-reference/)).
- **Fix:** add the missing field(s); validate against the target tool's frontmatter schema.
  Remediation: [learn — skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).

### AQ-4 — Description overlaps a sibling / routing degrades at scale
- **Flags:** two skills in the same library whose invoke clauses aren't mutually exclusive (both
  would plausibly fire on the same prompt); auditing a large library with no evidence any
  overlap pass has ever been run.
- **Why:** skill-selection accuracy degrades sharply — a phase transition, not a gradual slope —
  past a critical library size, and the routing problem becomes combinatorially hard as libraries
  grow; overlapping descriptions are the concrete defect that predicts
  ([separation-of-knowledge-and-execution §the Xu & Yan finding](https://agentpatterns.ai/agent-design/separation-of-knowledge-and-execution/)).
- **Fix:** add a skip clause naming the exact colliding sibling; narrow the invoke clause. Remediation:
  [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

---

## Body / governance (first-class)

### AQ-5 — Body too long / attention curve wasted / compliance ceiling
- **Flags:** a body pushing well past ~100 lines (or ~5,000 tokens) with no reference file split; a
  costliest guardrail buried mid-body instead of at the primacy or recency edge; a closing section
  that's a footer, not a restated guardrail.
- **Why:** compliance degrades as aggregate rule count and body length grow; every line pays a
  context load and a human cognitive load, so depth belongs in on-demand reference files, not the
  always-scanned body ([instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/),
  [skill-library-evolution §keep skills under 5,000 tokens](https://agentpatterns.ai/tool-engineering/skill-library-evolution/)).
- **Fix:** split catalogs/templates into a bundled reference file; move the costliest guardrail to
  an attention edge. Remediation: [learn — the-pre-completion-checklist](https://learn.agentpatterns.ai/verification/the-pre-completion-checklist/).

### AQ-6 — Not self-contained
- **Flags:** a `SKILL.md` that silently assumes another skill ran first (shared state, implicit
  ordering) with no declared pairing.
- **Why:** a dependency on another skill creates an ordering requirement agents may not follow;
  self-containment (or an explicitly declared pair) is a hard composability requirement
  ([skill-library-evolution §Composability](https://agentpatterns.ai/tool-engineering/skill-library-evolution/),
  [progressive-disclosure-agents §Self-contained skills](https://agentpatterns.ai/agent-design/progressive-disclosure-agents/)).
- **Fix:** inline the missing prerequisite, or declare the pairing explicitly in `Related / pairing`.
  Remediation: [learn — skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).

### AQ-7 — Progressive disclosure violated (nested references / no TOC)
- **Flags:** a bundled reference file that links to a *second* bundled file (`SKILL.md` → `a.md` →
  `b.md`); any bundled reference file over ~100 lines with no table of contents at the top.
- **Why:** depth belongs one hop from `SKILL.md`, loaded only when needed — a second hop risks a
  partial read (`head -100`) missing content entirely
  ([progressive-disclosure-agents §The pattern](https://agentpatterns.ai/agent-design/progressive-disclosure-agents/));
  the TOC-for-long-files half has no dedicated corpus page — cited directly to
  [Anthropic's skill-authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices),
  disclosed as external rather than invented.
- **Fix:** flatten the second hop into one file or link it directly from `SKILL.md`; add a TOC to any
  reference file over ~100 lines. Remediation: [learn — skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).

### AQ-8 — Guardrail is prose, not enforcement
- **Flags:** a "never/always/must" rule with no backing hook, schema, CI check, or hard hint stated
  or referenced anywhere near it — the rule exists only as natural language.
- **Why:** 29.9% of 402 deployed `SKILL.md` files silently violated their own declared rules on
  benign inputs; a natural-language rule documents intent, it doesn't stop anything
  ([skill-specification-violation-fuzzing](https://agentpatterns.ai/verification/skill-specification-violation-fuzzing/),
  [instruction-polarity §Hooks for true prohibitions](https://agentpatterns.ai/instructions/instruction-polarity/)).
- **Fix:** back a must-never-fail rule with a deterministic mechanism, or note explicitly that none
  exists and the rule is advisory. Remediation: [learn — hooks-and-deterministic-enforcement](https://learn.agentpatterns.ai/tool-engineering/hooks-and-deterministic-enforcement/).

### AQ-9 — Missing step-completion criteria
- **Flags:** a `Procedure` step ending on a vague state ("when it looks done", "once configured")
  instead of a checkable, exhaustive, observable condition.
- **Why:** missing done-conditions are the measured cause of premature completion — the agent stops
  at the first sign of progress while checks still fail
  ([premature-completion](https://agentpatterns.ai/anti-patterns/premature-completion/),
  [pre-completion-checklists](https://agentpatterns.ai/verification/pre-completion-checklists/)).
- **Fix:** rewrite the step's end-state as an observable, checkable condition. Remediation:
  [learn — the-pre-completion-checklist](https://learn.agentpatterns.ai/verification/the-pre-completion-checklist/).

### AQ-10 — Degrees-of-freedom mismatch / negative framing
- **Flags:** a high-freedom judgment call over-specified as a rigid script, or a low-freedom
  destructive step left as loose heuristic guidance; rules leaning on "never/don't" where a positive
  instruction would work just as well and the negative-phrasing exception doesn't apply.
- **Why:** procedure-step specificity should match task fragility, not a fixed default
  ([skill-as-knowledge](https://agentpatterns.ai/tool-engineering/skill-as-knowledge/)); negative
  phrasing measurably degrades compliance as instruction count grows and can raise the rate of the
  prohibited behavior it names ([instruction-polarity](https://agentpatterns.ai/instructions/instruction-polarity/)).
- **Fix:** dial specificity to fragility; reframe non-absolute negatives as positive requirements.
  Remediation: [learn — say-what-to-do](https://learn.agentpatterns.ai/prompt-engineering/say-what-to-do/).

### AQ-11 — Untestable / no eval axis named
- **Flags:** a skill with no stated way to check it changed behavior — no with-skill-vs-baseline
  comparison, no trigger-precision cases (does it fire/not-fire correctly on positive and negative
  prompts).
- **Why:** a skill's job is to change agent behavior on two independent axes — output quality and
  trigger precision — and both need a paired with-skill/baseline run to verify
  ([skill-evals](https://agentpatterns.ai/verification/skill-evals/)).
- **Fix:** add at least one with-skill-vs-baseline case and one trigger-precision (positive +
  negative) case. Remediation: [learn — the-skill-eval-loop](https://learn.agentpatterns.ai/claude-code/the-skill-eval-loop/).

---

## Tail — advisory

### AQ-12 — Nine contractual fields incomplete
- **Flags:** the skill's goals, input boundaries, permissions/stance, evidence requirements, output
  contract, quality criteria, verification steps, approval points, or handoff rules are answerable
  nowhere in the file — not missing as headings, missing as *content*.
- **Why:** contractual skill files pay off precisely because they answer these nine questions
  somewhere; unanswered ones are gaps a reader (or a downstream skill) has to guess at
  ([contractual-skill-files](https://agentpatterns.ai/instructions/contractual-skill-files/)). A
  Permissions/Stance field is documentation, never enforcement — don't read it as a guardrail.
- **Fix:** answer the missing field through the existing spine — never add it as a tenth heading.
  Remediation: [learn — skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).
