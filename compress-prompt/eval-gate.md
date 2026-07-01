# compress-prompt — eval gate (behavioral targets)

Loaded on demand when the compression target **carries behavior**: a skill or instruction artifact
that has — or, by convention, *should* have — a behavioral eval suite. This is the behavioral half of
the **semantics-preserving** guardrail: the deterministic gates (`check-citations` / `check-dispatch`
/ `check-frontmatter`) verify *structure*; only the eval suite verifies that no detector was silently
weakened. The go/no-go is the **eval result**, never the token delta.

## When this applies

| Target | Has a suite? | Action |
|---|---|---|
| Skill / instruction artifact (this factory) | Yes — `harness/<name>/evals/` | Eval-gate it (below). |
| Skill / instruction artifact | No suite yet | **Recommend one and offer to scaffold it** (`scaffold-skill` / `map-sources` conventions) *before* compressing. Don't silently compress un-evalled behavior. |
| One-off prompt / task text | No behavior to regress | Out of scope — compress and report a token delta as normal. |

"Eval suite" = the factory harness convention only — folder-per-check `input/` + `expected.md` +
`judge.md` (+ `manifest.json`). Do **not** invent a parallel eval format.

## The gate

1. **Baseline green (PRE).** Run the target's full suite *before touching anything* — output +
   precision + trigger evals, one isolated sub-agent per case (with-skill vs baseline, LLM-judged
   per the factory `evals/` machinery). Confirm **every** eval passes. A red baseline means fix the
   suite/skill first; you cannot prove a compression lossless against a broken oracle.
2. **Compress** (the normal procedure — structural then lexical, inventory-diffed).
3. **Re-run the same suite (POST).** Re-run the *identical* cases against the compressed artifact.
4. **Gate.** PASS only if the POST result equals the PRE result — **every eval still green**. On any
   regression: **hold the compression** (do not commit/apply), and **name the suspected dropped
   detector** by mapping the failing eval's `check` back to the body section you compressed (e.g.
   a failing `negative-constraints` output eval → you over-trimmed the "Never log …" boundary).
   Revert or re-do that section decisively; a half-trimmed rule hurts compliance more than the
   original.

## Reporting

Record the gate alongside the token delta (see the SKILL.md report template):

```
Eval suite: <S> evals — PRE <S>/<S> green → POST <S>/<S> green   ← gate; held on any regression
```

If no suite existed: report that, and whether one was scaffolded or the compression was deferred.

## Why it lives in the loop, not at release

`grade-skill` / `release` already gate on evals — but at *release* time, after many edits. This gate
is **per-compression and in-loop**: it runs before anything is committed, so a behavioral regression
is caught at the exact compression that caused it, while the suspected dropped detector is still
obvious. (Surfaced dogfooding compress-prompt across all 18 factory skill bodies, PR #56: the
deterministic gates passed, but only the manual ~201-eval run proved no detector was weakened — that
proof belongs in the skill.)
