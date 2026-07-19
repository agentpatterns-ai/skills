# write-skill — requirement catalog

Loaded on demand by `write-skill`. Apply each requirement while drafting. Each item: ID /
**Flags** (the defect it prevents) / **Why** (cited corpus) / **Fix** (remediation lesson).

## Contents
- [Core (WS-1…WS-10)](#core-ws-1ws-10)
- [Tail — advisory (WS-11…WS-12)](#tail-load-on-demand--advisory)

---

## Core (WS-1…WS-10)

### WS-1 — Description doesn't state what + when + capabilities
- **Flags:** a `description` that names capability but no trigger phrase a user would say, no
  discriminating capability list, or (when a sibling's domain overlaps) no negative trigger; a
  frontmatter missing a required field (`name`, `description`, `user-invocable`, `usage`,
  `version`), or one setting both `disable-model-invocation: true` and `user-invocable: false`.
- **Why:** the description is a learned retrieval key the agent matches against user intent before
  any body content loads — structure it `[what] + [when] + [capabilities]` with concrete trigger
  phrases, and add a negative trigger to stop over-firing on an overlapping sibling
  ([skill-authoring-patterns §Description craft](https://agentpatterns.ai/tool-engineering/skill-authoring-patterns/));
  the field contract — per-field defaults, invocation control, and the caveat that setting both
  `disable-model-invocation: true` and `user-invocable: false` leaves the skill unreachable — is
  specified in [skill-frontmatter-reference](https://agentpatterns.ai/tool-engineering/skill-frontmatter-reference/).
- **Fix:** rewrite to the three-part structure; test with "when would you use this skill?" — see
  `skills-as-a-tool-surface`.
  Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### WS-2 — States what the base model already knows
- **Flags:** instructions restating generic syntax, common-sense advice, or anything the model
  would already do correctly unprompted.
- **Why:** write skill content as a delta from baseline behavior — only team conventions, domain
  rules, and edge cases the model would otherwise get wrong; everything else wastes tokens and
  dilutes the rules that matter ([skill-authoring-patterns §Do not state the obvious](https://agentpatterns.ai/tool-engineering/skill-authoring-patterns/)).
- **Fix:** apply the no-op test line by line; cut what doesn't change behavior versus the default —
  see `skills-as-a-tool-surface`.
  Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### WS-3 — Knowledge collapses into embedded execution ("skill scripts")
- **Flags:** a step that only works via one tool's specific call syntax (`claude_code_tool(...)`,
  a hardcoded shell pipeline) instead of describing the rule/heuristic and letting the agent choose
  how to execute it in its own environment.
- **Why:** skills that embed tool calls, API sequences, or shell commands directly lose the
  portability that knowledge-only skills provide across any markdown-reading agent
  ([skill-as-knowledge §The anti-pattern: skill scripts](https://agentpatterns.ai/tool-engineering/skill-as-knowledge/)).
- **Fix:** rewrite the execution step as the underlying rule/heuristic; if the skill genuinely is a
  fixed procedure, name it a task skill instead of knowledge — see `skills-as-a-tool-surface`.
  Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### WS-4 — Wrong degrees of freedom for the step
- **Flags:** a high-freedom judgment call over-specified into a rigid script (removing valid
  alternative approaches), or a low-freedom/destructive step under-specified into vague heuristic
  language (leaving room for a wrong sequence to corrupt state).
- **Why:** skill content sits on a freedom spectrum from heuristic (multiple valid approaches) to
  deterministic script — match specificity to where the step actually sits
  ([skill-as-knowledge §Why knowledge-only skills are portable](https://agentpatterns.ai/tool-engineering/skill-as-knowledge/)).
- **Fix:** loosen an over-specified judgment step; tighten an under-specified fragile one — see
  `skills-as-a-tool-surface`.
  Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### WS-5 — Negative phrasing where positive would work
- **Flags:** a "never/don't/avoid" rule that isn't an absolute ban and isn't naming an unenumerable
  alternative space — a case where stating the required behavior directly would work just as well.
- **Why:** positive instructions raise the probability of the target behavior; negative instructions
  require suppression and degrade first as instruction count grows
  ([instruction-polarity](https://agentpatterns.ai/instructions/instruction-polarity/)).
- **Fix:** reframe as a positive requirement; keep the negative only for absolute bans or
  unenumerable spaces, moved toward the primacy edge — see `say-what-to-do`.
  Remediation: [learn — say-what-to-do](https://learn.agentpatterns.ai/prompt-engineering/say-what-to-do/).

### WS-6 — Reference file more than one hop from SKILL.md, or no self-containment
- **Flags:** `SKILL.md` → `a.md` → `b.md` chains; a skill that silently depends on another skill
  being loaded first with no declared pairing; a body that has become a reference dump instead of
  splitting into a bundled file.
- **Why:** progressive disclosure only pays off when every reference is directly reachable and each
  skill is self-contained — a dependency the agent may load in a different order (or not at all)
  produces inconsistent output ([progressive-disclosure-agents §Self-contained skills](https://agentpatterns.ai/agent-design/progressive-disclosure-agents/)).
- **Fix:** flatten nested references to one hop; declare the pairing explicitly if one is needed —
  see `skills-and-progressive-disclosure`.
  Remediation: [learn — skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).

### WS-7 — Missing or thin Gotchas section
- **Flags:** no `## Gotchas` section, or one with only vague warnings instead of a named mistake +
  the correct alternative.
- **Why:** Gotchas are the highest-signal content in a skill — they shift the model's prior in the
  narrow cases where it would otherwise guess wrong
  ([skill-authoring-patterns §Gotchas section, §Why it works](https://agentpatterns.ai/tool-engineering/skill-authoring-patterns/)).
- **Fix:** add entries built from real failures, each naming the mistake and the fix — see
  `skills-as-a-tool-surface`.
  Remediation: [learn — skills-as-a-tool-surface](https://learn.agentpatterns.ai/tool-engineering/skills-as-a-tool-surface/).

### WS-8 — No checkable done-condition on a Procedure step
- **Flags:** a step ending on "when it looks done" or with no stated completion signal at all.
- **Why:** missing done-conditions are the measured cause of premature completion — the agent stops
  at the first sign of progress while checks still fail
  ([premature-completion](https://agentpatterns.ai/anti-patterns/premature-completion/),
  [pre-completion-checklists](https://agentpatterns.ai/verification/pre-completion-checklists/)).
- **Fix:** add an observable, exhaustive done-condition to every step — see `the-pre-completion-checklist`.
  Remediation: [learn — the-pre-completion-checklist](https://learn.agentpatterns.ai/verification/the-pre-completion-checklist/).

### WS-9 — Time-anchored claim in the live instruction
- **Flags:** "before/after <date>, do X" written as a live rule instead of a collapsed aside; a
  currency-sensitive tool/API recommendation with no update path.
- **Why:** a dated conditional embedded in the live claim goes silently wrong once the date passes;
  the broader hazard is an instruction file whose claims drift out of sync with reality while the
  agent keeps treating it as authoritative
  ([stale-ai-configuration-artifacts](https://agentpatterns.ai/anti-patterns/stale-ai-configuration-artifacts/)).
  The collapsed "prior approach" aside itself is
  [Anthropic's authoring-guide pattern](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
  — cited transparently as external; the corpus page covers the drift-hazard class it prevents.
- **Fix:** state the current approach undated; move superseded guidance to a collapsed "prior
  approach" note — see `configuration-smells`.
  Remediation: [learn — configuration-smells](https://learn.agentpatterns.ai/anti-patterns/configuration-smells/).

### WS-10 — No stated eval axis
- **Flags:** a shipped skill with no way to tell whether it should be a with-skill-vs-baseline win,
  and no trigger-precision check separate from output-quality.
- **Why:** skills fail on two independent axes — output quality and trigger precision — and each
  needs its own dataset; output-only evals leave trigger failures invisible
  ([skill-evals](https://agentpatterns.ai/verification/skill-evals/)).
- **Fix:** name both axes and start with 2-3 hand-labeled cases per axis — see `evals-at-scale`.
  Remediation: [learn — evals-at-scale](https://learn.agentpatterns.ai/verification/evals-at-scale/).

---

## Tail (load on demand / advisory)

### WS-11 — Nine contractual questions unanswered *(advisory)*
- **Flags:** a skill where a reviewer can't find what outcome counts as success, what it accepts,
  what it may touch, what it must cite, its output shape, what "good" looks like, how output is
  verified, where a human approves, or how control hands off — even though none of these need their
  own heading.
- **Why:** the nine fields raise checkability under audit and multi-author review; they are a
  review lens, not a schema, and a `permissions`-shaped field never substitutes for a runtime
  guardrail ([contractual-skill-files](https://agentpatterns.ai/instructions/contractual-skill-files/)).
- **Fix:** check each of the nine maps to a line somewhere in the spine — see `skills-and-progressive-disclosure`.
  Remediation: [learn — skills-and-progressive-disclosure](https://learn.agentpatterns.ai/harness-engineering/skills-and-progressive-disclosure/).

### WS-12 — Must-never-fail rule left as prose *(advisory — enforcement hand-off)*
- **Flags:** a "never/must not" rule guarding a destructive or irreversible action, stated only as
  a natural-language instruction with no hook/CI backing it.
- **Why:** 29.9% of 402 deployed `SKILL.md` files silently violated their own declared rules on
  benign inputs — natural language documents intent, it does not enforce it
  ([skill-specification-violation-fuzzing](https://agentpatterns.ai/verification/skill-specification-violation-fuzzing/),
  [instruction-polarity §Hooks for true prohibitions](https://agentpatterns.ai/instructions/instruction-polarity/)).
- **Action:** flag the rule and recommend a deterministic gate; this skill does not implement
  hooks — hand off the placement decision to the harness/CI layer.
  Remediation: [learn — hooks-and-deterministic-enforcement](https://learn.agentpatterns.ai/tool-engineering/hooks-and-deterministic-enforcement/).
