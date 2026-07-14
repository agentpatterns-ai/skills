# audit-geo-content — detectors

Loaded on demand by `audit-geo-content`. **Run GEO-G1 first** — if it fires, suppress all others.
Each item: ID / **Flags** / **Why** (cited) / **Fix** (→ remediation lesson). Detection is
read-only. Several checks are *deterministic* (heading grep, word-count, page-floor, llms.txt
existence) — prefer them over judgement.

## Contents
- [Applicability guard — run first (GEO-G1)](#applicability-guard--run-first)
- [Structure — chunkability (GEO-S1…GEO-S5)](#structure--chunkability-per-page)
- [Substance — assertion density (GEO-D1…GEO-D3)](#substance--assertion-density-per-major-h2)
- [Machine-readability (GEO-M1…GEO-M3)](#machine-readability-once-per-corpus)

---

## Applicability guard — run first

### GEO-G1 — Applicability / no gaming
- **Flags (suppress checks below if any holds):** the corpus is private/intranet and not indexed by
  AI crawlers; a frequently-versioned API reference where per-release maintenance outweighs citation
  gain; or a tightly-coupled procedural sequence that would break if atomised. **Also flag (never
  recommend):** fabricating statistics, adding hedge tags, or keyword padding to raise density.
- **Why:** GEO optimises chunk-based RAG retrieval — no benefit when content is consumed whole or
  never indexed, and atomising coupled procedures strips the setup context the answer needs.
  Manufactured signal is detectable and the benchmark's PAWC metric rewards length even for
  fabricated stats — directional finding robust, exact % an upper bound ([assertion-density,
  *Caveats / Unsourceable Claims*](https://agentpatterns.ai/geo/assertion-density/);
  [geo-for-technical-docs, *When This Backfires*](https://agentpatterns.ai/geo/geo-for-technical-docs/);
  [answer-first-writing, *When This Backfires*](https://agentpatterns.ai/geo/answer-first-writing/)).
- **Fix:** report "GEO does not apply — suppressed" with the reason; never propose fabricating or
  padding. *(→ [capstone-measure-and-decide](https://learn.agentpatterns.ai/geo/capstone-measure-and-decide/) —
  closes the course with "When GEO isn't worth it", the skip-GEO judgment this guard operationalizes;
  the guard's specific suppress criteria trace to the corpus pages cited above.)*

---

## Structure — chunkability (per page)

### GEO-S1 — Buried answer (not answer-first)
- **Flags:** an H2 that opens with preamble/throat-clearing ("In this article we'll explore…",
  "Let's look at…") or a metaphor/dramatic lead instead of a direct **40–60-word** answer.
- **Why:** the chunk's dominant signal is its opening text; preamble makes the embedding a diluted
  average of throat-clearing and answer. Anthropic's Contextual Retrieval cut retrieval failure
  **49%** (contextual embeddings + contextual BM25 combined; embeddings alone ≈35%) by **prepending**
  context to each chunk — same lever: the chunk's opening signal decides retrieval ([answer-first-writing](https://agentpatterns.ai/geo/answer-first-writing/);
  [Anthropic](https://www.anthropic.com/news/contextual-retrieval)).
- **Fix:** lead each H2 with a self-contained 40–60-word answer; move context/caveats after it.
  *(→ [answer-first-atomic-pages](https://learn.agentpatterns.ai/geo/answer-first-atomic-pages/))*

### GEO-S2 — Generic / low-signal heading
- **Flags (deterministic):** bare headings `^#{1,3} (Overview|Background|Details|Summary|Introduction|Conclusion|Notes)$`.
- **Why:** generic headings carry zero discriminative semantic load and make weak deep-link anchors;
  descriptive headings are retrieval anchors even at chunk boundaries ([answer-first-writing](https://agentpatterns.ai/geo/answer-first-writing/);
  [atomic-pages-and-chunking](https://agentpatterns.ai/geo/atomic-pages-and-chunking/)).
- **Fix:** rename to a concept-naming heading ("How RAG Scores Section Openings", not "Overview").
  *(→ [answer-first-atomic-pages](https://learn.agentpatterns.ai/geo/answer-first-atomic-pages/))*

### GEO-S3 — Section / page length out of band
- **Flags:** an H2 section **>400 words** (~512+ tokens — fully deterministic, any H2); a
  **substantive prose** H2 section **<200 words**; or a whole page **under ~200 words** (below the
  retrieval floor). *Scope guard (short side only):* do **not** flag legitimately-short structural
  sections — prerequisites, code-only steps, changelog stubs, link hubs — the <200 test applies only
  to sections that carry a conceptual claim (the same "substantive H2" qualifier as GEO-D2).
- **Why:** 256–512-token chunks retrieve best; too-short = thin context the LLM can't answer from,
  too-long = a diluted embedding ([atomic-pages-and-chunking](https://agentpatterns.ai/geo/atomic-pages-and-chunking/),
  NVIDIA 2024 chunking benchmark; [GEO paper](https://arxiv.org/abs/2311.09735)).
- **Fix:** split or merge to land sections in 200–400 words; no page below the ~200-word floor.
  *(→ [answer-first-atomic-pages](https://learn.agentpatterns.ai/geo/answer-first-atomic-pages/))*

### GEO-S4 — Non-atomic (multi-concept) page
- **Flags:** one page whose H1/H2s describe distinct independent mechanisms (a "kitchen-sink" page —
  e.g. install + auth + billing + SDK on one page).
- **Why:** a multi-topic page produces a blended-average embedding less similar to any single query;
  one concept per page maps cleanly to one chunk ([atomic-pages-and-chunking](https://agentpatterns.ai/geo/atomic-pages-and-chunking/)).
  *Guard:* one *meaningful* concept per page — co-cited contrasts (authn vs authz) stay together.
- **Fix:** split into one-concept pages. *(→ [answer-first-atomic-pages](https://learn.agentpatterns.ai/geo/answer-first-atomic-pages/))*

### GEO-S5 — Section not self-contained
- **Flags:** forward references ("as we will see below"), reliance on the H1 title for context, or a
  concept named only in the heading (not in the opening sentence).
- **Why:** RAG retrieves by section and may present one H2 block as the whole answer — the chunk may
  not carry the title or neighbours ([answer-first-writing, *Section Independence*](https://agentpatterns.ai/geo/answer-first-writing/)).
- **Fix:** define terms in-section; name the concept in the opening sentence; drop forward refs.
  *(→ [answer-first-atomic-pages](https://learn.agentpatterns.ai/geo/answer-first-atomic-pages/))*

---

## Substance — assertion density (per major H2)

### GEO-D1 — Low assertion density (waffle over claims)
- **Flags (heuristic grep):** vague quantifiers (`many / often / most / significantly`),
  unattributed authority (`experts say / research shows / it is widely known`), anchorless
  comparatives (`much faster / far more accurate`), undated generalisations (`historically`).
- **Why:** specificity drives token-level match and attribution confidence — the Princeton GEO study
  measured Quotation +41% / Statistics +30% / Cite Sources +30% in source visibility ([assertion-density](https://agentpatterns.ai/geo/assertion-density/);
  [Aggarwal et al., KDD 2024](https://arxiv.org/abs/2311.09735)).
- **Fix:** replace with numbers+units, named/credentialed sources, dates, sample sizes.
  *(→ [assertion-density](https://learn.agentpatterns.ai/geo/assertion-density/))*

### GEO-D2 — No specific stat + source per major section
- **Flags:** a substantive H2 with **neither** a specific statistic (number with units) **nor** a
  markdown link to its source.
- **Why:** statistics and cited sources are the highest-impact single rewrites for citation
  ([assertion-density](https://agentpatterns.ai/geo/assertion-density/)). *Caveat:* PAWC rewards
  length — directional, not a license to pad (see GEO-G1).
- **Fix:** add ≥1 sourced statistic with units; if none is sourceable, weaken the claim — don't
  invent one. *(→ [assertion-density](https://learn.agentpatterns.ai/geo/assertion-density/))*

### GEO-D3 — Keyword stuffing
- **Flags:** repeated keyword padding (a phrase or term repeated for density, not meaning).
- **Why:** a transfer-negative from classic SEO — keyword stuffing **scored −10%** in the Princeton
  benchmark; keyword density shows negligible/negative effect on generative visibility ([assertion-density](https://agentpatterns.ai/geo/assertion-density/);
  [atomic-pages-and-chunking](https://agentpatterns.ai/geo/atomic-pages-and-chunking/)).
- **Fix:** remove padding; spend tokens on specific, sourced claims instead.
  *(→ [assertion-density](https://learn.agentpatterns.ai/geo/assertion-density/))*

---

## Machine-readability (once per corpus)

### GEO-M1 — Missing / stale machine-readable corpus
- **Flags (deterministic + judgement):** no `/llms.txt` or `/llms-full.txt` at site root; malformed
  structure (the spec requires exactly one H1); or **stale entries / dead links**.
- **Why:** `llms.txt` is a curated inference-time navigation index — and **stale entries pointing to
  dead links are worse than no file**. State it honestly: it is **not** a citation or ranking signal
  (no major provider has confirmed reading it) ([llms-txt](https://agentpatterns.ai/geo/llms-txt/);
  [llmstxt.org](https://llmstxt.org)).
- **Fix:** publish/refresh a curated `llms.txt` (+ `llms-full.txt`); automate freshness; do **not**
  recommend it as a ranking lever. *(→ [machine-readable-corpora](https://learn.agentpatterns.ai/geo/machine-readable-corpora/))*

### GEO-M2 — Schema fidelity / type mismatch
- **Flags (where JSON-LD is emitted):** FAQPage answers outside **40–80 words**; body text drifted
  from the markup (contradictory signals); wrong type for shape (e.g. HowTo on prose, FAQPage on
  unrelated Q&A).
- **Why:** schema is the one machine-readable file with *measured* citation lift (FAQPage
  **2.7×–3.2×**), but it accrues at **indexing** time, not live fetch — and stale/mismatched markup
  gets a page *deprioritised* ([schema-and-structured-data](https://agentpatterns.ai/geo/schema-and-structured-data/);
  [Frase.io](https://www.frase.io/blog/faq-schema-ai-search-geo-aeo)).
- **Fix:** keep body in sync with markup; match type to content shape; FAQ answers 40–80 words,
  standalone. *(→ [machine-readable-corpora](https://learn.agentpatterns.ai/geo/machine-readable-corpora/))*

### GEO-M3 — Crawler policy blocks retrieval bots (robots.txt)
- **Flags (deterministic):** `/robots.txt` issues a `Disallow: /` that reaches a Tier-1 **retrieval**
  user-agent — `OAI-SearchBot`, `Claude-SearchBot`, `Claude-User`, `PerplexityBot`, `Perplexity-User`,
  `ChatGPT-User` — including a blanket `User-agent: *` + `Disallow: /` with **no** retrieval-bot allow
  override. Also flag: a non-compliant bot (`Bytespider`) "blocked" only in robots.txt with no WAF
  rule. **Advisory sub-flag (not a violation):** training scrapers (`GPTBot`, `ClaudeBot`,
  `Google-Extended`, `Meta-ExternalAgent`) left **allowed** — note it against the three-tier policy,
  but allowing training crawl is a legitimate publisher choice, orthogonal to citability.
- **Why:** a blocked retrieval bot never fetches the page, so it can never be cited — this is a
  **live-fetch** block (not indexing-time hygiene like M1/M2), and **fatal to citability**. The
  recommended policy is three-tier: **allow** retrieval / **disallow** training (advisory) / **WAF-block**
  non-compliant. robots.txt is **advisory, not enforceable** — `ChatGPT-User` was exempted (Dec 2025)
  and Perplexity rotates user-agents/ASNs to evade it (bot names and exemption status are provider
  policy and change — verify the current list against the cited ai-crawler-policy source before
  relying on this enumeration) — so any *hard* block needs a CDN/WAF rule, not
  a robots.txt line ([ai-crawler-policy](https://agentpatterns.ai/geo/ai-crawler-policy/);
  [Cloudflare, Aug 2025](https://blog.cloudflare.com/perplexity-is-using-stealth-undeclared-crawlers-to-evade-website-no-crawl-directives/)).
- **Fix:** allow Tier-1 retrieval bots; enforce non-compliant blocks at CDN/WAF — never rely on
  robots.txt for hard enforcement. Advise (don't require) disallowing Tier-2 training scrapers —
  the publisher's call. **Severity High** (a blocked
  retrieval bot loses every citation for the page, at fetch time — not the Low machine-readable
  hygiene band). *(→ [machine-readable-corpora](https://learn.agentpatterns.ai/geo/machine-readable-corpora/))*
