# audit-tool-definition — detector catalog

Loaded on demand by `audit-tool-definition`. Run each detector against every tool definition in
scope. Each item: ID / **Flags** / **Why** (cited corpus) / **Fix** (remediation lesson). All
detection is read-only — flag and recommend, never rewrite (that is `write-tool-description`).

---

## Core ergonomics (first-class)

### TD-1 — Description states *what*, not *when* (missing selection signal)
- **Flags:** a description accurate about capability but with no "use this tool when X" / "prefer
  over [other tool] when Y", and no statement of the return shape. The top failure mode.
- **Why:** selection is a reasoning step over descriptions; positive selection signals are
  instructions to the agent, not interface docs ([tool-description-quality §Iterating, §Example](https://agentpatterns.ai/tool-engineering/tool-description-quality/)).
- **Fix:** add a positive trigger and describe the return shape — see `schema-and-description-altitude`.
  Remediation: [learn — schema-and-description-altitude](https://learn.agentpatterns.ai/tool-engineering/schema-and-description-altitude/).

### TD-2 — MCP description not self-contained
- **Flags:** an MCP tool description that leans on server-level prose or adjacent tools the agent may
  never read ("see the X tool", "as above").
- **Why:** each MCP tool description must be independently self-contained — agents may have no context
  from adjacent tools ([tool-description-quality §MCP Server Tool Descriptions](https://agentpatterns.ai/tool-engineering/tool-description-quality/)).
- **Fix:** make each description stand alone — see `schema-and-description-altitude`.
  Remediation: [learn — schema-and-description-altitude](https://learn.agentpatterns.ai/tool-engineering/schema-and-description-altitude/).

### TD-3 — Ambiguous parameter names + no onboarding context
- **Flags:** bare `user`/`date`/`id`/`type` with no qualifier; missing ID format, query syntax, valid
  filter values, or traversal order — the implicit context a new hire would need.
- **Why:** write descriptions as agent onboarding; surface implicit context and use unambiguous
  parameter names ([tool-descriptions-as-onboarding §Unambiguous Parameter Names, §What Implicit Context Looks Like](https://agentpatterns.ai/tool-engineering/tool-descriptions-as-onboarding/)).
- **Fix:** qualify names; state format/syntax/valid values — see `schema-and-description-altitude`.
  Remediation: [learn — schema-and-description-altitude](https://learn.agentpatterns.ai/tool-engineering/schema-and-description-altitude/).

### TD-4 — No poka-yoke on the schema
- **Flags:** free text where an enum belongs; unbounded params (no min/max/default); relative file
  paths; no read-before-write / prerequisite gate; a format requiring line-counting or escaping; **no
  concrete sample call** in the definition.
- **Why:** structurally wrong usage should fail at the interface — mandatory absolute paths killed a
  whole failure class, and a sample call lifted complex-parameter accuracy 72% → 90%
  ([poka-yoke-agent-tools §Absolute Paths, §Tool Use Examples in Definitions](https://agentpatterns.ai/tool-engineering/poka-yoke-agent-tools/)).
- **Fix:** enum the fixed-value params; add bounds+defaults; require absolute paths; add a sample call
  — see `schema-and-description-altitude` and `idempotent-tools`.
  Remediation: [learn — schema-and-description-altitude](https://learn.agentpatterns.ai/tool-engineering/schema-and-description-altitude/).

### TD-6 — Oversized / full-passthrough output + opaque identifiers
- **Flags:** returns the whole API object when a paragraph would do; UUIDs/MIME/internal codes the
  agent can't reason over; no pagination/filtering/`response_format` enum; no sensible default limit.
- **Why:** tool output is a context injection — attention dilutes and the middle is lost; return only
  the next decision's inputs, with semantic identifiers and tool-layer filtering
  ([token-efficient-tool-design §Sizing Tool Output](https://agentpatterns.ai/token-engineering/token-efficient-tool-design/),
  [semantic-tool-output §Principles](https://agentpatterns.ai/tool-engineering/semantic-tool-output/)).
- **Fix:** size to the next decision; semantic ids; add `response_format` + a default limit — see
  `token-efficient-tool-design`, `result-shaping`.
  Remediation: [learn — token-efficient-tool-design](https://learn.agentpatterns.ai/tool-engineering/token-efficient-tool-design/).

### TD-8 — Errors are opaque/diagnostic, not actionable
- **Flags:** `"400 Bad Request"` / HTML error bodies; no "what to try next"; for HTTP services no
  RFC 9457 fields (`type`/`title`/`detail` + extensions `retryable`, `retry_after`,
  `owner_action_required`).
- **Why:** errors steer behavior — machine-readable, actionable errors let the agent branch control
  flow instead of guessing ([rfc9457-machine-readable-errors](https://agentpatterns.ai/tool-engineering/rfc9457-machine-readable-errors/),
  [semantic-tool-output §Make Errors Actionable](https://agentpatterns.ai/tool-engineering/semantic-tool-output/),
  [tool-descriptions-as-onboarding §Steering Behavior Through Error Messages](https://agentpatterns.ai/tool-engineering/tool-descriptions-as-onboarding/)).
- **Fix:** emit RFC 9457 base + extension fields with a next-step hint — see `errors-as-teaching-signal`.
  Remediation: [learn — errors-as-teaching-signal](https://learn.agentpatterns.ai/tool-engineering/errors-as-teaching-signal/).

### TD-9 — MCP annotation misdeclaration
- **Flags:** `readOnlyHint:true` on a tool whose implementation mutates anything (even a `last_seen`
  write); a pure read missing the read-only triple (`readOnlyHint:true`, `idempotentHint:true`,
  `destructiveHint:false`).
- **Why:** annotations are load-bearing once a harness wires `readOnlyHint` into parallel dispatch — a
  misannotated mutating tool races ([read-only-hint-concurrency §The Contract Shift](https://agentpatterns.ai/tool-engineering/read-only-hint-concurrency/)).
- **Fix:** verify the hint against the implementation; annotate pure reads fully — see
  `annotations-and-concurrency`. *Disposition:* when the input is a `tools/list` payload / `.mcp.json`
  with **no implementation visible** (third-party vetting), report the annotation as
  "UNVERIFIED against implementation" — not pass/fail; the missing read-only triple on a
  self-described pure read stays definition-visible and verdictable. *Seam:* TD-9 verifies annotation
  **truth against the implementation**; declaration-level presence/defaults across a server is
  `audit-mcp-server` MS-5.
  Remediation: [learn — annotations-and-concurrency](https://learn.agentpatterns.ai/tool-engineering/annotations-and-concurrency/).

### TD-10 — Non-idempotent create/append annotated idempotent, or no idempotency affordance
- **Flags:** `idempotentHint:true` on an append/create tool with no `idempotency_key`; no
  check-before-act or uniqueness key on a retryable mutation.
- **Why:** idempotency needs check-before-act and unique identifiers; declaring it without an
  idempotency key is a false contract ([idempotent-agent-operations §Core Techniques](https://agentpatterns.ai/agent-design/idempotent-agent-operations/),
  [poka-yoke-agent-tools §Uniqueness Constraints](https://agentpatterns.ai/tool-engineering/poka-yoke-agent-tools/)).
- **Fix:** require an `idempotency_key`/uniqueness key before annotating `idempotentHint` — see
  `idempotent-tools`. *Disposition:* schema-visible (the key's presence/absence is in the definition);
  whether the backend truly upserts is implementation-side — mark "UNVERIFIED" when no source is
  available rather than guessing.
  Remediation: [learn — idempotent-tools](https://learn.agentpatterns.ai/tool-engineering/idempotent-tools/).

---

## Tail (load on demand / advisory)

### TD-5 — Wrong altitude / over-prescribed call sequence
- **Flags:** a call sequence baked into the description ("*always* call `list_sprints` first"), making
  the tool brittle when the ID is already known.
- **Why:** over-specifying expected call sequences backfires — it removes the agent's room to act on
  context it already has ([tool-descriptions-as-onboarding §When This Backfires](https://agentpatterns.ai/tool-engineering/tool-descriptions-as-onboarding/)).
- **Fix:** describe capability + selection signal, not a fixed sequence — see `schema-and-description-altitude`.
  Remediation: [learn — schema-and-description-altitude](https://learn.agentpatterns.ai/tool-engineering/schema-and-description-altitude/).

### TD-7 — No graceful-truncation contract for variable-size output
- **Flags:** a tool with potentially large output that hard-errors instead of returning a useful
  prefix + a structurally-distinct PARTIAL marker + a continuation handle; or a trailing-prose marker
  the model ignores.
- **Why:** the contract has three load-bearing parts; a trailing marker fails because the model
  doesn't reliably read past the payload ([graceful-tool-output-truncation §The Contract](https://agentpatterns.ai/tool-engineering/graceful-tool-output-truncation/)).
- **Fix:** prefix + distinct PARTIAL marker + continuation handle — see `result-shaping`.
  Remediation: [learn — result-shaping](https://learn.agentpatterns.ai/tool-engineering/result-shaping/).

### TD-11 — Overlap / sprawl at the definition layer *(advisory — scope-edge)*
- **Flags:** two tools with non-mutually-exclusive descriptions; API-shaped one-tool-per-endpoint
  where the pair is always called together; related tools with no namespace prefix; toolset bloat
  inflating per-call selection cost.
- **Why:** consolidating overlapping tools and namespace-grouping cuts selection cost — toolset bloat
  drove a 7–85% accuracy drop on LongFuncEval ([consolidate-agent-tools §Consolidation Principle, §Namespace Grouping](https://agentpatterns.ai/tool-engineering/consolidate-agent-tools/),
  [token-efficient-tool-design §Cap Toolset Size](https://agentpatterns.ai/token-engineering/token-efficient-tool-design/)).
- **Fix:** report definition-visible signals only (overlapping descriptions, missing namespace
  prefix); note count/bloat as advisory — see `consolidation-vs-sprawl`, `tool-discoverability-at-scale`.
  A standalone toolset-curation skill, if spun out later, owns the set-level half.
  Remediation: [learn — consolidation-vs-sprawl](https://learn.agentpatterns.ai/tool-engineering/consolidation-vs-sprawl/).

### TD-12 — ToolLeak / malicious definition *(security hand-off — corpus gap)*
- **Flags:** a `description` or `inputSchema` that requests the system prompt, conversation history,
  secrets, or API keys.
- **Action:** **refuse at install; do not reword.** Hand off to `audit-lethal-trifecta` — this is a
  security/exfiltration concern, not tool ergonomics.
  Remediation: [learn — the-lethal-trifecta](https://learn.agentpatterns.ai/security/the-lethal-trifecta/).
- **Note:** there is **no agentpatterns.ai corpus page** backing this; the only source is a
  project-auditor rule (provenance in the harness source-map). Do **not** ship it as a core normative
  ergonomics claim — flag-and-route only, pending a commissioned corpus page.
