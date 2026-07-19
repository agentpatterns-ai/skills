# audit-memory-retrieval-integrity — detectors

Loaded on demand by `audit-memory-retrieval-integrity`. Run every detector against each retrieval /
memory call site. Each item: ID / **Flags** / **Why** (cited) / **Fix** (→ lesson for remediation
walk-through). All detection is read-only. Two families: **authorization** (MR-A*) and **integrity**
(MR-I*).

## Contents
- [Authorization family (MR-A1…MR-A5)](#authorization-family--who-may-see-this-chunk)
- [Integrity family (MR-I1…MR-I3)](#integrity-family--is-this-chunk-trustable)
- [Detector → family map](#detector--family-map-for-the-report)

---

## Authorization family — who may see this chunk

### MR-A1 — Unscoped retrieval (no tenant pre-filter)
- **Flags:** a `search()` / retriever injects top-K into context with **no** tenant/metadata predicate
  at query, post-retrieval, or index level (ANN-first, ACL-never), while a multi-tenant surface exists
  (`tenant_id` / `workspace` / `org` referenced anywhere).
- **Why:** retrieval ranks by relevance, not authorization; on a shared index the top chunk for one
  tenant can belong to another — ungated retrieval leaked cross-tenant data in 98–100% of probes
  ([gap](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
- **Fix:** pass a tenant/ABAC filter **into** the search call (pre-filter the search space), tag every
  chunk with `tenant_id` + ABAC attributes at ingest. **High** when all three signals are zero on a
  multi-tenant surface. Remediation: [learn — the-chunk-that-wasnt-yours](https://learn.agentpatterns.ai/security/the-chunk-that-wasnt-yours/).

### MR-A2 — Post-only authorization (no two-tier filter)
- **Flags:** authorization runs **only** on the returned list in app code, against the full shared
  index — a pre-filter into the search call is missing (or vice-versa: only a pre-filter, no post-ACL).
- **Why:** a pre-filter alone is bypassed by ANN paths that traverse outside the metadata filter; a
  post-filter alone collapses recall because the top-K budget was spent on forbidden chunks. Both
  layers are required ([gap, *Why two-tier filtering*](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
- **Fix:** add the missing layer — pre-filter into the index **and** post-ACL on the top-K for
  defence-in-depth. **Partial leg** → mark Medium, don't zero. Remediation:
  [learn — the-chunk-that-wasnt-yours](https://learn.agentpatterns.ai/security/the-chunk-that-wasnt-yours/).

### MR-A3 — Agent-supplied tenant identity
- **Flags:** the `tenant_id` used for retrieval originates from LLM call args, parsed prompt text, or
  other agent-rewritable input rather than a server-side session principal, signed JWT claim, or
  orchestrator-set header.
- **Why:** an LLM-supplied identity is attacker-steerable — the model can be injected into asking for
  another tenant's scope, defeating the filter ([gap, *Don't*](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
- **Fix:** bind the tenant id from the server-side principal / signed claim; never let the agent name
  its own scope. **High.** Remediation:
  [learn — the-chunk-that-wasnt-yours](https://learn.agentpatterns.ai/security/the-chunk-that-wasnt-yours/).

### MR-A4 — Session-scoped state / unset multi-tenant isolation knobs
- **Flags:** multi-turn or memory state keyed by **session** not tenant identity (so a token refresh /
  impersonation inherits the prior tenant's context); **or** a shared-container Agent-SDK host calling
  `query()` without `settingSources: []`, `CLAUDE_CONFIG_DIR`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`,
  per-tenant `cwd`, and per-tenant egress.
- **Why:** by default the SDK loads filesystem settings (whenever `settingSources` is omitted —
  `settingSources: []` blocks them), while global config and auto-memory are read **regardless of
  `settingSources`** and need their own knobs (`CLAUDE_CONFIG_DIR`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY`) —
  this is the SDK's current default behavior and may change across SDK versions; verify against the
  installed SDK version rather than trusting this description to stay accurate. Each pathway is a
  cross-tenant leak in a shared container, and session-keyed state carries
  one tenant's data forward across an identity switch
  ([isolation-knobs](https://agentpatterns.ai/security/multi-tenant-isolation-knobs-agent-sdk/);
  [gap, *Context accumulation*](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
- **Fix:** scope state by tenant id; set all four SDK knobs + per-tenant egress on every `query()`.
  Remediation: [learn — the-chunk-that-wasnt-yours](https://learn.agentpatterns.ai/security/the-chunk-that-wasnt-yours/).

### MR-A5 — PII untokenized at the retrieval boundary
- **Flags:** records with email / name / account / SSN / DOB fields are placed into prompt or
  retrievable context with **no** interstitial tokenization/redaction step; or redaction relies on a
  prompt instruction; or detection is regex-only with no NER/schema step.
- **Why:** any value in context may be logged, cached, or observed by inference infra; the boundary
  must be enforced by deterministic code, not model judgment
  ([pii-tokenization](https://agentpatterns.ai/security/pii-tokenization-in-agent-context/)).
- **Fix:** deterministic tokenizer in the execution environment, type-prefixed tokens (`{{EMAIL_N}}`),
  session-scoped map with expiry, de-tokenization audit log. **High** for raw PII; **Medium** for
  regex-only detection (misses quasi-identifiers). Remediation:
  [learn — the-lethal-trifecta](https://learn.agentpatterns.ai/security/the-lethal-trifecta/) (treats
  PII tokenization as trifecta leg-removal — the deepest coverage available; no dedicated PII
  tokenization lesson exists yet).

---

## Integrity family — is this chunk trustable

### MR-I1 — Retrieved chunks trusted as instructions / no provenance tier (RAG poisoning)
- **Flags:** retrieved top-K passages are wired into context with no provenance/trust tier and no
  generation-stage defense; high-stakes queries do not default to a human-curated tier; an attacker can
  write to the corpus (web ingestion, user-submitted docs, scraped feeds).
- **Why:** under knowledge-base poisoning, attack-success spans 24.4–81.9% across architectures at
  comparable clean accuracy; for 3 of 4 the failure is at *generation*, and agentic loops *amplify*
  meta-epistemic "this is the authoritative source" framing rather than filtering it
  ([rag-poisoning-robustness](https://agentpatterns.ai/security/rag-architecture-poisoning-robustness/)).
- **Fix:** tag each chunk with a source tier at ingest, expose a provenance-tier filter on retrieval,
  default high-stakes workflows to the curated tier, add generation-stage defensive prompting, render
  source alongside each fact. **High** with a writable corpus, else **Medium**. Remediation:
  [learn — the-provenance-blind-model](https://learn.agentpatterns.ai/security/the-provenance-blind-model/).

### MR-I2 — Writable oracle / KG queried via tool-use (oracle poisoning)
- **Flags:** the agent queries a knowledge graph / structured oracle via tool-use (MCP / SDK tool
  call) that has a **writable path** from third-party ingestion or a shared write API, with directed
  single-source queries and no read-only access control.
- **Why:** corrupting a tool-queried KG produces 100% trust at moderate (L2) attacker sophistication
  across nine models — the data path, not the instruction path, carries the payload, and tool-delivered
  facts are trusted as ground truth (0% inline vs 100% tool-use on the same fact)
  ([oracle-poisoning](https://agentpatterns.ai/security/oracle-poisoning-knowledge-graph/)).
- **Fix:** enforce **read-only** access control on the KG (the only fully-effective defense); treat any
  third-party-writable graph as an untrusted-input surface; prefer open-ended multi-source queries;
  add provenance signatures (partial). **High** when a writable path exists. Remediation:
  [learn — the-payload-that-waits](https://learn.agentpatterns.ai/security/the-payload-that-waits/).

### MR-I3 — Memory store auto-ingests untrusted writes (memory poisoning)
- **Flags:** an agent-memory store (sliding-window, RAG-memory, memory tool, agentic memory) **writes**
  content derived from tool returns, assistant summaries, scraped web, email bodies, or MCP returns —
  with no source-trust guard and no per-entry provenance tag; worse if the same principal also has
  egress (cross-session exfil trifecta).
- **Why:** no memory backend is safe by construction — provenance blindness lets a dormant payload be
  written in one session and activated to exfiltrate in a later one; write-time review happens in a
  context that lacks the trigger, so review and activation windows differ
  ([trojan-hippo-memory-attack](https://agentpatterns.ai/security/trojan-hippo-memory-attack/), also
  cited by the auditor `memory-and-retrieval-poisoning` cluster).
- **Fix:** restrict memory writes to user-authored / human-curated sources; attach a per-entry
  provenance tier and filter on it at retrieval; gate writes with a confirmation step; remove one
  trifecta leg (deny untrusted writes / tokenize PII / default-deny egress). **High** when the audited
  artifact shows untrusted writes + egress co-existing on one principal — score it locally when one
  scan surfaces both; else **Medium**. *Seam:* MR-I3 owns the **memory-store audit**; the
  cross-session trifecta composition that co-existence enables (write→read pivot to exfiltration) is
  `audit-lethal-trifecta` (LT-A4) — a High here also names that handoff in the finding. Remediation:
  [learn — the-payload-that-waits](https://learn.agentpatterns.ai/security/the-payload-that-waits/).

---

## Detector → family map (for the report)

| Family | Detectors | Boundary property |
|---|---|---|
| Authorization | MR-A1 unscoped · MR-A2 post-only · MR-A3 agent-supplied id · MR-A4 state scope / SDK knobs · MR-A5 PII | the requester may see this chunk |
| Integrity | MR-I1 RAG poisoning · MR-I2 oracle poisoning · MR-I3 memory poisoning | the chunk is what it claims, from a trustable source |

> Audit **per call site**; an aggregate-tenant-aware pipeline hides one unfiltered retrieval. Mark
> partial controls (post-only filter, regex-only PII, signature-only provenance) as partial — never
> zero an ambiguous control to a clean "No".
