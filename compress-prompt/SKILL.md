---
name: compress-prompt
description: Compress any prompt or instruction file to maximize signal per token — convert prose to tables/bullets/rules, front-load critical constraints, and apply semantics-preserving transforms. Invoke when asked to shorten, tighten, densify, or "make this prompt/instruction file more concise". Skip when editing ordinary docs or source code, or when the goal is to detect (not fix) density problems — use audit-instruction-file for that.
user-invocable: true
version: "0.1.0"
usage: /compress-prompt [path-or-text]
---

# Compress Prompt

Maximize signal per token in prompts and instruction files. Compression removes
**words, not meaning** — shorter, denser instructions are followed more reliably
(Anthropic: "Bloated CLAUDE.md files cause Claude to ignore your actual
instructions"; [Claude Code best practices](https://code.claude.com/docs/en/best-practices))
and cost fewer tokens on every turn an always-loaded file is in context.

This skill **transforms** prompt/instruction text — the fix-side complement to
`audit-instruction-file`'s **CE-10** detector (detector → transform; run as a pair).

## Scope

Transforms prompt and instruction text (system prompts, `AGENTS.md`, `CLAUDE.md`,
skill bodies, agent instructions, one-off task prompts). It does **not** run, eval,
or benchmark prompts, and it does not measure live compliance — it edits text and
reports a token delta.

## Stance

- **Recommend the compressed version by default.** Show before/after; apply in place
  only on explicit confirmation.
- **Semantics-preserving only.** Strip ceremony, never meaning. A transform that
  drops a constraint, a conditional, or compliance-critical rationale is a bug, not
  a compression.
- **Do not invent precedence or restructure governance.** Criticality and ordering
  are human judgments — surface them, don't impose them.

## Input

- `path-or-text` (optional): a file to compress in place, or pasted prompt text.
  Default: the prompt/instruction text in the current request.

## Procedure

1. **Measure baseline.** Count tokens (~chars/4) and normative statements
   (`never|always|must|do not|required`). This is the before-figure.
2. **Inventory constraints.** List every rule, conditional, and piece of
   compliance-critical rationale *before* editing. This list is the invariant the
   transform must preserve.
3. **Apply techniques** (catalog in [`techniques.md`](techniques.md) — load it now), in order:
   structural first (move task-specific content out), then lexical (prose → table/bullet/rule).
4. **Run the compression test** on every surviving line: *"Can I remove this word —
   or this whole sentence — without losing a constraint?"* If yes, cut it.
5. **Verify no constraint was lost.** Diff the compressed text against the step-2
   inventory. Every constraint and every load-bearing conditional must still be
   present. Confirm you compressed **decisively** — no rule left half-trimmed.
6. **Report** the before/after token delta and show both versions. Apply in place
   only if confirmed.

## Techniques

Apply **structural first** (move task-specific content to on-demand files — the highest-leverage
move, since aggregate rule load, not file count, drives compliance degradation: past the ceiling even
frontier models hold ~68% accuracy, [IFScale, arXiv:2507.11538](https://arxiv.org/abs/2507.11538)),
then **lexical**. Full catalog (with when-to-use) + a worked before/after in
[`techniques.md`](techniques.md) — load it before compressing.

- **Tables over prose** · **Bullets over sentences** · **Rules over explanations** (cut the *why* unless compliance needs it)
- **Negative constraints** ("Never X" beats enumerating valid options) · **Examples over descriptions**
- **Front-load critical rules** — attention is U-shaped ([Liu et al. 2023, arXiv:2307.03172](https://arxiv.org/abs/2307.03172))
- **Structural compression** — move task-specific content out; don't lexically squeeze it

## Guardrails

A naive "make it shorter" breaks both — these are the non-obvious edges.

- **Don't strip meaning.** Cut ceremony freely; never cut high-density tokens (rationale, conditionals,
  disambiguating names). Removing them shifts the interpretive burden to reasoning — a 17% input-token
  cut raised total session cost **67%** ([Ustynov 2026, arXiv:2604.07502](https://arxiv.org/abs/2604.07502)).
  Step 5's inventory diff is what enforces this.
- **Compress decisively, not halfway.** Constraint violations peak at *medium* compression (a U-curve,
  97.2% prevalence; [arXiv:2512.17920](https://arxiv.org/abs/2512.17920)) — go all the way to a crisp,
  unambiguous rule, or leave it verbose. A half-trimmed rule hurts compliance *more* than the original.

## Report template

```
# Prompt compression

Baseline: <N> tokens, <R> normative statements
Compressed: <M> tokens (−<X>%), <R> normative statements   ← rule count unchanged

Constraints preserved: <list — must match the step-2 inventory>
Techniques applied: <e.g. prose→table, structural move of <topic> to a skill>

[before block]
[after block]
```

Apply in place only on explicit confirmation.

## Related

- Corpus: [Prompt Compression](https://agentpatterns.ai/context-engineering/prompt-compression/) ·
  [Semantic Density Optimization](https://agentpatterns.ai/context-engineering/semantic-density-optimization/) ·
  [Negative Space Instructions](https://agentpatterns.ai/instructions/negative-space-instructions/) ·
  [Instruction Compliance Ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/).
- Out of scope: *automated* token-level compressors — LLMLingua / LongLLMLingua
  ([2310.05736](https://arxiv.org/abs/2310.05736), [2310.06839](https://arxiv.org/abs/2310.06839)).
  This skill is authoring-craft (lexical/structural rewrites), not a soft-compression pipeline.

## Critical rules (read last)

- **Semantics-preserving only.** A transform that drops a constraint, a conditional, or
  compliance-critical rationale is a bug, not a compression.
- **Recommend; apply only on explicit confirmation.** Show before/after; never invent precedence or
  restructure governance — criticality is a human judgment to surface, not impose.
- **Load [`techniques.md`](techniques.md)** for the technique catalog + worked example before compressing.
