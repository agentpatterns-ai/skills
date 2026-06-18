---
name: audit-instruction-file
description: Audit any agent instruction file (AGENTS.md, CLAUDE.md, .cursorrules, copilot-instructions.md, GEMINI.md, …) against context-engineering principles and recommend the smallest fix. Invoke when reviewing, writing, or debugging an instruction/memory file, or when an agent keeps ignoring documented rules. Skip when editing ordinary docs or source code.
user-invocable: true
version: "0.6.0"
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
4. **Run the checks** below against each file. Collect findings with line references.
5. **Report** using the output template. One row per finding.
6. **Recommend, then (only if asked) apply.** Offer the fix; apply only on explicit
   confirmation. Default to *restate at an edge*, not *relocate*, so wording the user
   relies on elsewhere is not silently moved.

## Checks

Each check is self-contained: an ID, what it **Flags**, **Why** (with a source), and the **Fix**.
Add new checks here as new principles are learned (see *Expanding this skill*).

### CE-1 — Wasted recency slot
- **Flags:** the closing section is a footer (links, related, changelog, credits, low-stakes
  housekeeping) rather than a rule that must be obeyed.
- **Why:** transformer attention is U-shaped — strongest at the start and end, weakest in the
  middle (Liu et al. 2023, *Lost in the Middle*, arXiv:2307.03172). The tail is the second
  high-attention zone; spending it on a footer wastes it.
- **Fix:** add a short *"Critical rules (read last)"* block that **restates** the 2–3
  highest-cost-to-violate rules, **identically worded** (varied wording reads as a *conflicting*
  rule — see CE-6). This is deliberate primacy+recency repetition (Critical Instruction
  Repetition), the one intentional exception to CE-6's de-dup rule. Restate, do not relocate.
- **Precedence:** if the file is heavily `@`-imported, fix the *arrangement* first (CE-7) — a
  restatement on top of a curve already corrupted by bulky imports just adds tokens. Restate only
  when the assembled tail genuinely lacks a guardrail.
- **Skip the restatement when it isn't earned** — a tail echo is a *targeted* fix, not a default:
  - the file is **short** (no real dead middle — repetition is just noise);
  - **several rules are already repeated** (repeating everything erases the priority signal — the
    edge slot only works for the *few* genuinely critical rules);
  - the file is **not in the assembled global recency position** (e.g. an `@`-imported file that
    lands mid-context under Claude Code — see *Precedence*);
  - the restatement would **materially grow the file** — weigh the recency gain against CE-3/CE-8/CE-10.

  When none of these hold and the tail is a footer, the restatement is warranted; otherwise CE-1's
  finding is "tail under-used" with **no** fix, or a re-order (CE-7) rather than an echo.

### CE-2 — Hard guardrails in the dead middle
- **Flags:** *never / always / must / do not* rules whose violation is expensive, located in
  the middle zone.
- **Why:** identical instructions are followed less reliably in the middle than at the edges
  (Liu et al. 2023). A guardrail in the dead middle is technically present but practically weak.
- **Fix:** surface each critical guardrail at the primacy edge, and/or restate it in the
  CE-1 closing block. Keep the canonical statement once; the edge copy is a pointer/restatement.

### CE-3 — Oversized file / large dead middle
- **Flags:** an always-loaded file that is long (rough trigger: root instruction file > ~50
  lines, or growing unbounded). The longer the file, the larger its low-attention middle.
- **Why:** output quality degrades as context fills — "context rot," a gradient not a cliff,
  with onset nearer an absolute token threshold than a fixed % (Anthropic, *Effective Context
  Engineering*; RULER, arXiv:2404.06654). Every line displaces reasoning and buries its neighbours.
- **Fix:** move path/topic-scoped guidance into files that load **on demand** (e.g. nested
  per-directory files, path-scoped rule files, or linked references) so the always-loaded core
  stays small. Cut any line whose removal would not change agent behavior.

### CE-4 — Discoverable content (bloat)
- **Flags:** any line that fails the inclusion test — *can the agent discover this itself with
  read / grep / glob?* If yes, it does not belong. Common offenders:
  directory trees & file structure; API signatures and function params; dependency versions
  (readable from lockfiles); code/naming conventions visible in any existing file; config files
  (`tsconfig`, `.eslintrc`, `pyproject.toml`); test patterns readable from tests; tech-stack
  restatements; standards duplicated from a linked file.
- **Why:** instruction files load on every interaction, so each line is a recurring tax paid
  before any task token. Discoverable content also creates a **second source of truth that goes
  stale** (a directory tree is wrong within a sprint; the agent then follows the stale copy). A
  controlled eval found structural overviews raise inference cost **>20% with no task-success
  gain** — agents given high-level structure explore *more broadly, not more precisely*
  (Shi et al., *Evaluating AGENTS.md*, 2026, arXiv:2602.11988). Only **non-discoverable** context
  earns space: rationale ("why X over Y"), constraints/gotchas not encoded in code, domain rules,
  conventions present-but-unstated, out-of-band integrations.
- **Fix:** delete discoverable lines. Where *direction* helps, replace the copy with a **pointer**
  — a path, not a duplicate ("use the repository pattern in `src/repos/`"). Litmus for each kept
  line: *would it still be true and useful if the codebase changed next week?*
- **Caveat:** if the target agent lacks exploration tools, structure is non-discoverable *to it* —
  audit actual tool access before flagging. In high-churn repos, keep structural pointers in a
  separate frequently-updated file, not the main always-loaded one.

### CE-5 — Contradiction / unresolved layering
- **Flags:** two rules that conflict — within one file, or across the layer stack
  (system prompt → project `AGENTS.md`/`CLAUDE.md` → skill `SKILL.md` → user message) — or a rule
  silently superseded by a later one, with no precedence stated.
- **Why:** instructions arrive from multiple layers at once. Precedence follows **specificity**
  (closer-to-task wins) but is a behavioral tendency, not an enforced rule, so unresolved conflicts
  make behavior unpredictable and position-dependent.
- **Fix:** state the precedence explicitly, or remove the loser. Keep one source of truth per rule.

### CE-6 — Layer hygiene
- **Flags:** structural layering defects detectable across the files in scope:
  - **Duplication across layers** — the same *substantive* convention copied into both a project
    file and a skill (or a `@`-imported file and its host). Changing one copy authors a silent
    contradiction. **Carve-out:** a short hard-constraint repeated *identically* at a file's primacy
    and recency edges is intentional (CE-1 / Critical Instruction Repetition) — do **not** flag it.
  - **Contradictory restatement** — the same rule stated two ways (e.g. edge vs middle) with
    different wording; the model reads them as conflicting constraints. Collapse to one wording.
  - **Wrong-layer scope** — a *task-specific* rule sitting in the always-loaded project layer
    (belongs in a skill or a path-scoped rule), or a universal rule buried only in one skill.
  - **Broken / silent `@`-import** — an `@path` whose target does not exist or moved; Claude Code
    loads fewer instructions with **no error**. Resolve every `@path` and confirm the file exists.
  - **Sub-agent injection gap** *(advisory)* — a sub-agent definition that relies on project
    conventions it never restates; sub-agents start fresh and inherit nothing unless passed.
- **Why:** each convention should live in exactly one layer; duplication drifts, mis-scoped rules
  tax every task, broken imports fail invisibly, and un-injected conventions simply don't reach
  sub-agents.
- **Fix:** one rule, one layer — others point or stay silent. Repair or remove dead imports. Move
  task-specific rules down to the skill/path-scoped layer. Pass critical constraints into
  sub-agents explicitly.

### CE-7 — Composition corrupts the assembled curve
- **Flags:** the `@`-import arrangement degrades the attention curve of the *assembled* file
  (audit the resolved assembly from the procedure, not raw files):
  - **Displacement** — a bulky reference import near the top pushes the host file's hard rules out
    of primacy and into the dead middle; or a large import at the end buries the real tail guardrail.
  - **Duplicated bulk** — the same imported content pulled into more than one place, wasting edge
    real estate and inflating the middle (distinct from CE-6's *rule*-level duplication).
- **Why:** Claude Code expands `@path` imports verbatim at load time, so the U-shaped curve applies
  to the concatenated whole. Arrangement *is* positioning: where you place an import decides what
  occupies primacy/recency and what falls into the low-attention middle.
- **Fix (in order):** (1) cut discoverable/duplicated bulk first (CE-3/CE-4/CE-6); (2) **re-order
  imports** so reference material sits in the middle and rule-bearing files at the edges; (3) only
  then restate (CE-1). Output the proposed assembled order as a recommendation; apply the re-order
  only on confirmation (per *Stance*) — never infer intent and rewrite.

### CE-8 — Compliance ceiling
- **Flags:** the assembled always-loaded context carries **too many rules** — high total normative
  load across all files + imports (rough trigger: many tens of `never/always/must` statements), even
  if each individual file is short.
- **Why:** aggregate instruction load — not file count — drives degradation; past the compliance
  ceiling the model silently drops lower-priority rules (usually the project layer). Ten rules in a
  flat prompt outperform forty spread across layers. A short file can still exceed the ceiling.
- **Fix:** cut rules the agent doesn't need every task (move task-specific ones to skills /
  path-scoped rules that load on demand), merge overlapping rules, and reserve edge-repetition
  (CE-1) for the *few* genuinely critical constraints so the priority signal survives.

### CE-9 — Cache-busting volatile content in the prefix
- **Flags:** an always-loaded instruction file (part of the cached static prefix) that embeds
  **volatile, per-session content** — timestamps / dates ("as of <date>", "last updated …"),
  absolute machine paths, `cwd`, hostnames, usernames, session/run IDs, env-specific values, or
  any "current state" that changes between runs or machines.
- **Why:** prompt caching reuses an exact byte-identical prefix at ~10% of base price; any volatile
  token in the prefix forces a **cache write (125–200%) every turn**, silently — a miss never errors,
  it just bills. Volatile-in-prefix also loses on attention: it's noise that dilutes the real rules.
  (Static prose that merely *describes* a path is fine — flag values that actually change per run.)
- **Fix:** move volatile/state content to the dynamic tail or compute it at runtime; keep the
  always-loaded file static and byte-stable across sessions and machines.

### CE-10 — Low semantic density (ceremony / filler)
- **Flags:** lines that carry little guidance per token — hedging and filler ("it is important that
  you ensure…", "please try to…"), prose where a table/bullet/rule would be denser, explanations
  with no compliance value, restated standard conventions the model already knows.
- **Why:** verbose instructions don't improve accuracy — they bury the real rules (Anthropic:
  bloated files get ignored). But "shorter" is **not** monotonically better: constraint violations
  peak at *medium* compression (arXiv:2512.17920), and stripping *meaning* (not ceremony) shifts the
  burden to reasoning (Ustynov 2026 — a 17% input-token cut raised session cost 67%).
- **Fix:** apply the compression test (cut any word/sentence that changes no behavior); convert prose
  to tables / bullets / rules; compress **decisively** to a crisp, unambiguous rule — never leave it
  half-trimmed (the U-curve trough). Keep rationale only where compliance on unforeseen inputs needs
  it. If content doesn't compress but isn't needed every task, move it to an on-demand skill
  (structural compression — complements CE-3). To *apply* the fix, hand the flagged section to the
  **`compress-prompt`** skill — this detector pairs with that transform.

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

## Principles (why these checks exist)

- **Attention is U-shaped** — edges beat the middle, regardless of content importance
  (Liu et al. 2023, arXiv:2307.03172; corroborated by RULER, arXiv:2404.06654).
- **Context degrades with length** ("context rot") — onset is nearer an absolute token
  threshold than a fixed percentage (Anthropic, *Effective Context Engineering for AI Agents*).
- **Only non-discoverable context belongs in always-loaded files** — anything the agent can
  find by reading or listing is pollution.
- **Instructions layer by specificity** (system → project → skill → user); each convention belongs
  in exactly one layer.
- **The compliance ceiling caps rule count** — aggregate normative load, not file count, drives
  degradation; past the ceiling low-priority rules are silently dropped.
- **`@`-imports assemble verbatim** — the U-curve and the rule load apply to the *concatenated*
  whole, so import arrangement is itself an attention-positioning decision.
- **Always-loaded files are the cached prefix** — static, byte-stable content caches at ~10% of
  base; volatile content in the prefix forces a full-price cache write every turn, silently.
- **Compress to crisp, not merely short** — density raises compliance, but violations peak at
  *medium* compression and stripping meaning shifts cost to reasoning; compress decisively or stay verbose.
- **Attention-edge placement and static-first caching are compatible, not opposed** — a static
  instruction block with rules at its own edges satisfies both. Only volatile-in-prefix (CE-9) loses
  on both axes; a tail restatement (CE-1) is cache-safe but worth its tokens only when length and
  criticality justify it — which is why CE-1 is conditional, not a default.

## Expanding this skill

This catalog grows as new context-engineering principles are learned. To add a check:

1. Append a `### CE-N — <name>` block with **Flags / Why / Fix**, one principle per check, and
   cite a primary source in *Why*.
2. If the principle is a *structural* property (position, length, density), make the check
   detectable from file structure alone so it stays project-agnostic.
3. Add a row example to the output template only if it introduces a new severity rationale.
4. Bump `version` on any contract change (new/changed check IDs or output shape).

Keep every check independent: a reader should be able to run one check without the others.
