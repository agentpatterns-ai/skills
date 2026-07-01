---
name: audit-mcp-server
description: Audit an MCP server's own declaration — primitive choice (tool/resource/prompt), tool naming, input/output schemas, behavior annotations, error channels, transport, server instructions, and tool-count/token budget — against the MCP spec and agent-friendliness rules. Invoke when authoring, reviewing, or publishing an MCP server, or vetting a third-party server before wiring it in. Skip when auditing a lone (non-MCP) tool/function definition's quality (use audit-tool-definition) or the whole agent harness's exfiltration/security architecture (use audit-lethal-trifecta).
user-invocable: true
version: "0.1.0"
usage: /audit-mcp-server [path-to-mcp-server-source-or-manifest]
---

# Audit MCP Server

An MCP server's declaration is the contract the host reads at startup and hands the agent.
Spec-correct ≠ agent-friendly: a valid MCP server can still misroute every call. This audit reads the
**server's own declaration** and flags deviations from the design rules in
[mcp-server-design](https://agentpatterns.ai/tool-engineering/mcp-server-design/).

**Stance — detect and recommend; read-only; change nothing without confirmation.**
- **Pick the primitive by *who invokes* before naming or schema.** Agent-invoked action → tool;
  read-only context the client attaches → **resource**; user-triggered workflow → **prompt**.
  Read-only context dumped as a tool is the costliest mistake — bloats the surface, misroutes
  ([mcp-server-design, *First Decision*](https://agentpatterns.ai/tool-engineering/mcp-server-design/)).
- **Annotations are claims, not enforcement.** `readOnlyHint`/`destructiveHint` etc. are metadata,
  *not trustable from untrusted servers* — never read a hint as a security control ([mcp-server-design,
  *Output Schema and Annotations*](https://agentpatterns.ai/tool-engineering/mcp-server-design/)).
- **Schemas don't sanitize input.** The stdio SDK runs commands even when the local process fails to
  start — unsanitized args are a command-injection hole no richer schema closes ([mcp-server-design,
  *When This Backfires*](https://agentpatterns.ai/tool-engineering/mcp-server-design/)).

## Input
- `path` (optional): an MCP server source / SDK module (`server.tool` / `registerTool` / FastMCP) or
  a tool-manifest JSON. Default: scan the repo for the declaration + its per-tool `name` /
  `description` / `inputSchema` / `outputSchema` / annotations + server `instructions`.

## Scope
Audits the **server's declaration / spec-conformance**: primitive choice, naming, schemas,
annotations, error channels, transport, server instructions, tool-count/token budget, scope
shadowing. **Out of scope:** a lone non-MCP tool/function def's generic quality (→ `audit-tool-definition`);
whether wiring it in creates a private-data × untrusted-input × egress trifecta (→ `audit-lethal-trifecta`);
the client/host harness that consumes it.

## Procedure
1. **Identify the surface.** List every primitive (tools, resources, prompts), each tool's fields,
   server `instructions`, transport, and total tool count / token footprint.
2. **Run the detectors** in [`checks.md`](checks.md) (MS-1…MS-9), one pass per tool plus a
   server-level pass for count/token budget, instructions, and scope shadowing.
3. **Classify severity.** High = misroutes or misleads the agent (wrong primitive, dishonest
   annotation, bare errors, bloated surface). Medium = degrades accuracy (missing examples/enums,
   absent output contract, no server instructions). Low/advisory = transport fit, input sanitization.
4. **Recommend the spec-correct fix per finding** (cite the rule); prefer the deterministic form
   (`additionalProperties:false`, an enum, an idempotency key) over a prose caveat. Hand annotation-
   trust and stdio-injection *whole-harness* risk to `audit-lethal-trifecta`.
5. **Report** with the template below. Read-only — apply nothing without confirmation.

## Checks
The nine detectors (MS-1 wrong primitive, MS-2 naming, MS-3 input schema, MS-4 output contract,
MS-5 annotations, MS-6 errors, MS-7 surface/token budget, MS-8 discoverability/version, MS-9
transport/sanitization) — each with Flags / Why+source / Fix — live in [`checks.md`](checks.md),
loaded when auditing.

## Worked example
A `get_config` tool returns the server's read-only settings; an agent must *call* it to see config.
→ **MS-1 (High):** read-only context exposed as a tool, not a resource the client attaches. Fix:
re-expose as a `config://settings` **resource** ([mcp-server-design, *First Decision*](https://agentpatterns.ai/tool-engineering/mcp-server-design/)).
Same server: `delete_records` annotated `readOnlyHint:true` → **MS-5 (High):** a dishonest hint that
misleads the confirmation gate; metadata, not a control — route the harness-trust question to
`audit-lethal-trifecta`.

## Output template
```
# MCP-server audit — <server name / path>
Surface: <N tools, M resources, K prompts> · transport: <stdio|HTTP> · ~<tokens> of tool defs

## Findings
| Sev | Check | Tool / surface | Flag | Spec-correct fix |
|---|---|---|---|---|
| High | MS-1 | get_config | read-only context as a tool | expose as a `config://` resource |
| High | MS-5 | delete_records | readOnlyHint:true on a destructive op | drop the hint; mark destructiveHint:true; harness-trust → audit-lethal-trifecta |
| Med  | MS-3 | search_orders | no examples / enums / additionalProperties:false | add 1–5 examples, enum the status field, close the schema |
| Med  | MS-7 | server | 24 tools, no lazy-discovery plan | split server or defer behind search/read (target <15) |

**Clean:** <primitives/tools that conform, and why.>
**Highest-impact fix:** <the one change to make first.>
**Routed out:** <whole-harness exfil → audit-lethal-trifecta; generic tool-def quality → audit-tool-definition.>
```

## Related
- Sibling **`audit-tool-definition`** — generic tool/function-def quality (name, description, input
  schema, errors) for *any* protocol; this skill owns the **MCP-specific** surface and defers generic
  parts to it.
- **`audit-lethal-trifecta`** — owns whole-harness exfiltration risk; this skill flags annotation-
  trust (MS-5) and stdio-injection (MS-9) as *design* defects and routes the architecture question there.
- Corpus: [tool-description-quality](https://agentpatterns.ai/tool-engineering/tool-description-quality/),
  [poka-yoke-agent-tools](https://agentpatterns.ai/tool-engineering/poka-yoke-agent-tools/),
  [rfc9457-machine-readable-errors](https://agentpatterns.ai/tool-engineering/rfc9457-machine-readable-errors/),
  [mcp-eager-vs-jit-loading](https://agentpatterns.ai/tool-engineering/mcp-eager-vs-jit-loading/),
  [scoped-mcp-server-discovery](https://agentpatterns.ai/tool-engineering/scoped-mcp-server-discovery/).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue — interactive: recommend and confirm first (never auto-file); autonomous: self-file — shaped like `track-backlog` (title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table). Each finding, in both the report **and** the filed issue, carries its **Fix → lesson** link — resolved by check ID via [`checks.md`](checks.md), which stays the link's canonical home (citation standard).

## Critical rules (read last)
- **Primitive first, by who invokes** — read-only context is a resource, not a tool.
- **Annotations are metadata, not enforcement** — a hint never gates a destructive op; trust questions
  go to `audit-lethal-trifecta`.
- **Schemas don't sanitize** — flag unsanitized stdio args as command injection.
- Read-only: recommend the spec-correct, deterministic fix; apply nothing without confirmation.
