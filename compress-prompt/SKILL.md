---
name: compress-prompt
description: Compress any prompt or instruction file to maximize signal per token — convert prose to tables/bullets/rules, front-load critical constraints, and apply semantics-preserving transforms. Invoke when asked to shorten, tighten, densify, or "make this prompt/instruction file more concise". Skip when editing ordinary docs or source code, or when the goal is to detect (not fix) density problems — use audit-instruction-file for that.
user-invocable: true
version: "0.1.0"
usage: /compress-prompt [path-or-text]
---

# Compress Prompt

Maximize signal per token. Compression removes **words, not meaning** — shorter, denser
instructions are followed more reliably (Anthropic: "Bloated CLAUDE.md files cause Claude to
ignore your actual instructions"; [Claude Code best practices](https://code.claude.com/docs/en/best-practices))
and cost fewer tokens every turn an always-loaded file is in context.

This skill **transforms** text — the fix-side complement to `audit-instruction-file`'s **CE-10**
detector (detector → transform; run as a pair).

## Scope

Transforms prompt/instruction text (system prompts, `AGENTS.md`, `CLAUDE.md`, skill bodies,
agent instructions, one-off task prompts). For one-off text it edits and reports a token delta —
no benchmark, no live-compliance measurement. **When the target is a skill or instruction artifact
that has (or should have) a behavioral eval suite, the compression is eval-gated**: run the suite
before and after; any regression holds the compression rather than shipping it ([`eval-gate.md`](eval-gate.md)).

## Stance

- **Recommend by default.** Show before/after; apply in place only on explicit confirmation.
- **Semantics-preserving only.** Strip ceremony, never meaning. Dropping a constraint, a
  conditional, or compliance-critical rationale is a bug, not a compression. When the target carries
  an eval suite this is enforced **behaviorally** — green before and after, or the compression is
  held (go/no-go is the eval result, never the token delta).
- **Do not invent precedence or restructure governance.** Criticality and ordering are human
  judgments — surface them, don't impose them.

## Input

- `path-or-text` (optional): a file to compress in place, or pasted prompt text.
  Default: the prompt/instruction text in the current request.

## Procedure

1. **Establish the eval baseline (behavioral targets).** If the target is a skill/instruction artifact
   that has — or, by the factory harness convention, *should* have — an eval suite, run it **before
   touching anything** and confirm a green baseline. No suite? **Recommend one and offer to scaffold
   it** before compressing — don't silently compress un-evalled behavior. One-off text → skip to step 2.
   Detail in [`eval-gate.md`](eval-gate.md) — load it now if this applies.
2. **Measure baseline.** Count tokens (~chars/4) and normative statements
   (`never|always|must|do not|required`). The before-figure.
3. **Inventory constraints.** List every rule, conditional, and compliance-critical rationale
   *before* editing — the invariant the transform must preserve.
4. **Apply techniques** (catalog in [`techniques.md`](techniques.md) — load it now), in order:
   structural first (move task-specific content out), then lexical (prose → table/bullet/rule).
5. **Compression test** on every surviving line: *"Can I remove this word — or this whole
   sentence — without losing a constraint?"* If yes, cut it.
6. **Verify no constraint was lost.** Diff against the step-3 inventory; every constraint and
   load-bearing conditional must still be present. Compress **decisively** — no rule left half-trimmed.
7. **Re-run the eval suite (behavioral targets).** Re-run the **same** suite from step 1
   post-compression — **every eval must still pass.** On any regression, **hold the compression and
   name the suspected dropped detector**; go/no-go is the eval result, not the token delta.
8. **Report** the before/after token delta (and eval result); show both versions. Apply in place only if confirmed.

## Techniques

**Structural first** (move task-specific content to on-demand files — the highest-leverage move:
aggregate rule load, not file count, drives compliance degradation; past the ceiling even frontier
models hold ~68% accuracy, [IFScale, arXiv:2507.11538](https://arxiv.org/abs/2507.11538)), then
**lexical**. Full catalog + worked before/after in [`techniques.md`](techniques.md) — load before compressing.

- **Tables over prose** · **Bullets over sentences** · **Rules over explanations** (cut the *why* unless compliance needs it)
- **Negative constraints** ("Never X" beats enumerating valid options; [negative-space-instructions](https://agentpatterns.ai/instructions/negative-space-instructions/); Remediation: [learn — guardrails-beat-guidance](https://learn.agentpatterns.ai/prompt-engineering/guardrails-beat-guidance/)) · **Examples over descriptions**
- **Front-load critical rules** — attention is U-shaped ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); [Liu et al. 2023, arXiv:2307.03172](https://arxiv.org/abs/2307.03172); Remediation: [learn — lost-in-the-middle](https://learn.agentpatterns.ai/context-engineering/lost-in-the-middle/))
- **Structural compression** — move task-specific content out; don't lexically squeeze it ([instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/); Remediation: [learn — the-ceiling](https://learn.agentpatterns.ai/prompt-engineering/the-ceiling/))

## Guardrails

Naive "make it shorter" breaks both — the non-obvious edges:

- **Don't strip meaning.** Cut ceremony freely; never cut high-density tokens (rationale, conditionals,
  disambiguating names). Removing them shifts interpretive burden to reasoning — a 17% input-token cut
  raised total session cost **67%** ([semantic-density-optimization](https://agentpatterns.ai/context-engineering/semantic-density-optimization/); [Ustynov 2026, arXiv:2604.07502](https://arxiv.org/abs/2604.07502); Remediation: [learn — signal-per-token](https://learn.agentpatterns.ai/context-engineering/signal-per-token/)).
  Step 6's inventory diff enforces this.
- **Compress decisively, not halfway.** Constraint violations peak at *medium* compression (a U-curve,
  97.2% prevalence; [prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/), [arXiv:2512.17920](https://arxiv.org/abs/2512.17920); Remediation: [learn — the-goldilocks-zone](https://learn.agentpatterns.ai/prompt-engineering/the-goldilocks-zone/)) — go all the way to a
  crisp, unambiguous rule, or leave it verbose. A half-trimmed rule hurts compliance *more* than the original.

## Report template

```
# Prompt compression

Baseline: <N> tokens, <R> normative statements
Compressed: <M> tokens (−<X>%), <R> normative statements   ← rule count unchanged

Eval suite: <S> evals — PRE <S>/<S> green → POST <S>/<S> green   ← gate; held on regression (or: n/a — one-off text)
Constraints preserved: <list — must match the step-3 inventory>
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
  Authoring-craft (lexical/structural rewrites), not a soft-compression pipeline.

## Critical rules (read last)

- **Semantics-preserving only.** Dropping a constraint, a conditional, or compliance-critical
  rationale is a bug, not a compression. For a target with an eval suite this is **eval-gated** —
  green before and after, or hold the compression and name the suspected dropped detector
  ([`eval-gate.md`](eval-gate.md)); the eval result is the go/no-go, never the token delta.
- **Recommend; apply only on explicit confirmation.** Show before/after; never invent precedence or
  restructure governance — criticality is a human judgment to surface, not impose.
- **Load [`techniques.md`](techniques.md)** for the catalog + worked example before compressing.
