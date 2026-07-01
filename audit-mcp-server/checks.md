# audit-mcp-server — detectors

Loaded on demand by `audit-mcp-server`. Run one pass per tool, plus a server-level pass for MS-7/8.
Each item: ID / **Flags** / **Why** (cited) / **Fix**. All detection is read-only. Severity in the
intro of each check is the default; downgrade only with a stated reason.

---

### MS-1 — Wrong primitive *(High)*
- **Flags:** read-only context (config, schema, env/info) exposed as a **tool** the agent must call,
  rather than a **resource** the client attaches; a user-triggered multi-step workflow not modelled
  as a **prompt**.
- **Why:** the first decision is *who invokes* — tools = agent-invoked actions, resources = read-only
  context, prompts = user-triggered workflows; getting it wrong misroutes and bloats the surface
  ([mcp-server-design, *First Decision*](https://agentpatterns.ai/tool-engineering/mcp-server-design/) —
  "Read-only context is exposed as resources, not tools").
- **Fix:** re-expose read-only context as a resource (`config://…`); model the workflow as a prompt.
  Remediation: [learn — what-a-server-exposes](https://learn.agentpatterns.ai/mcp-server-design/what-a-server-exposes/).

### MS-2 — Non-conforming tool names *(Med)*
- **Flags:** names not `verb_noun` snake_case ≤32 chars; version numbers or abbreviations in the name
  (`prod_lookup_v2`, `cfgLookupV2`); opaque names that won't match tool search.
- **Why:** `verb_noun` snake_case ≤32 chars is the deterministic naming rule; descriptive names match
  tool search, version-baked names break on every bump ([mcp-server-design, *Tool Naming*](https://agentpatterns.ai/tool-engineering/mcp-server-design/);
  [tool-description-quality](https://agentpatterns.ai/tool-engineering/tool-description-quality/)).
- **Fix:** rename to `verb_noun` ≤32 chars; move the version out of the name. Remediation:
  [learn — the-right-call-obvious](https://learn.agentpatterns.ai/mcp-server-design/the-right-call-obvious/).

### MS-3 — Input schema without guardrails *(Med)*
- **Flags:** `inputSchema` missing `additionalProperties:false`; a parameterless tool that omits the
  schema instead of `{"type":"object","additionalProperties":false}`; no enums/defaults where the
  value set is stable; descriptions lacking 1–5 realistic examples; no negative guidance ("do NOT use
  for X — use Y").
- **Why:** schemas define types but not usage — 1–5 realistic examples raised tool-use accuracy
  72%→90%; enums and `additionalProperties:false` cut guesswork; negative guidance prevents misrouting
  ([mcp-server-design, *Schema Design*](https://agentpatterns.ai/tool-engineering/mcp-server-design/);
  [poka-yoke-agent-tools](https://agentpatterns.ai/tool-engineering/poka-yoke-agent-tools/)).
- **Fix:** close the schema, enum stable fields, add defaults, add 1–5 examples + negative guidance.
  Remediation: [learn — the-right-call-obvious](https://learn.agentpatterns.ai/mcp-server-design/the-right-call-obvious/).

### MS-4 — Missing / unpaired output contract *(Med)*
- **Flags:** no `outputSchema` where results have stable structure; or `outputSchema` declared but the
  tool returns only `structuredContent` (or only a text blob) instead of **both** `structuredContent`
  and a serialized JSON copy in `content`.
- **Why:** `outputSchema` enables structured validation; return both the validated `structuredContent`
  and serialized JSON in `content` for backwards compatibility with older clients ([mcp-server-design,
  *Output Schema and Annotations*](https://agentpatterns.ai/tool-engineering/mcp-server-design/)).
- **Fix:** declare `outputSchema`; return both channels. Remediation: [learn — typed-and-hinted](https://learn.agentpatterns.ai/mcp-server-design/typed-and-hinted/).

### MS-5 — Dishonest or absent behavior annotations *(High)*
- **Flags:** behavior defaults not overridden explicitly on every tool; pure-read tools missing the
  `readOnlyHint:true` + `idempotentHint:true` + `destructiveHint:false` triple; `idempotentHint:true`
  on an append/create tool with no idempotency key; a destructive tool labelled `destructiveHint:false`.
- **Why:** annotations are **metadata only, not trustable from untrusted servers** — a hint never
  gates anything; a dishonest hint misleads the client's confirmation prompt ([mcp-server-design,
  *Output Schema and Annotations*](https://agentpatterns.ai/tool-engineering/mcp-server-design/)).
  The override-the-defaults and idempotency-key requirements are auditor rules (`mcp-tool-annotations`)
  citing [mcp-client-server-architecture](https://agentpatterns.ai/tool-engineering/mcp-client-server-architecture/) —
  verify before stating "defaults to true" normatively.
- **Fix:** set the read-only triple on pure reads; require an idempotency key for `idempotentHint:true`;
  correct destructive labels. **Hand the harness trust question to `audit-lethal-trifecta`** — a hint
  is not a control. Remediation: [learn — typed-and-hinted](https://learn.agentpatterns.ai/mcp-server-design/typed-and-hinted/).

### MS-6 — Generic errors / wrong error channel *(High)*
- **Flags:** execution errors returned as bare `"Error"`/`"Invalid"` strings instead of `isError:true`
  carrying the violated constraint + the violation + recovery context; business-logic failures raised
  as JSON-RPC protocol errors (or protocol errors hidden inside results).
- **Why:** two channels — **protocol errors** (JSON-RPC codes) are for the client; **tool execution
  errors** (`isError:true`) are for the agent and the spec says they must contain "actionable feedback
  that language models can use to self-correct and retry" ([mcp-server-design, *Error Handling*](https://agentpatterns.ai/tool-engineering/mcp-server-design/);
  [rfc9457-machine-readable-errors](https://agentpatterns.ai/tool-engineering/rfc9457-machine-readable-errors/)).
- **Fix:** return `isError:true` with constraint+violation+recovery; keep protocol vs execution errors
  on their own channels. Remediation: [learn — the-right-call-obvious](https://learn.agentpatterns.ai/mcp-server-design/the-right-call-obvious/).

### MS-7 — Bloated / preloaded surface *(High)*
- **Flags:** >15 tools per server; a corpus/documents dumped into context at startup instead of behind
  `search`/`read` tools; a large catalog (≥10 tools or >10K tokens of definitions) with no lazy-
  discovery/deferral plan; overlapping tools forcing disambiguation.
- **Why:** keep tool count under 15 — selection accuracy degrades past ~30–50 visible tools; load data
  just-in-time, not eagerly ([mcp-server-design, *Token Efficiency* + checklist "under 15 tools"](https://agentpatterns.ai/tool-engineering/mcp-server-design/);
  [token-efficient-tool-design](https://agentpatterns.ai/token-engineering/token-efficient-tool-design/);
  [mcp-eager-vs-jit-loading](https://agentpatterns.ai/tool-engineering/mcp-eager-vs-jit-loading/);
  [retrieval-augmented-agent-workflows](https://agentpatterns.ai/context-engineering/retrieval-augmented-agent-workflows/)).
- **Fix:** split the server or defer the catalog behind `search`/`read`; target <15 tools. Remediation:
  [learn — found-and-versioned](https://learn.agentpatterns.ai/mcp-server-design/found-and-versioned/).

### MS-8 — Discoverability / version-stability defects *(Med)*
- **Flags:** no server-level `instructions` for tool-search discovery (silent search-miss → false "no
  tool available"); `enum` on a fast-churning upstream field; a version baked into a tool name; a
  server name that collides across scopes and gets silently shadowed.
- **Why:** write clear server `instructions` so tool search finds yours; `enum` encodes a snapshot and
  breaks when the upstream adds a value until redeploy — use a thin string for evolving fields; across
  scopes the **most-specific definition wins** and duplicates are suppressed before the agent sees them
  ([mcp-server-design, *Design for lazy discovery* / *Enums vs. evolving upstream APIs*](https://agentpatterns.ai/tool-engineering/mcp-server-design/);
  [scoped-mcp-server-discovery](https://agentpatterns.ai/tool-engineering/scoped-mcp-server-discovery/)).
- **Fix:** add server `instructions`; drop enums on churning fields; give parallel-intent servers
  distinct names. Remediation: [learn — found-and-versioned](https://learn.agentpatterns.ai/mcp-server-design/found-and-versioned/).

### MS-9 — Transport / input-sanitization defects *(Advisory)*
- **Flags:** stdio where shared state / multiple concurrent clients demand Streamable HTTP (or HTTP for
  a single local tool); an HTTP server that doesn't validate `Origin` / bind localhost / authenticate;
  a stdio server that doesn't sanitize args.
- **Why:** transport is a topology choice, not a latency tweak; **schemas do not cover input
  sanitization** — the stdio SDK runs commands even when the local process fails to start, so
  unsanitized args are a command-injection hole ([mcp-server-design, *When This Backfires*](https://agentpatterns.ai/tool-engineering/mcp-server-design/);
  [mcp-client-server-architecture](https://agentpatterns.ai/tool-engineering/mcp-client-server-architecture/)).
- **Fix:** match transport to topology; validate `Origin`/bind localhost/auth on HTTP; sanitize stdio
  args. **Route whole-harness exfiltration risk to `audit-lethal-trifecta`.** Remediation:
  [learn — what-a-server-exposes](https://learn.agentpatterns.ai/mcp-server-design/what-a-server-exposes/).
