---
name: audit-tool-definition
description: Audit one or many individual agent-facing tool/function definitions — name, description, inputSchema, output/error contract, and MCP annotations — for the ergonomics failures that make an agent mis-select or misuse a tool, and emit a findings report. Invoke when reviewing the quality of individual tool definitions (function-calling specs, or the per-tool defs an MCP server exposes) or diagnosing why an agent picks or calls a tool wrong. Skip when the goal is to produce the corrected description/schema for one tool (use write-tool-description), to audit an MCP server's own declaration — primitive choice, transport, server instructions, tool-count budget, annotation presence/defaults — rather than its individual tool defs (use audit-mcp-server), or to audit per-path data/egress security architecture (use audit-lethal-trifecta).
user-invocable: true
version: "0.2.0"
usage: /audit-tool-definition [path-to-tool-defs-or-mcp-config]
---

# Audit Tool Definition

An agent selects and calls a tool by **reasoning over its definition** — `name`, `description`,
`inputSchema` are its only context at decision time. A definition that states *what* a tool does but
not *when* to reach for it, names parameters ambiguously, or returns an opaque full-API blob makes
the agent mis-select or misuse it — and the fix is the **definition**, not a longer prompt. Improving
tool ergonomics cut task-completion time **40%** in one Anthropic case; a concrete sample call lifted
complex-parameter accuracy **72% → 90%** ([tool-description-quality](https://agentpatterns.ai/tool-engineering/tool-description-quality/),
[poka-yoke-agent-tools](https://agentpatterns.ai/tool-engineering/poka-yoke-agent-tools/)).

**Stance — detect, inventory, recommend; read-only; rewrite nothing.** This skill *flags* definition
problems across one or many tools and reports them. It does **not** author the corrected text — hand
a single tool's rewrite to **`write-tool-description`**. It does **not** own security: a large-output
tool inside a trifecta session, or a `description`/`inputSchema` requesting the system prompt /
secrets / API keys (ToolLeak), is flagged and **handed off to `audit-lethal-trifecta`**, not treated
as an ergonomics nit. A "use me wisely" note in a description is documentation, not a guardrail —
prefer a schema that makes misuse structurally impossible (enum, bound, absolute-path requirement)
over prose that asks the model to behave.

## Input
- `path` (optional): a dir/repo with tool definitions. Default scan: `.mcp.json` / MCP server source
  exposing `name`/`description`/`inputSchema`, `tools/list` payloads, function-calling tool specs in
  code (Python/TS dicts or decorators), and OpenAPI specs that *generate* tools (audit the spec, not
  the generated def).

## Scope
Audits **per-tool ergonomics**: selection signal, self-containment, parameter clarity, schema
poka-yoke, output/error contract, MCP annotations. **Out of scope:** rewriting a definition
(`write-tool-description`); an MCP server's own *declaration* — primitive choice, transport, server
instructions, tool-count budget (`audit-mcp-server`); per-path data/egress/untrusted-reach security
(`audit-lethal-trifecta`); toolset-shape curation (overlap/namespace/count) reported only at the
**definition-visible** level (TD-11), advisory on count.

## Procedure
1. **Enumerate every tool definition** in scope; record `name`, `description`, `inputSchema`,
   output/error contract, and any MCP annotations (`readOnlyHint`, `idempotentHint`,
   `destructiveHint`, `openWorldHint`).
   Done when every in-scope tool has all five fields recorded — no blanks.
2. **Run the detectors in [`checks.md`](checks.md)** (TD-1…TD-12) against each. Do not collapse an
   ambiguous signal to a false pass — note it.
   Done when every tool × check pair has a recorded verdict.
3. **Cross-check mutating-vs-read annotations** against the *implementation*: a `readOnlyHint:true`
   tool that writes anything is a misannotation (TD-9), load-bearing once a harness wires the hint
   into parallel dispatch.
   Done when every annotated tool has a recorded verdict — match, mismatch, or UNVERIFIED (definition-only input).
4. **Hand off, don't own:** a ToolLeak/secrets-exfil definition (TD-12) or an output-size finding
   *inside a trifecta session* → `audit-lethal-trifecta`; single-tool rewrites → `write-tool-description`.
   Done when every routed finding names its destination skill.
5. **Report** with the template below — one finding row per (tool × check), severity-ranked.
   Done when every finding has its severity-ranked row in the report.

## Checks
The 12 detectors (ID / Flags / Why+citation / Fix-lesson) live in [`checks.md`](checks.md); load when
auditing. Core ergonomics: TD-1 selection signal, TD-2 MCP self-containment, TD-3 parameter
clarity, TD-4 schema poka-yoke, TD-6 output sizing, TD-8 actionable errors, TD-9 annotation truth,
TD-10 idempotency. Tail (load on demand): TD-5 altitude, TD-7 graceful truncation, TD-11
overlap/sprawl (advisory), TD-12 ToolLeak (security hand-off, corpus-gap — not a core normative claim).

## Output template
```
# Tool-definition audit — <target>

## Tools inventoried
| Tool | Surface checked | Annotations | Verdict |
|---|---|---|---|
| get_sprint_issues | desc + schema + output + errors | (none) | 5 findings |
| search_issues     | desc + schema + output + errors | readOnlyHint, idempotentHint | clean |

## Findings
| Severity | Tool | Check | What's wrong | Fix (remediation) |
|---|---|---|---|---|
| High   | get_sprint_issues | TD-9 | readOnlyHint:true but impl writes a last_seen timestamp | drop readOnlyHint or stop the write — see annotations-and-concurrency |
| High   | get_sprint_issues | TD-4 | bare `sprint_id` string, no enum on `status`, no sample call | enum the status; add a sample call — schema-and-description-altitude |
| Medium | get_sprint_issues | TD-1 | "Get issues for a sprint." — no "use when / prefer over" signal | add a positive selection signal — schema-and-description-altitude |
| Medium | get_sprint_issues | TD-6 | returns the 40-field raw API object with UUIDs; no limit/response_format | size to the next decision; semantic ids — token-efficient-tool-design |
| Medium | get_sprint_issues | TD-8 | errors are "400 Bad Request" with no "what to try next" | RFC 9457 fields + actionable hint — errors-as-teaching-signal |

**Hand-offs:** <ToolLeak/security findings → audit-lethal-trifecta; single-tool rewrites → write-tool-description.>
**Clean tools:** <which passed all detectors, and why they're the model.>
**Smallest high-impact change:** <the one definition edit with the biggest selection/safety payoff.>
```
Severity: **High** = a misannotation a harness acts on (TD-9/TD-10) or a structural misuse that
fails silently; **Medium** = a selection/output/error ergonomics defect; **Low** = advisory
(toolset count/namespace, TD-11).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue — interactive: recommend and confirm first (never auto-file); autonomous: self-file — shaped like `track-backlog` (title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table). Each finding, in both the report **and** the filed issue, carries its **Fix → lesson** link — resolved by check ID via [`checks.md`](checks.md), which stays the link's canonical home (citation standard).

## Critical rules (read last)
- **Flag, don't fix.** Inventories definition problems; rewriting one tool is
  `write-tool-description`. Keep the cross-reference reciprocal.
- **Annotation truth is load-bearing.** Verify `readOnlyHint`/`idempotentHint` against the
  implementation, not the prose — a misannotated read races under parallel dispatch
  ([read-only-hint-concurrency](https://agentpatterns.ai/tool-engineering/read-only-hint-concurrency/)).
- **A description note is not a guardrail.** Prefer a schema constraint (enum/bound/absolute path)
  the model can't override over prose asking it to behave.
- **Hand security off.** ToolLeak/secrets-exfil and output-size-inside-a-trifecta belong to
  `audit-lethal-trifecta`; this skill flags and routes, it does not own them.
