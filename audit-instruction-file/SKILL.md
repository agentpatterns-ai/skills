---
name: audit-instruction-file
description: Audit any agent instruction file (AGENTS.md, CLAUDE.md, .cursorrules, copilot-instructions.md, GEMINI.md, …) against context-engineering principles and recommend the smallest fix. Invoke when reviewing, writing, or debugging an instruction/memory file, or when an agent keeps ignoring documented rules. Skip when editing ordinary docs or source code.
user-invocable: true
version: "0.1.0"
usage: /audit-instruction-file [path-or-dir]
---

# Audit Instruction File

Agent instruction files are *always-loaded* context: they sit in the window for every
turn and govern behavior. This skill audits them for the failure modes that make a
well-written rule go unfollowed — wrong position, excess length, discoverable bloat —
and recommends the smallest fix. It is project-agnostic: every check runs from the
target file's own structure, with no knowledge of any specific repo.

**Stance — detect, recommend, and (only on request) apply mechanical fixes.** By default it
produces a findings report. On explicit confirmation it applies *semantics-preserving* transforms:
de-duplicate to a single source plus a pointer, repair or remove broken imports, **re-order imports**
so reference bulk leaves the high-attention edges, and add a tiny tail restatement. It does **not**
infer authorial intent and silently restructure a governance file — precedence and criticality are
human judgments the skill surfaces for ratification, not ones it invents.

## Input

- `path-or-dir` (optional): a single file, or a directory to scan. Default: scan the repo root.
- Nothing else required — no project context, no config.

## Scope

Audits instruction **files** — always-loaded markdown. It does **not** audit harness/runtime cache
discipline: deterministic tool ordering, `cache_read` vs `cache_creation` monitoring, compact-by-fork,
or mutating tool definitions / the system prompt mid-session. Those live in harness *code and config*,
not in a file, so they belong to a separate harness/cache audit — not CE checks here. The one
file-detectable slice of caching, **volatile content in the cached prefix**, is covered by CE-9.
(Same boundary applies to session-management moves like compact/clear/rewind — runtime, not file.)
Cross-document *semantic* conflict across a skill library is `conflicting-instruction-detector`'s lane.

## Procedure

1. **Locate** instruction files. Match these names anywhere in scope (case-insensitive):
   ```sh
   # from repo root (or the given dir)
   find . -type d \( -name node_modules -o -name .git \) -prune -o -type f \
     \( -iname 'AGENTS.md' -o -iname 'CLAUDE.md' -o -iname 'GEMINI.md' \
        -o -iname '.cursorrules' -o -iname '.windsurfrules' -o -iname 'CONVENTIONS.md' \
        -o -ipath '*/.github/copilot-instructions.md' -o -ipath '*/.cursor/rules/*.mdc' \) -print
   ```
2. **Measure** each file:
   ```sh
   lines=$(wc -l < "$f"); chars=$(wc -m < "$f"); echo "$f: $lines lines, ~$((chars/4)) tokens"
   ```
   Also count `##`/`###` headings (section count) and the number of **normative statements**
   (lines matching `never|always|must|do not|don'?t|required?` — the rule load for CE-8).
   Then build the `@`-import graph: for each file, expand `@path` references (up to 5 levels),
   flag any target that does not exist (CE-6), and record the **assembled load order** and the
   **total rule load** across all always-loaded files + imports — the inputs CE-7 and CE-8 audit.
3. **Map attention zones.** First resolve any `@path` imports inline (Claude Code expands them
   verbatim at load time), so zones reflect the *assembled* file — an `@other.md` on line 1 puts
   that file's content in primacy and pushes the host file's body toward the middle/tail. Then
   treat the first ~20% of the assembled lines as the **primacy** zone, the last ~20% as the
   **recency** zone, and everything between as the **dead middle**.
4. **Run the CE checks** (catalog below) against each file. Collect findings with line references.
5. **Report** using the output template. One row per finding.
6. **Recommend, then (only if asked) apply.** Offer the fix; apply only on explicit
   confirmation. Default to *restate at an edge*, not *relocate*, so wording the user
   relies on elsewhere is not silently moved.

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
set uniformly at release, not bumped per change.

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

## Critical rules (read last)

- **Recommend; apply only on explicit confirmation.** Never infer authorial intent and silently
  restructure a governance file — precedence and criticality are human judgments to surface, not invent.
- **One rule, one layer/source.** Keep a single canonical statement; edges carry pointers or
  *identically-worded* restatements, never divergent copies (the CE-6 trap).
- **Load [`checks.md`](checks.md) before auditing** — the cited CE-1…CE-10 catalog lives there.
