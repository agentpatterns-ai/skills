---
name: audit-geo-content
description: Audit a documentation / content-markdown corpus for AI answer-engine retrieval and citability (GEO) — chunkability (answer-first, atomic, descriptive headings), assertion density, and machine-readable signals (llms.txt, schema). Invoke when reviewing or hardening published docs for AI citation / generative-engine visibility ("is this page citable by ChatGPT/Perplexity/Claude?", a GEO review, or before/after a docs restructure aimed at retrieval). Skip when auditing agent-harness config — tool allowlists, MCP, sandbox/egress (use audit-lethal-trifecta) — or instruction-file prose/attention such as a SKILL.md or CLAUDE.md (use audit-instruction-file); and suppress for private / non-indexed docs.
user-invocable: true
version: "0.2.1"
usage: /audit-geo-content [path-to-docs-corpus]
---

# Audit GEO Content

Audits **published content markdown** (docs tree, knowledge base, docs-site source) for
retrieval/citability by AI answer engines (RAG / generative search). Engines retrieve and cite the
most relevant **chunk** (~256–512 tokens), not the page — so structure, not prose polish, decides
whether a passage surfaces with enough self-contained signal to be cited
([atomic-pages-and-chunking](https://agentpatterns.ai/geo/atomic-pages-and-chunking/), NVIDIA 2024
chunking benchmark: 256–512-token chunks retrieve best for factoid queries). Also target site-root
machine-readable files: `llms.txt`, `llms-full.txt`, any JSON-LD/schema emission.

**Stance — detect and recommend; read-only; apply nothing without confirmation; never reward
gaming.** The two costliest guardrails go first:
- **Applicability guard (check first; suppress if it fires).** Do **not** flag content AI crawlers
  don't index — private/intranet docs, frequently-versioned API refs where maintenance cost
  outweighs gain, or tightly-coupled procedural sequences that break if atomised. GEO optimises
  chunk-based RAG retrieval; it adds nothing when content is consumed whole or never
  indexed ([answer-first-writing, *When This Backfires*](https://agentpatterns.ai/geo/answer-first-writing/);
  [geo-for-technical-docs, *When This Backfires*](https://agentpatterns.ai/geo/geo-for-technical-docs/)).
- **Never recommend fabricating signal.** Do not suggest inventing statistics, hedge tags, or
  keyword padding to raise density — manufactured stats are detectable, and keyword stuffing
  *scored −10%* in the Princeton benchmark. Weaken or remove an unsourceable claim; do not dress it
  up ([assertion-density, *Unsourceable Claims*](https://agentpatterns.ai/geo/assertion-density/)).
- **Measured-lift caveat.** The benchmark's PAWC metric rewards length and even permitted fabricated
  stats — directional finding (specific > vague) is robust; exact percentages are an upper bound.
  `llms.txt` is a navigation aid, **not** a citation/ranking signal.

## Input
- `path` (optional): docs corpus / docs-site source. Default: scan markdown under `docs/` (or given
  root) plus site-root `llms.txt` / `llms-full.txt` / `robots.txt` and any emitted JSON-LD. Read-only.

## Scope
Audits **published content markdown for retrieval/citability**: per-page chunkability, assertion
density, machine-readable corpus files. **Out of scope** (hand off):
- agent-harness config — tool allowlists, MCP, sandbox/egress → `audit-lethal-trifecta`
- instruction-file prose / attention / density (SKILL.md, CLAUDE.md) → `audit-instruction-file`
- human-first / private / non-indexed docs → suppressed by the applicability guard (GEO-G1)

## Procedure
1. **Applicability guard (GEO-G1) first.** If the corpus is private/non-indexed, version-churned, or
   tightly procedural, report "GEO does not apply here — suppressed" with the reason and stop. Do not
   over-claim.
   Done when a verdict is recorded: the suppression report issued with its reason (stop) or
   applicability confirmed (proceed).
2. **Per page — Structure checks** (GEO-S1…S5): answer-first openers, descriptive headings,
   section/page length band, one-concept atomicity, section self-containment.
   Done when every page × S-check has a recorded verdict — no blanks.
3. **Per major H2 — Substance checks** (GEO-D1…D3): assertion density, ≥1 stat+source, keyword stuffing.
   Done when every major H2 × D-check has a recorded verdict — no blanks.
4. **Once per corpus — Machine-readability checks** (GEO-M1…M3): `llms.txt`/`llms-full.txt` presence
   + freshness; schema fidelity/type-for-shape where JSON-LD is emitted; `robots.txt` crawler policy
   (a `Disallow` on a retrieval bot is fatal to citability — GEO-M3).
   Done when all three M-checks carry a recorded corpus-level verdict.
5. **Report** with the template below — per-finding `Check / Where / Flag / Fix`, citing the page and
   linking the remediation lesson. Recommend; apply nothing without confirmation.
   Done when the Findings table, Clean/passing, and Highest-impact fix sections are all filled.

Per-check detectors (Flags / Why+source / Fix, grouped Structure / Substance / Machine-readability /
Guard) live in [`checks.md`](checks.md) — load when auditing. Several are deterministic (heading grep,
word-count, page-floor, llms.txt existence); prefer them over judgement.

## Output template
```
# GEO content audit — <corpus>

Applicability (GEO-G1): APPLIES — public, indexed docs corpus.   (else: SUPPRESSED — <reason>; stop.)

## Findings
| Check | Sev | Where | Flag | Fix (→ lesson) |
|---|---|---|---|---|
| GEO-S1 buried answer | High | guide/setup.md#configuration | Opens "In this article we'll explore…" — preamble averages the chunk embedding | Lead with a 40–60-word direct answer (answer-first-atomic-pages) |
| GEO-S2 generic heading | Med | guide/setup.md#overview | `## Overview` — zero discriminative signal, weak deep-link anchor | Rename to a concept-naming heading (answer-first-atomic-pages) |
| GEO-S3 length out of band | Med | guide/setup.md | 2,040-word page, no H2 splits — diluted blended embedding | Split into 200–400-word atomic sections (answer-first-atomic-pages) |
| GEO-S4 non-atomic | High | guide/setup.md | One page covers install + auth + billing + SDK — five mechanisms | One concept per page (answer-first-atomic-pages) |
| GEO-D1 low assertion density | Med | guide/setup.md#auth | "experts say this is significantly faster" — unattributed + anchorless | Numbers+units, named/dated source (assertion-density) |
| GEO-D2 no stat+source | Med | guide/setup.md#billing | No statistic and no source link in a substantive H2 | Add ≥1 sourced stat with units (assertion-density) |
| GEO-M1 missing llms.txt | Low | <site root> | No `/llms.txt` or `/llms-full.txt` | Publish curated index; note: navigation aid, NOT a ranking signal (machine-readable-corpora) |
| GEO-M3 retrieval bot blocked | High | /robots.txt | `User-agent: * / Disallow: /` with no retrieval-bot allow — live-fetch block, fatal to citability | Allow Tier-1 retrieval bots; WAF-block non-compliant, not robots.txt (machine-readable-corpora) |

**Clean / passing:** <pages that pass every check, and why.>
**Highest-impact fix:** <the one change that most raises citability — usually answer-first + atomicity.>
**Do NOT:** fabricate stats, hedge-tag, or keyword-stuff to raise density.
```
Severity by retrieval impact: **High** = structural miss that loses the chunk entirely (buried
answer, non-atomic page) **and GEO-M3** (a blocked retrieval bot is a live-fetch block — fatal to
citability); **Medium** = substance gaps (low density, missing stat/source) and weaker structural
signal (generic heading, length out of band); **Low** = **GEO-M1/M2 only** — machine-readable
hygiene (llms.txt/schema), accruing at indexing time, not live fetch.

## Related / pairing
A **content-target** audit — the factory's first. Siblings audit *agent files*
(`audit-lethal-trifecta`) and *instruction prose* (`audit-instruction-file`): same detect→cite→
recommend shape, different artifact and different "good" (citable by an engine, not safe/loadable by
an agent). Corpus + per-check primary sources (Aggarwal et al. GEO KDD 2024
[arXiv:2311.09735](https://arxiv.org/abs/2311.09735), Anthropic Contextual Retrieval, NVIDIA 2024,
llmstxt.org, Frase.io) cited inline per check in [`checks.md`](checks.md).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- **Run GEO-G1 first** — if docs aren't AI-indexed (private/versioned/procedural), GEO doesn't apply;
  suppress rather than over-claim.
- **Never recommend fabricating** statistics, hedge tags, or keyword padding — manufactured signal
  is detectable; keyword stuffing measured −10%. Weaken or remove unsourceable claims instead.
- **Optimise the chunk, not the page** — answer-first + atomic is the highest-impact lever; llms.txt
  is navigation, not a ranking signal. Read-only — recommend; apply nothing without confirmation.
