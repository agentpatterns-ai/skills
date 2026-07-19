# audit-instruction-file — CE check catalog

Loaded on demand by `audit-instruction-file`. The full `CE-1…CE-10` checks (the summary table in
`SKILL.md` points here), the principles behind them, and how to add a check. Each check is
self-contained: an ID, what it **Flags**, **Why** (with a source), and the **Fix**.

## Contents
- [CE-1 — Wasted recency slot](#ce-1--wasted-recency-slot)
- [CE-2 — Hard guardrails in the dead middle](#ce-2--hard-guardrails-in-the-dead-middle)
- [CE-3 — Oversized file / large dead middle](#ce-3--oversized-file--large-dead-middle)
- [CE-4 — Discoverable content (bloat)](#ce-4--discoverable-content-bloat)
- [CE-5 — Contradiction / unresolved layering](#ce-5--contradiction--unresolved-layering)
- [CE-6 — Layer hygiene](#ce-6--layer-hygiene)
- [CE-7 — Composition corrupts the assembled curve](#ce-7--composition-corrupts-the-assembled-curve)
- [CE-8 — Compliance ceiling](#ce-8--compliance-ceiling)
- [CE-9 — Cache-busting volatile content in the prefix](#ce-9--cache-busting-volatile-content-in-the-prefix)
- [CE-10 — Low semantic density (ceremony / filler)](#ce-10--low-semantic-density-ceremony--filler)
- [Principles (why these checks exist)](#principles-why-these-checks-exist)
- [Expanding this skill](#expanding-this-skill)

## CE-1 — Wasted recency slot
- **Flags:** the closing section is a footer (links, related, changelog, credits, low-stakes
  housekeeping) rather than a rule that must be obeyed.
- **Why:** transformer attention is U-shaped — strongest at the start and end, weakest in the
  middle (Liu et al. 2023, *Lost in the Middle*, arXiv:2307.03172;
  [corpus — lost-in-the-middle](https://agentpatterns.ai/context-engineering/lost-in-the-middle/)).
  The tail is the second high-attention zone; spending it on a footer wastes it.
- **Fix:** add a short *"Critical rules (read last)"* block that **restates** the 2–3
  highest-cost-to-violate rules, **identically worded** (varied wording reads as a *conflicting*
  rule — see CE-6). This is deliberate primacy+recency repetition (Critical Instruction
  Repetition), the one intentional exception to CE-6's de-dup rule. Restate, do not relocate.
  Remediation: [learn — top-and-tail](https://learn.agentpatterns.ai/prompt-engineering/top-and-tail/).
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

## CE-2 — Hard guardrails in the dead middle
- **Flags:** *never / always / must / do not* rules whose violation is expensive, located in
  the middle zone.
- **Why:** identical instructions are followed less reliably in the middle than at the edges
  (Liu et al. 2023;
  [corpus — lost-in-the-middle](https://agentpatterns.ai/context-engineering/lost-in-the-middle/)).
  A guardrail in the dead middle is technically present but practically weak.
- **Fix:** surface each critical guardrail at the primacy edge, and/or restate it in the
  CE-1 closing block. Keep the canonical statement once; the edge copy is a pointer/restatement.
  Remediation: [learn — top-and-tail](https://learn.agentpatterns.ai/prompt-engineering/top-and-tail/).

## CE-3 — Oversized file / large dead middle
- **Flags:** an always-loaded file that is long for its job — a large low-attention middle (e.g.
  topic- or path-scoped bulk loaded on every turn regardless of task) — or growing unbounded.
  There is no sourced fixed line quota: judge from step 2's **measurements** — the file's token
  count, its share of the assembled always-loaded load, and the size of its dead middle — not a
  bare line count. The longer the file, the larger its low-attention middle.
- **Why:** output quality degrades as context fills — "context rot," a gradient not a cliff,
  with onset nearer an absolute token threshold (roughly 32K–100K *assembled* tokens, varying by
  task type — reasoning-heavy tasks degrade earliest) than a fixed % (Anthropic, *Effective Context
  Engineering*; RULER, arXiv:2404.06654;
  [corpus — context-window-dumb-zone](https://agentpatterns.ai/context-engineering/context-window-dumb-zone/)).
  Every always-loaded line spends that budget before any task token and buries its neighbours.
- **Fix:** move path/topic-scoped guidance into files that load **on demand** (e.g. nested
  per-directory files, path-scoped rule files, or linked references) so the always-loaded core
  stays small. Cut any line whose removal would not change agent behavior.
  Remediation: [learn — the-dumb-zone](https://learn.agentpatterns.ai/context-engineering/the-dumb-zone/).

## CE-4 — Discoverable content (bloat)
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
  (Gloaguen et al., *Evaluating AGENTS.md*, 2026, arXiv:2602.11988;
  [corpus — evaluating-agents-md-context-files](https://agentpatterns.ai/instructions/evaluating-agents-md-context-files/)).
  Only **non-discoverable** context
  earns space: rationale ("why X over Y"), constraints/gotchas not encoded in code, domain rules,
  conventions present-but-unstated, out-of-band integrations.
- **Fix:** delete discoverable lines. Where *direction* helps, replace the copy with a **pointer**
  — a path, not a duplicate ("use the repository pattern in `src/repos/`"). Litmus for each kept
  line: *would it still be true and useful if the codebase changed next week?*
  Remediation: [learn — discoverable-or-not](https://learn.agentpatterns.ai/context-engineering/discoverable-or-not/).
- **Caveat:** if the target agent lacks exploration tools, structure is non-discoverable *to it* —
  audit actual tool access before flagging. In high-churn repos, keep structural pointers in a
  separate frequently-updated file, not the main always-loaded one.

## CE-5 — Contradiction / unresolved layering
- **Flags:** two rules that conflict — within one file, or across the layer stack
  (system prompt → project `AGENTS.md`/`CLAUDE.md` → skill `SKILL.md` → user message) — or a rule
  silently superseded by a later one, with no precedence stated.
- **Why:** instructions arrive from multiple layers at once. Precedence follows **specificity**
  (closer-to-task wins) but is a behavioral tendency, not an enforced rule, so unresolved conflicts
  make behavior unpredictable and position-dependent
  ([corpus — layered-instruction-scopes](https://agentpatterns.ai/instructions/layered-instruction-scopes/)).
- **Fix:** state the precedence explicitly, or remove the loser. Keep one source of truth per rule.
  Remediation: [learn — most-specific-wins](https://learn.agentpatterns.ai/prompt-engineering/most-specific-wins/).
- **Cross-document scope:** semantic conflicts *across multiple files* in a skill library are
  `conflicting-instruction-detector`'s lane (flag-only); CE-5 owns within-file and layer-stack.

## CE-6 — Layer hygiene
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
  sub-agents
  ([corpus — instruction-file-ecosystem](https://agentpatterns.ai/instructions/instruction-file-ecosystem/)).
- **Fix:** one rule, one layer — others point or stay silent. Repair or remove dead imports. Move
  task-specific rules down to the skill/path-scoped layer. Pass critical constraints into
  sub-agents explicitly.
  Remediation: [learn — the-compliance-stack](https://learn.agentpatterns.ai/prompt-engineering/the-compliance-stack/).

## CE-7 — Composition corrupts the assembled curve
- **Flags:** the `@`-import arrangement degrades the attention curve of the *assembled* file
  (audit the resolved assembly from the procedure, not raw files):
  - **Displacement** — a bulky reference import near the top pushes the host file's hard rules out
    of primacy and into the dead middle; or a large import at the end buries the real tail guardrail.
  - **Duplicated bulk** — the same imported content pulled into more than one place, wasting edge
    real estate and inflating the middle (distinct from CE-6's *rule*-level duplication).
- **Why:** Claude Code expands `@path` imports verbatim at load time, so the U-shaped curve applies
  to the concatenated whole. Arrangement *is* positioning: where you place an import decides what
  occupies primacy/recency and what falls into the low-attention middle
  ([corpus — import-composition-pattern](https://agentpatterns.ai/instructions/import-composition-pattern/)).
- **Fix (in order):** (1) cut discoverable/duplicated bulk first (CE-3/CE-4/CE-6); (2) **re-order
  imports** so reference material sits in the middle and rule-bearing files at the edges; (3) only
  then restate (CE-1). Output the proposed assembled order as a recommendation; apply the re-order
  only on confirmation (per *Stance*) — never infer intent and rewrite.
  Remediation: [learn — assembling-the-prompt](https://learn.agentpatterns.ai/context-engineering/assembling-the-prompt/).

## CE-8 — Compliance ceiling
- **Flags:** the assembled always-loaded context carries **too many rules** — high total normative
  load across all files + imports (rough trigger: many tens of `never/always/must` statements), even
  if each individual file is short.
- **Why:** aggregate instruction load — not file count — drives degradation; past the compliance
  ceiling the model silently drops lower-priority rules (usually the project layer). Ten rules in a
  flat prompt outperform forty spread across layers. A short file can still exceed the ceiling
  (IFScale 2025, arXiv:2507.11538;
  [corpus — instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/)).
- **Fix:** cut rules the agent doesn't need every task (move task-specific ones to skills /
  path-scoped rules that load on demand), merge overlapping rules, and reserve edge-repetition
  (CE-1) for the *few* genuinely critical constraints so the priority signal survives.
  Remediation: [learn — the-ceiling](https://learn.agentpatterns.ai/prompt-engineering/the-ceiling/).

## CE-9 — Cache-busting volatile content in the prefix
- **Flags:** an always-loaded instruction file (part of the cached static prefix) that embeds
  **volatile, per-session content** — timestamps / dates ("as of <date>", "last updated …"),
  absolute machine paths, `cwd`, hostnames, usernames, session/run IDs, env-specific values, or
  any "current state" that changes between runs or machines.
- **Why:** prompt caching reuses an exact byte-identical prefix; any volatile token busts it, so
  the whole prefix re-bills at full price **every turn**, silently — a miss never errors, it just
  bills. The economics are provider-specific: on Anthropic-priced caching, reads are ~10% of base
  and a busted prefix incurs a cache **write** (125–200%); on other providers it forfeits the
  cached-input discount rather than surcharging. Either way volatile-in-prefix pays full price —
  and loses on attention too: it's noise that dilutes the real rules.
  (Static prose that merely *describes* a path is fine — flag values that actually change per run.)
  ([corpus — static-content-first-caching](https://agentpatterns.ai/context-engineering/static-content-first-caching/))
- **Fix:** move volatile/state content to the dynamic tail or compute it at runtime; keep the
  always-loaded file static and byte-stable across sessions and machines.
  Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

## CE-10 — Low semantic density (ceremony / filler)
- **Flags:** lines that carry little guidance per token — hedging and filler ("it is important that
  you ensure…", "please try to…"), prose where a table/bullet/rule would be denser, explanations
  with no compliance value, restated standard conventions the model already knows.
- **Why:** verbose instructions don't improve accuracy — they bury the real rules (Anthropic:
  bloated files get ignored). But "shorter" is **not** monotonically better: constraint violations
  peak at *medium* compression (arXiv:2512.17920), and stripping *meaning* (not ceremony) shifts the
  burden to reasoning (Ustynov 2026 — a 17% input-token cut raised session cost 67%;
  [corpus — semantic-density-optimization](https://agentpatterns.ai/context-engineering/semantic-density-optimization/)).
- **Fix:** apply the compression test (cut any word/sentence that changes no behavior); convert prose
  to tables / bullets / rules; compress **decisively** to a crisp, unambiguous rule — never leave it
  half-trimmed (the U-curve trough). Keep rationale only where compliance on unforeseen inputs needs
  it. If content doesn't compress but isn't needed every task, move it to an on-demand skill
  (structural compression — complements CE-3). To *apply* the fix, hand the flagged section to the
  **`compress-prompt`** skill — this detector pairs with that transform.
  Remediation: [learn — signal-per-token](https://learn.agentpatterns.ai/context-engineering/signal-per-token/).
- **Consolidate repeats:** when the same low-density root cause (e.g. hedging/ceremony phrasing)
  recurs across several paragraphs of one file, that is one systemic finding, not several — emit a
  **single** finding row citing every location as evidence (e.g. `L3–6, L8–10, L12–14, L16–18,
  L20–21`), not one row per occurrence.

---

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
- **Always-loaded files are the cached prefix** — static, byte-stable content is what the
  provider's cache discount applies to; volatile content in the prefix busts the cache and pays
  full price every turn, silently (the exact economics are provider-specific — see CE-9).
- **Compress to crisp, not merely short** — density raises compliance, but violations peak at
  *medium* compression and stripping meaning shifts cost to reasoning; compress decisively or stay verbose.
- **Attention-edge placement and static-first caching are compatible, not opposed** — a static
  instruction block with rules at its own edges satisfies both. Only volatile-in-prefix (CE-9) loses
  on both axes; a tail restatement (CE-1) is cache-safe but worth its tokens only when length and
  criticality justify it — which is why CE-1 is conditional, not a default.

## Expanding this skill

This catalog grows as new context-engineering principles are learned. To add a check:

1. Append a `## CE-N — <name>` block with **Flags / Why / Fix**, one principle per check, and
   cite a primary source in *Why*.
2. If the principle is a *structural* property (position, length, density), make the check
   detectable from file structure alone so it stays project-agnostic.
3. Add a row example to the output template (in `SKILL.md`) only if it introduces a new severity rationale.
4. Note any contract change (new/changed check IDs or output shape) for the release notes — the shelf
   version is set uniformly at release, not bumped per check.

Keep every check independent: a reader should be able to run one check without the others.
