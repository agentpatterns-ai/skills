# compress-prompt — eval gate (behavioral targets)

Loaded on demand when the compression target **carries behavior**: a skill or instruction artifact
that has — or, by convention, *should* have — a behavioral test/eval suite. This is the behavioral
half of the **semantics-preserving** guardrail: structural checks (linters, schema/format gates)
verify *shape*; only a behavioral suite verifies that no rule or detector was silently weakened.
The go/no-go is the **eval result**, never the token delta.

## When this applies

| Target | Has a suite? | Action |
|---|---|---|
| Skill / instruction artifact | Yes — a behavioral suite exists | Eval-gate it (below). |
| Skill / instruction artifact | No suite yet | **Recommend one and offer to scaffold it** in the target's own eval convention *before* compressing. Don't silently compress un-evalled behavior. |
| One-off prompt / task text | No behavior to regress | Out of scope — compress and report a token delta as normal. |

Use the target repo's **existing** eval convention — do not invent a parallel format.

## The gate

1. **Baseline green (PRE).** Run the target's full suite *before touching anything*. Confirm
   **every** eval passes. A red baseline means fix the suite/target first; you cannot prove a
   compression lossless against a broken oracle.
2. **Compress** (the normal procedure — structural then lexical, inventory-diffed).
3. **Re-run the same suite (POST).** Re-run the *identical* cases against the compressed artifact.
4. **Gate.** PASS only if the POST result equals the PRE result — **every eval still green**. On any
   regression: **hold the compression** (do not commit/apply), and **name the suspected dropped
   rule/detector** by mapping the failing eval back to the section you compressed. Revert or re-do
   that section decisively; a half-trimmed rule hurts compliance more than the original.

## Reporting

Record the gate alongside the token delta (see the SKILL.md report template):

```
Eval suite: <S> evals — PRE <S>/<S> green → POST <S>/<S> green   ← gate; held on any regression
```

If no suite existed: report that, and whether one was scaffolded or the compression was deferred.

## Why it lives in the loop, not at release

Release pipelines usually gate on evals — but at *release* time, after many edits. This gate is
**per-compression and in-loop**: it runs before anything is committed, so a behavioral regression is
caught at the exact compression that caused it, while the suspected dropped rule is still obvious.
(Measured in practice: a full-library compression pass whose structural gates were all green still
required the behavioral suite to prove no detector was weakened — that proof belongs in the loop.)
