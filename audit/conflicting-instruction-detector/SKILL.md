---
name: conflicting-instruction-detector
description: Surface semantic conflicts across two or more agent instruction files / a skill library — pairs of independently-authored rules in different files that contradict in effect though they share no wording — and present them as candidate conflicts for human review, never auto-resolved. Invoke only when the concern is cross-file — rules in different files disagree, a multi-file instruction set or skill library needs checking for cross-document contradictions, or an agent behaves inconsistently because rules in separate files conflict. Skip when the target is a single file's own attention/density or within-file contradictions and precedence (use audit-instruction-file), or when you want a conflict mechanically fixed.
user-invocable: true
version: "0.2.1"
usage: /conflicting-instruction-detector [path-or-dir]
---

# Conflicting-Instruction Detector

A skill library / multi-file instruction set accumulates rules from many authors; two can quietly
contradict in *effect* while sharing no wording and living in different files. This skill reads the
whole set and surfaces those **candidate** conflicts for a human to adjudicate. The cross-document,
**semantic** lane — its sibling `audit-instruction-file` (CE-5 / CE-6) already owns within-one-file
and layer-stack contradiction + precedence.

**Stance — detect and surface for human review; read-only; never resolve, never auto-fix.** By
design, not a limitation. Semantic cross-document detection is *unreliable*: even strong models drop
to 74.6% F1 on cross-type conflicts — bottoming at 60.3% on the worst pair ([ConInstruct, arXiv:2511.14342](https://arxiv.org/abs/2511.14342)),
so many flags are false positives — every flag is a **review candidate, never a confirmed conflict**,
and there is **no `--fix` / auto-resolve path**. Measured: GPT-4o directly generates a response in
97.5% of cases on instructions carrying 1–2 conflicts, silently resolving rather than flagging
(ConInstruct) — so this skill forces the surface and hands the choice to a human.

## Input
- `path-or-dir` (optional): instruction set / skill library to scan — a dir of `SKILL.md`s,
  `AGENTS.md`/`CLAUDE.md`, rule files. Default: scan the repo for instruction files.
- Needs **≥2 documents** — a single file's internal contradictions are `audit-instruction-file`'s job.

## Scope
Surfaces **cross-document semantic** conflicts: two rules in *different* files that cannot both be
satisfied for the same input. **Out of scope** (hand off, do not duplicate): a single file's
attention/length/density and its *within-file* / *layer-stack* contradictions + precedence — those are
`audit-instruction-file` CE-5 / CE-6; and **resolving** any conflict, picking a precedence winner, or
editing files. It reports; the human decides.

## Procedure
1. **Collect** every instruction document in scope; record each rule with file + line. ≥2 docs required.
   Done when every in-scope document's rules are recorded with file + line.
2. **Pair.** Enumerate every pair of rules prescribing *different* actions — the candidate set.
   Done when every differing-action rule pair is in the candidate set.
3. **Overlap gate — run BEFORE ranking; the main false-positive filter.** For each candidate pair,
   try to construct a *concrete input that triggers both rules at once*. If you cannot — their
   conditions are disjoint (different branch type, subtree, file glob, role, audience) — the rules
   **do not conflict**: route to *likely-not-a-conflict*, drop from the queue. Only pairs with a real
   overlapping input proceed. At the ~60% F1 floor the dominant error is rules that look opposite but
   apply to disjoint scopes; this gate removes them.
   Done when every candidate pair is routed — proceeds with a named overlapping input, or listed
   likely-not-a-conflict.
4. **Classify** each surviving pair against the conflict classes in [`checks.md`](checks.md) — load it
   now — confirming both rules genuinely cannot hold on that overlapping input.
   Done when every surviving pair carries a conflict class.
5. **Rank by review-worthiness**, not certainty: `(confidence the conflict is real) × (cost if an
   agent follows the wrong one)`. Surface high-cost pairs first even at middling confidence — that is
   what a human-review queue is for.
   Done when every pair has a recorded rank score.
6. **Write a clarifying question per candidate** — never a resolution. State the apparent
   contradiction and ask which rule governs (or whether they coexist), because the skill must not
   silently pick ([ConInstruct recognition–communication gap](https://arxiv.org/abs/2511.14342)).
   Done when every candidate has its clarifying question — no blanks.
7. **Report** with the template. Label everything *candidates to review*, not findings.
   Done when the filled template lists every candidate.

## Conflict classes & triage
Classes (`CI-1` direct cross-document contradiction … `CI-5` cross-doc drift), each with Flags / Why /
human-review question, live in [`checks.md`](checks.md) — loaded on demand. Each is read-only and emits
a question, not a fix.

## Output template
```
# Candidate instruction conflicts — <target>   (review queue, not findings)

⚠ Detection of semantic conflicts is unreliable (74.6% F1 cross-type, worst pair 60.3% — ConInstruct).
Each row is a CANDIDATE to confirm — not a confirmed conflict. Nothing here is auto-resolved.

| Rank | Class | Rule A (file:line) | Rule B (file:line) | Apparent contradiction | Question for you |
|------|-------|--------------------|--------------------|------------------------|------------------|
| 1 | CI-2 | testing.md:12 "always run the full suite before commit" | speed.md:4 "skip slow tests in pre-commit; run them in CI" | both fire pre-commit; can't run-all and skip-slow at once | Which governs pre-commit — full suite or fast subset? |
| 2 | CI-3 | CLAUDE.md:8 "use tabs" | style/SKILL.md:20 "indent with 2 spaces" | same files, different indent; no precedence stated | Which file wins, or is one scoped to a subtree? |

**Likely NOT a conflict (shown for transparency):** <pairs that look like conflicts but coexist —
e.g. different scopes/conditions — so the reader sees the false-positive surface, not just the hits.>

**Nothing here is resolved.** Pick a winner per row, then apply it yourself or with
`audit-instruction-file` (within a file) — this skill does not edit.
```

## Related / pairing
- **Sibling, not overlap** — `audit-instruction-file` CE-5 / CE-6 owns *within-file / layer-stack* +
  *lexical* contradiction + precedence, and may apply fixes. This owns *cross-document semantic*
  conflict and **never applies anything**. Keep the two stances in sync (see this skill's source map).
- Sits beside `audit-lethal-trifecta` in the audit-shaped detector family.
- Sources: [ConInstruct, arXiv:2511.14342](https://arxiv.org/abs/2511.14342) (detection unreliability +
  recognition–communication gap); [WIRE, arXiv:2605.27784](https://arxiv.org/abs/2605.27784) (within-policy
  conflicts common; diagnostics *conditional, not deployment-frequency*); [SEFZ, arXiv:2605.13044](https://arxiv.org/abs/2605.13044)
  (a natural-language label is not enforcement).

## Critical rules (read last)
- **Run the overlap gate before ranking.** A pair with no single input that triggers *both* rules is
  not a conflict — route it to likely-not-a-conflict, never to Rank 1. Rules that read as opposites
  but apply to disjoint scopes (e.g. `never force-push` to shared history vs `force-push ok` on your
  own un-pulled branch) are the top false-positive source; this gate removes them.
- **Flag, never resolve.** No `--fix`, no auto-resolve, no precedence-winner picked by the skill — the
  human adjudicates. The headline contract, not a footnote.
- **Every row is a candidate, not a finding.** 74.6% F1 cross-type (60.3% floor) means expect false
  positives; present the likely-not-conflict pairs too so the skill is not over-trusted.
- **Read-only.** It reports a review queue and asks questions; it edits nothing.
