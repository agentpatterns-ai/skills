---
name: audit-instruction-file
description: Audit a single agent instruction file (AGENTS.md, CLAUDE.md, .cursorrules, copilot-instructions.md, GEMINI.md, …) against context-engineering principles and recommend the smallest fix. Invoke when reviewing, writing, or debugging one instruction/memory file, when an agent ignores documented rules within a single file, or when asked to look at that one file and diagnose why a rule "isn't sticking". Skip when the goal is to shorten/tighten the file rather than detect problems (use compress-prompt), when auditing or installing the file's standing self-improvement/learning-loop protocol section (use audit-self-improvement-protocol), when a must-never-fail rule needs moving into a hook/gate (use audit-prompt-hook-placement), when editing ordinary docs or source code, or when rules across peer files with no defined load order contradict each other (use conflicting-instruction-detector) — cross-layer precedence within one load stack (e.g. CLAUDE.md vs a loaded SKILL.md) stays here.
user-invocable: true
version: "0.3.2"
usage: /audit-instruction-file [path-or-dir]
---

# Audit Instruction File

Agent instruction files are *always-loaded* context — in the window every turn, governing
behavior. This skill audits them for the failure modes that make a well-written rule go
unfollowed (wrong position, excess length, discoverable bloat) and recommends the smallest
fix. Project-agnostic: every check runs from the target file's own structure, no
repo-specific knowledge.

**Stance — detect, recommend, and (only on request) apply mechanical fixes.** Default output is
a findings report. On explicit confirmation it applies *semantics-preserving* transforms:
de-duplicate to a single source + pointer; repair or remove broken imports; **re-order imports**
so reference bulk leaves the high-attention edges; add a tiny tail restatement. It does **not**
infer authorial intent and silently restructure a governance file — precedence and criticality are
human judgments the skill surfaces for ratification, not ones it invents.

## Input

- `path-or-dir` (optional): a single file, or a directory to scan. Default: scan the repo root.
- Nothing else required — no project context, no config.

## Scope

Audits instruction **files** — always-loaded markdown. It does **not** audit harness/runtime cache
discipline (deterministic tool ordering, `cache_read` vs `cache_creation` monitoring, compact-by-fork,
mutating tool definitions / the system prompt mid-session): those live in harness *code and config*,
not a file, so they belong to a separate harness/cache audit, not CE checks here. The one
file-detectable slice of caching, **volatile content in the cached prefix**, is CE-9. Session-management
moves (compact/clear/rewind) are runtime, not file. Cross-document *semantic* conflict across a skill
library is `conflicting-instruction-detector`'s lane.

## Procedure

1. **Locate** instruction files anywhere in scope (case-insensitive), pruning `node_modules`/`.git`:
   `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `.cursorrules`, `.windsurfrules`, `CONVENTIONS.md`,
   `.github/copilot-instructions.md`, `.cursor/rules/*.mdc`. Done when every match is listed.
2. **Measure** each file: line count, ~token count (chars/4),
   `##`/`###` headings (section count), and the **normative statements**
   (lines matching `never|always|must|do not|don'?t|required?` — the rule load for CE-8).
   Then build the `@`-import graph: expand each file's `@path` references (up to 5 levels),
   flag any target that does not exist (CE-6), and record the **assembled load order** and
   **total rule load** across all always-loaded files + imports — the inputs CE-7 and CE-8 audit.
   Done when every file has all counts recorded and every import resolved or flagged.
3. **Map attention zones.** First resolve `@path` imports inline (Claude Code expands them
   verbatim at load time) so zones reflect the *assembled* file — an `@other.md` on line 1 puts
   that file's content in primacy and pushes the host body toward the middle/tail. Then treat the
   first ~20% of assembled lines as **primacy**, the last ~20% as **recency**, the rest as the
   **dead middle**. Done when each file's three zone boundaries are recorded.
4. **Run the CE checks** (catalog below) against each file. Collect findings with line references.
   Done when every file × CE check has a recorded verdict — no blanks.
5. **Report** using the output template. One row per finding.
   Done when every finding is a row with severity, check, and line evidence.
6. **Recommend, then (only if asked) apply.** Offer the fix; apply only on explicit
   confirmation. Default to *restate at an edge*, not *relocate*, so wording relied on
   elsewhere isn't silently moved.
   Done when every finding has an offered fix and none was applied without confirmation.

## Checks

Run all ten against each file. The **full catalog — Flags / Why / Fix, each cited — is in
[`checks.md`](checks.md); load it before auditing.** Summary:

| ID | Flags | Default fix |
|----|-------|-------------|
| CE-1 | closing section is a footer, not a guardrail (wasted recency) | restate the top 2–3 rules at the tail (only when earned) |
| CE-2 | a hard `never/always/must` rule stranded in the dead middle | surface at the primacy edge / restate |
| CE-3 | oversized always-loaded file / large dead middle | move scoped guidance to on-demand files |
| CE-4 | discoverable content (dir trees, signatures, versions, config) | delete; replace with a pointer |
| CE-5 | contradiction / unresolved precedence (within file or layer stack) | state precedence, or remove the loser |
| CE-6 | layer hygiene — cross-layer dup, wrong-layer scope, broken `@`-import | one rule, one layer; repair imports |
| CE-7 | `@`-import arrangement corrupts the assembled attention curve | cut bulk, re-order imports, then restate |
| CE-8 | compliance ceiling — too many aggregate normative rules | cut / merge rules; move task-specific out |
| CE-9 | volatile content in the cached prefix (dates, abs paths, IDs) | move to dynamic tail / compute at runtime |
| CE-10 | low semantic density (ceremony / filler) | compress decisively; hand to `compress-prompt` |

To add or change a check, edit `checks.md` (see *Expanding this skill* there); the shelf version is
set uniformly at release, not per change.

## Output template

```
# Instruction-file audit

## <relative/path/to/file>  —  <lines> lines, ~<tokens> tokens, <N> sections

| Severity | Check | Evidence (lines) | Recommendation |
|----------|-------|------------------|----------------|
| High     | CE-2  | L52–61           | Restate "no force-push to <branch>" at an edge |
| Medium   | CE-1  | tail (L130+)     | Add a "Critical rules (read last)" block |
| Low      | CE-4  | L40–48           | Replace command list with a pointer |

**What this file does well:** <1–2 specific positives — e.g. primacy used for the costliest rule.>

**Smallest high-impact change:** <the one edit to make first.>
```

Severity guide: **High** = a safety/correctness guardrail in a weak position or a contradiction;
**Medium** = structural attention waste (CE-1, oversized core); **Low** = bloat that costs tokens
but not correctness.

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)

- **Recommend; apply only on explicit confirmation.** Never infer authorial intent and silently
  restructure a governance file — precedence and criticality are human judgments to surface, not invent.
- **One rule, one layer/source.** Keep a single canonical statement; edges carry pointers or
  *identically-worded* restatements, never divergent copies (the CE-6 trap).
- **Load [`checks.md`](checks.md) before auditing** — the cited CE-1…CE-10 catalog lives there.
