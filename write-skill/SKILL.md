---
name: write-skill
description: Write or substantially revise one SKILL.md — dispatch description craft (what + when + capabilities, trigger phrases), the knowledge-not-behavior boundary, degrees-of-freedom matching, positive framing, one-level-deep progressive disclosure, a Gotchas section, and eval-able output. Invoke when authoring a new agent skill or reworking an existing SKILL.md's frontmatter or body. Skip when reviewing or grading an existing skill's quality without rewriting it (use audit-skill-quality), auditing a plain instruction file that isn't a SKILL.md (use audit-instruction-file), or shrinking an existing prompt under a token budget (use compress-prompt).
user-invocable: true
version: "0.3.1"
usage: /write-skill [skill idea/purpose + target agent tool + rough scope, or an existing draft SKILL.md to revise]
---

# Write Skill

A skill loads in two stages: the agent scans its `description` before any body content exists in
context, then — only if that description wins — reads the body. Get either stage wrong and the
skill is dead weight: invisible if the description doesn't match how a user asks, misdirecting if
the body doesn't earn the load. **Knowledge, not behavior**, is the load-bearing distinction this
skill teaches throughout: a `SKILL.md` should encode what the agent needs to know (domain rules,
heuristics, a Gotchas list), not a fixed tool-call sequence — knowledge is portable across any agent
that reads markdown; embedded execution steps work in exactly one
([skill-as-knowledge](https://agentpatterns.ai/tool-engineering/skill-as-knowledge/)).

**Stance — generate a draft; the human wires it in; invent no capability.** Output a proposed
`SKILL.md` (+ reference file, if warranted) for review, never a silent in-place overwrite of a
shipped skill. Every claim in the draft traces to what the input establishes about the skill's
purpose and scope — a capability, tool, or behavior the input never mentioned is a bug, not a
helpful addition. A dispatch collision with a sibling skill the author didn't name is **flagged**,
not silently resolved by guessing which one should win.

## Input
- Required: the skill's **purpose** (what task it should make an agent good at) and its **rough
  scope** (one paragraph).
- Optional: target agent tool/format, an existing draft `SKILL.md` to revise, named sibling skills
  it might overlap. Missing siblings → state the assumption; do not invent a skip clause for a
  skill you were not told exists.

## Scope
Authors or substantially revises **one** `SKILL.md` — frontmatter and body together. **Out of
scope:** *grading* an existing skill's quality without rewriting it → `audit-skill-quality` (the
detector pair); auditing a general natural-language instruction file that isn't a `SKILL.md` →
`audit-instruction-file`; shrinking an existing prompt under a token budget → `compress-prompt`;
choosing whether a skill should exist at all versus a smaller artifact (a snippet, a CLAUDE.md line).

## Procedure
1. **Restate purpose + boundary**, then write the `description` as `[what it does] + [when to use
   it] + [key capabilities]`, with concrete trigger phrases a user would actually type and at least
   one negative trigger if the domain overlaps a sibling (WS-1).
   Done when the description names what, when, and capabilities, with no invoke-overlap against a
   named sibling.
2. **Apply the delta principle.** Cut anything the base model already knows (standard syntax,
   generic advice); keep only team conventions, domain rules, and the edge cases the model would
   otherwise get wrong (WS-2).
   Done when every retained line fails the "would the model already do this?" test.
3. **Draft the body as knowledge, not a script.** Domain rules, heuristics, decision tables — never
   embedded tool-call sequences a different agent tool can't run (WS-3). Match each `Procedure`
   step's specificity to its fragility: high freedom (heuristic, judgment) for steps with several
   valid paths, low freedom (an exact sequence) only where one wrong step corrupts state (WS-4).
   Done when no step embeds a tool-specific call and every step's specificity matches its fragility.
4. **Frame rules positively.** State the required behavior; reserve a bare "never/don't" for an
   absolute ban or a space of alternatives too large to enumerate, and put those at the primacy edge
   (WS-5).
   Done when every rule is positive except the ones meeting the negative-phrasing exception.
5. **Structure for progressive disclosure.** Keep the body lean; push catalogs/templates into one
   bundled reference file linked directly from `SKILL.md` — never a second hop — and make the skill
   self-contained unless a pairing is declared (WS-6).
   Done when every reference file is one hop from `SKILL.md` and no undeclared cross-skill dependency exists.
6. **Write a `## Gotchas` section** — the cases where the model would do something plausible but
   wrong. Each entry names the mistake and the correct alternative; only include entries traceable
   to a real failure mode in the input, not invented ones (WS-7).
   Done when every Gotchas entry names a mistake and a fix.
7. **End every step on a checkable done-condition** tied to observable state, not "when it looks
   done" (WS-8). Keep any currency-sensitive fact (a tool name, a version-gated API) undated in the
   live claim, with a prior-approach note if one is needed (WS-9).
   Done when every Procedure step in the draft has a checkable done-condition and no live claim carries an inline date-conditional.
8. **Name how it's testable.** State the with-skill-vs-baseline eval axis and the trigger-precision
   axis this skill's own evals will check (WS-10); emit the draft.
   Done when both eval axes are named and the draft is emitted for review.

## Requirements (WS-1…WS-10, plus tail WS-11/WS-12)
The ten core requirements above, plus two tail/advisory ones — answer the nine contractual
questions (goals/input/permissions/evidence/output/quality/verification/approval/handoff) through
the spine, never as nine headings (WS-11); prefer a deterministic guardrail over a natural-language
"never" for anything that must not fail, since prose alone documents, it doesn't enforce, and 29.9%
of deployed skills silently violate their own declared rules on benign inputs (WS-12) — live in
[`checks.md`](checks.md); load it when drafting.

## Output template
Input: *"a skill for writing Terraform modules"*, drafted with no craft applied:

```yaml
description: Helps with Terraform. Use for infrastructure stuff.
```
```markdown
# Terraform Helper
Don't write bad Terraform. Don't forget to run `terraform plan` before applying. Don't hardcode
values. Always use variables.
```

Corrected (excerpt — frontmatter + the sections the fix touches):

```yaml
---
name: write-terraform-module
description: Write or review a Terraform module against team conventions — variable typing, state
  backend config, provider version pinning. Invoke when authoring a new .tf module or reviewing one
  before a PR. Skip when running terraform commands interactively (use the CLI directly) or auditing
  cloud IAM policy (use a security-review skill).
---
**Stance** — generate a draft module for review; never apply infrastructure changes directly.

## Gotchas
- **Missing a `required_version` pin lets a provider upgrade silently break state** — pin
  `required_providers` version constraints in every module.
- **A default on a `sensitive` variable still shows in `terraform plan` diffs** — mark it
  `sensitive = true` at the output too, not just the variable.

## Critical rules (read last)
- **Pin every provider version.** An unpinned provider is the highest-frequency state-corruption cause.
- **Type every variable explicitly.** No bare `variable "x" {}` blocks.
```
The correction: description now states what/when/capabilities with a real skip clause (WS-1); the
generic "don't write bad Terraform" is gone (WS-2, no-op test); prohibitions became positive
requirements except the still-negative "never apply directly" (an absolute, WS-5); a Gotchas section
replaced vague don'ts with named mistake+fix pairs (WS-7).

## Related / pairing
- **Detector pair — `audit-skill-quality`.** It *flags* quality defects across one or many shipped
  skills, rewriting nothing; this skill *authors* or revises one. Keep the WS-*↔ coverage
  reciprocal as both evolve. Full cited catalog + corpus/lesson links: [`checks.md`](checks.md).

## Critical rules (read last)
- **Generate a draft, never silently overwrite a shipped skill.** The human owns the merge.
- **Invent no capability.** A behavior, tool, or claim the input never established is a bug.
- **Knowledge, not behavior — the anchor.** If a step only works in one agent tool, it's execution
  logic in disguise; put it in the agent or command layer, not the skill.
