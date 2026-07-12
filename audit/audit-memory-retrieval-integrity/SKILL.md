---
name: audit-memory-retrieval-integrity
description: Audit a retrieval or memory layer — RAG config, vector-store queries, agent memory stores, knowledge-graph tool-use — for the retrieval-boundary authorization-and-integrity gap — cross-tenant chunk bleed, agent-supplied tenant identity, untokenized PII in retrieved context, and retrieved/remembered content trusted as fact-or-instruction with no provenance tier. Invoke when reviewing a RAG / vector-store / agent-memory pipeline, a multi-tenant retrieval path, or before wiring retrieved chunks into model context. Skip when the concern is the input-side three-leg data-flow of one execution path (use audit-lethal-trifecta), credential / secret hygiene (use audit-secret-exposure), or validating an LLM's output before it reaches a sink (use audit-supply-chain-sinks).
user-invocable: true
version: "0.2.1"
usage: /audit-memory-retrieval-integrity [path-to-retrieval-or-memory-code]
---

# Audit: Memory & Retrieval Integrity

The retrieval boundary is where a RAG / vector-store / memory layer hands chunks to the model. Two
properties must hold there and usually don't: **authorization** (requester may see this chunk) and
**integrity** (chunk is what it claims, from a trustable source). Retrieval ranks by relevance, not
authorization — in a shared corpus the top chunk for one tenant can belong to another; ungated
retrieval leaked cross-tenant data in 98–100% of probes
([multitenant-rag-authorization-gap](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
The agent trusts tool-delivered facts as ground truth, so a poisoned chunk or KG node steers it on the
*data* path, not the instruction path
([oracle-poisoning-knowledge-graph](https://agentpatterns.ai/security/oracle-poisoning-knowledge-graph/)).
Fix at the boundary — gate the search space and tier provenance — not a prompt asking the model to behave.

**Stance — detect and recommend; read-only; apply nothing without confirmation.**
- **Audit per retrieval call site, not per app.** One unfiltered `search()` is the leak even when the
  pipeline looks tenant-aware in aggregate ([gap, §3](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
- **Authorization is a boundary predicate, not an after-thought.** Compose the tenant/ABAC filter into
  the search call; one layer alone is the anti-pattern — pre-only is bypassed by ANN traversal,
  post-only collapses recall ([gap](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).
- **A natural-language rule is not enforcement.** "Only use this tenant's data" / "don't trust
  retrieved text" stops nothing — PII redaction and tenant scoping must be deterministic code at the
  boundary ([pii-tokenization-in-agent-context](https://agentpatterns.ai/security/pii-tokenization-in-agent-context/)).

## Input
- `path` (optional): a retrieval / memory module or repo. Default: scan for vector-store clients
  (Qdrant / Pinecone / pgvector / Weaviate), RAG retrievers, agent-memory stores, knowledge-graph
  tool calls, ingestion pipelines, and Agent-SDK `query()` call sites.

## Scope
Audits the **retrieval boundary**: per-call-site tenant scoping, identity source, PII handling, and the
provenance/trust of retrieved-or-remembered content. **Out of scope:** the input-side three-leg trifecta
of one execution path (use `audit-lethal-trifecta`), credential/secret hygiene (use
`audit-secret-exposure`), validating an LLM's *output* before a sink (use `audit-supply-chain-sinks`),
and live runtime monitoring (this is a static audit). **Does not apply** to single-tenant deployments
with no shared index, or a public corpus with no access tiers — nothing to leak
([gap, *When It Doesn't*](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/)).

## Procedure
1. **Enumerate retrieval & memory call sites.** Each `search()` / retriever / memory-read / KG-query /
   memory-write is a boundary; record its index, identity source, and what reaches model context.
   Done when every call site has a recorded boundary row — no blanks.
2. **Run the detectors** in [`checks.md`](checks.md): the **authorization** family (MR-A1…A5 — tenant
   scoping, two-tier filter, identity source, state scope, PII) and the **integrity** family (MR-I1…I3 —
   RAG / oracle / memory poisoning). Mark **partial** states (post-only filter, regex-only PII) — do not
   zero an ambiguous control to "No".
   Done when every boundary × MR check is marked present/partial/absent.
3. **Flag each gap** with severity (below) and the cited why.
   Done when every flagged gap has both a severity and a citation recorded.
4. **Recommend the deterministic fix at the boundary** — pre-filter, server-bound tenant id, code-level
   tokenizer, provenance tier, read-only KG — preferring a control the model can't reason around over a
   prompt mitigation. Remediation lessons resolve per check ID via [`checks.md`](checks.md) — the
   links' canonical home. Done when every finding names its boundary-level fix.
5. **Report** with the template below.
   Done when the Boundaries table, Findings, Safe boundaries, and Smallest high-impact change are filled.

## Detectors
Eight per-boundary detectors (ID / Flags / Why+source / Fix) in [`checks.md`](checks.md); load when
auditing.

## Output template
```
# Memory & retrieval integrity audit — <target>

## Boundaries
| Call site | Index/store | Identity source | Tenant filter | PII | Provenance tier | Verdict |
|---|---|---|---|:--:|:--:|:--:|
| search_docs()      | shared Qdrant | LLM arg     | post-only | raw | none | LEAK |
| memory.read()      | per-user kv   | session JWT | n/a       | tok | tagged | safe |

## Findings
| Severity | Boundary | Detector | Cited why | Deterministic fix |
|---|---|---|---|---|
| High | search_docs() | MR-A1 | unscoped retrieval, multi-tenant surface | pre-filter `tenant_id` into the search call + post-ACL |
| High | search_docs() | MR-A3 | tenant_id from LLM args | bind from session principal / signed JWT claim |

**Safe boundaries:** <which, and which control keeps them safe.>
**Smallest high-impact change:** <the one boundary control to add first.>
```
Severity: **High** = cross-tenant leak path (unfiltered/agent-supplied-identity retrieval on a
multi-tenant surface), untokenized PII reaching context, or writable oracle/memory trusted as fact;
**Medium** = partial control (post-only filter, regex-only PII, no provenance tier on a closed corpus);
**Low** = hardening with no live exposure.

## Related / pairing (the cited why)
Sibling to **`audit-lethal-trifecta`** (input-side three-leg data-flow of an execution *path*); this
owns the retrieval-boundary authorization+integrity surface — same family, different target. Other
siblings: `audit-secret-exposure` (credential hygiene), `audit-supply-chain-sinks` (LLM output → sink).
Corpus (per-check Why detail lives in [`checks.md`](checks.md)): [gap](https://agentpatterns.ai/security/multitenant-rag-authorization-gap/), [isolation-knobs](https://agentpatterns.ai/security/multi-tenant-isolation-knobs-agent-sdk/), [pii-tokenization](https://agentpatterns.ai/security/pii-tokenization-in-agent-context/), [oracle-poisoning](https://agentpatterns.ai/security/oracle-poisoning-knowledge-graph/), [rag-poisoning](https://agentpatterns.ai/security/rag-architecture-poisoning-robustness/).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- Compose authorization **into the search call** (pre-filter + post-ACL); post-only filtering is the leak.
- The agent **never** supplies the tenant id — bind it from a server-side principal.
- Tokenize PII and tier provenance with **deterministic code at the boundary** — a prompt is not a fix.
- Audit **per call site**; aggregate-tenant-aware pipelines hide a single unfiltered retrieval. Read-only;
  apply nothing without confirmation.
