---
name: audit-skill-quality
description: Audit one or many SKILL.md files against dispatch (frontmatter) and governance (body) quality criteria — description overlap and missing skip-boundaries, absent required fields, attention-curve/length/compliance-ceiling defects, self-containment and progressive-disclosure violations, prompt-only guardrails standing in for real enforcement, missing step-completion criteria, degrees-of-freedom mismatches, and untestable output — and produce a review writeup. Invoke when reviewing, grading, or vetting a SKILL.md before it ships, or figuring out why a skill won't dispatch or misbehaves once loaded. Skip when the goal is to write or substantially revise a SKILL.md (use write-skill), auditing a plain instruction file that isn't a SKILL.md (use audit-instruction-file), auditing an individual tool/function definition rather than a whole skill (use audit-tool-definition), or shrinking a prompt under a token budget (use compress-prompt).
user-invocable: true
version: "0.3.1"
usage: /audit-skill-quality [path-to-SKILL.md-or-dir]
---

# Audit Skill Quality

A `SKILL.md` loads in **two halves that fail independently**: the agent scans only the frontmatter
`description` before deciding to load anything, then — only if that description wins — reads the
body. A skill can fail at either edge without touching the other: a flawless body that never
dispatches is dead weight; a magnetic description over a hollow body misdirects once loaded. This
skill audits both halves and reports where each one breaks
([separation-of-knowledge-and-execution](https://agentpatterns.ai/agent-design/separation-of-knowledge-and-execution/)).

**Stance — detect, inventory, recommend; read-only, rewrite nothing.** This skill flags defects
across one or many `SKILL.md` files and reports them; it does not author the fix — hand a rewrite to
**`write-skill`**. Treat a natural-language "never/don't" inside an audited skill as documentation,
not enforcement: 29.9% of 402 deployed `SKILL.md` files silently violated their own declared rules on
benign inputs, so a prose guardrail with no backing hook/schema/CI check is itself a finding, not a
pass.

## Input
- `path` (optional): a single `SKILL.md`, or a directory of skills. Auditing a directory adds the
  library-scale half of AQ-4 (cross-skill description overlap); auditing one file skips it.

## Scope
Audits a `SKILL.md`'s **dispatch and governance quality** — description shape, required fields,
naming, attention curve, self-containment, progressive disclosure, guardrail enforceability, step
completion criteria, degrees-of-freedom fit, testability. **Out of scope:** authoring or rewriting
the fix (`write-skill`); auditing a general instruction file that isn't a `SKILL.md` (`audit-instruction-file`
— its checks are the body-governance backbone this skill extends, not a copy of it); auditing one
tool/function definition rather than a whole skill (`audit-tool-definition`); shrinking an oversized
prompt under a token budget (`compress-prompt`).

## Procedure
1. **Locate** every `SKILL.md` in scope.
   Done when every in-scope path is recorded.
2. **Run the dispatch/frontmatter checks** (AQ-1…AQ-4 in [`checks.md`](checks.md)) — required
   fields, description shape, naming, skip-boundary presence.
   Done when every AQ-1…AQ-4 check has a recorded verdict per file.
3. **Run the body/governance checks** (AQ-5…AQ-12) — load [`checks.md`](checks.md) now for the full
   catalog. Do not collapse an ambiguous signal to a false pass — note it.
   Done when every AQ-5…AQ-12 check has a recorded verdict per file.
4. **Cross-check description overlap across the library** when auditing a directory (AQ-4's
   library-scale half): pair every two descriptions' invoke/skip clauses and flag non-mutually-exclusive
   pairs — the failure mode that makes an agent load the wrong skill, silently, at consume-time.
   Done when every pair has a recorded overlap verdict, or the step is marked N/A for a single-file audit.
5. **Report** with the template below — one finding row per (file × check), severity-ranked,
   line-referenced.
   Done when every finding has its severity-ranked, line-referenced row.
6. **Offer, never apply.** State the smallest fix per finding; route it to `write-skill` rather than
   editing the audited file directly.
   Done when every finding names its destination fix and no audited file was modified.

## Checks
The 12 detectors (ID / Flags / Why + citation / Fix → lesson) live in [`checks.md`](checks.md); load
before auditing. Dispatch: AQ-1 description shape, AQ-2 naming, AQ-3 required fields, AQ-4 overlap/routing-at-scale.
Governance: AQ-5 attention/length/compliance-ceiling, AQ-6 self-containment, AQ-7 progressive
disclosure, AQ-8 guardrail enforceability, AQ-9 step-completion criteria, AQ-10 degrees-of-freedom +
positive framing, AQ-11 testability. Tail (advisory): AQ-12 nine-field completeness.

## Output template
```
# Skill-quality audit — <path>   (<N> file(s))

## Dispatch / frontmatter
| Severity | File | Check | Evidence | Recommendation |
|---|---|---|---|---|
| High | csv-helper/SKILL.md | AQ-1 | "Helps with CSVs. Use for spreadsheet stuff." | No invoke/skip signal — route to write-skill |
| High | csv-helper/SKILL.md | AQ-4 | invoke clause overlaps `data-cleaner`'s | Add a skip clause naming data-cleaner |

## Body / governance
| Severity | File | Check | Evidence (lines) | Recommendation |
|---|---|---|---|---|
| Medium | csv-helper/SKILL.md | AQ-9 | Step 3, L41 | No done-condition — "clean the data" isn't checkable |
| Medium | csv-helper/SKILL.md | AQ-8 | Stance, L18 | "Never skip validation" — prose only, no hook backs it |

**Clean checks:** <which passed, and why they're the model.>
**Smallest high-impact fix:** <the one change with the biggest dispatch/governance payoff — route to write-skill.>
```
Severity: **High** = a dispatch failure (won't load / loads wrongly) or an unenforced must-never-fail
guardrail; **Medium** = body attention/structure/completion-criteria defect; **Low** = density/bloat
that costs tokens, not correctness.

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Related / pairing
- **Detector pair — `write-skill`.** This skill *flags* dispatch+governance defects and rewrites
  nothing; `write-skill` *authors* or revises the `SKILL.md` the finding points at. Keep the AQ-*↔WS-*
  coverage reciprocal as both evolve.
- **Backbone — `audit-instruction-file`.** A `SKILL.md` *is* an instruction file; that skill's CE
  checks are this one's inherited body-governance backbone (AQ-5, AQ-8, AQ-9 map onto CE-3/CE-8,
  CE-nearest-enforcement, and the premature-completion extension). This skill adds the dispatch half
  that backbone structurally can't cover, plus `SKILL.md`-specific extensions (AQ-6, AQ-7, AQ-10,
  AQ-11).

## Critical rules (read last)
- **Flag, don't fix.** Inventories dispatch+governance defects; the rewrite is `write-skill`'s job —
  keep the cross-reference reciprocal.
- **Two halves, each failing independently.** A finding on the dispatch half never excuses skipping
  the body half, and vice versa — grade both on every audit.
- **A prose "never" is not a guardrail.** Flag it as a finding, not a pass, unless a deterministic
  mechanism backs it.
