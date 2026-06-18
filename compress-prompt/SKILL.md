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

This skill **transforms** prompt/instruction text. It is the fix-side complement
to `audit-instruction-file`, whose **CE-10** check *detects* low-density ceremony;
this skill *applies* the compression. Detector → transform; run them as a pair.

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
3. **Apply techniques** (catalog below), in order: structural first (move
   task-specific content out), then lexical (prose → table/bullet/rule).
4. **Run the compression test** on every surviving line: *"Can I remove this word —
   or this whole sentence — without losing a constraint?"* If yes, cut it.
5. **Verify no constraint was lost.** Diff the compressed text against the step-2
   inventory. Every constraint and every load-bearing conditional must still be
   present. Confirm you compressed **decisively** — no rule left half-trimmed.
6. **Report** the before/after token delta and show both versions. Apply in place
   only if confirmed.

## Techniques catalog

| Technique | What it does | When |
|-----------|--------------|------|
| **Tables over prose** | Rows carry contrast (do/don't, in/out) with zero explanation overhead | Any set of parallel cases or paired rules |
| **Bullets over sentences** | One idea per line; drops transitional/connective filler | Lists of rules or steps buried in paragraphs |
| **Rules over explanations** | State the rule; omit the *why* unless compliance depends on it | "It is important that you ensure X" → `Do X` |
| **Negative constraints** | "Never X" is cheaper and harder to misread than enumerating valid options; greppable | Known failure modes with a clear boundary ([negative-space-instructions](https://agentpatterns.ai/instructions/negative-space-instructions/)) |
| **Examples over descriptions** | One example collapses a pattern + its application into imitable text | Style/format guidance that's easier shown than described |
| **Front-load critical rules** | Highest-cost-to-violate rules first; attention is U-shaped, the middle is weak | Any file with a real dead middle ([Liu et al. 2023, arXiv:2307.03172](https://arxiv.org/abs/2307.03172)) |
| **Structural compression** | Move task-specific content to on-demand/skill files instead of lexically squeezing it | Content that doesn't compress but isn't needed every task — cuts always-loaded cost without losing guidance |

Structural compression is the highest-leverage move: aggregate rule load (across all
always-loaded files), not file count, drives compliance degradation — past the ceiling
even frontier models hold only ~68% accuracy ([IFScale 2025, arXiv:2507.11538](https://arxiv.org/abs/2507.11538)).
Lexically squeezing a rule that doesn't belong in the always-loaded layer is the wrong fix.

## The two guardrails

These are the non-obvious edges. A naive "make it shorter" violates both.

### 1. Don't strip meaning (the compression paradox)

Cutting **ceremony** is free; cutting **meaning** backfires. High-density tokens —
descriptive names, rationale, error messages, disambiguating conditionals — cannot be
removed without shifting the interpretive burden to the model's reasoning phase. A
controlled experiment ([Ustynov 2026, *Beyond Human-Readable*, arXiv:2604.07502](https://arxiv.org/abs/2604.07502))
found a compressed log format cut input tokens **17%** but raised total session cost
**67%**, because the model spent reasoning tokens reconstructing meaning it could have
read directly. Protect high-density tokens; only cut zero-density filler.

### 2. Compress decisively, not halfway (the compliance U-curve)

"Shorter" is **not** monotonically "better." Constraint violations peak at *medium*
compression: a benchmark across instruction-following under compression
([*Separating Constraint Compliance from Semantic Accuracy*, arXiv:2512.17920](https://arxiv.org/abs/2512.17920))
found a universal U-curve (97.2% prevalence), violations worst at ~half compression,
with compliance recovering at *both* the verbose and the extreme-compression ends. A
half-trimmed rule — paraphrased tighter but still loose — hurts compliance *more* than
the verbose original. Compress all the way to a crisp, unambiguous rule, or leave it
verbose. The middle is the worst of both worlds.

## Before / after

Same constraints, far fewer tokens (lexical: rules over explanations, bullets over
sentences):

```markdown
<!-- Before — ~70 tokens -->
## Testing Requirements
It is very important that you make sure all code changes are thoroughly tested
before submitting them for review. You should always write unit tests that cover
the main logic of any function you add or modify. Try to ensure that edge cases are
handled appropriately. Please do not submit code that has not been tested.
```

```markdown
<!-- After — ~28 tokens, 60% smaller -->
## Testing
- Write unit tests for every function added or modified
- Cover edge cases
- Do not submit untested code
```

Every constraint survives (unit tests, edge cases, no-untested-code); only ceremony
("It is very important that…", "Try to…", "Please…") is gone. Note what is *not* cut:
a conditional like `integration tests when the function touches the database, unit
tests otherwise` is meaning, not ceremony — compressing it away applies the rule
uniformly where it should apply selectively.

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

- `audit-instruction-file` — its **CE-10** check *detects* low-density ceremony;
  this skill *fixes* it. Detector → transform; pair them.
- [Prompt Compression](https://agentpatterns.ai/context-engineering/prompt-compression/) — the cornerstone treatment.
- [Semantic Density Optimization](https://agentpatterns.ai/context-engineering/semantic-density-optimization/) — high-density vs zero-density tokens; the compression paradox.
- [Negative Space Instructions](https://agentpatterns.ai/instructions/negative-space-instructions/) — negative constraints that compress guidance.
- [Instruction Compliance Ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/) — why aggregate rule load drives degradation (structural compression).
- See also (out of scope here): *automated* token-level prompt compressors — LLMLingua / LongLLMLingua ([arXiv:2310.05736](https://arxiv.org/abs/2310.05736), [arXiv:2310.06839](https://arxiv.org/abs/2310.06839)). This skill is authoring-craft (lexical/structural rewrites a human or agent applies), not an automated soft-compression pipeline.
