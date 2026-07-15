---
name: audit-prompt-cache-hygiene
description: Audit an agent harness — prompt-assembly code, tool-definition serialization, and SDK/CLI cache config — for prompt-cache hygiene — volatile content in the static prefix, non-deterministic or mutated tool enumeration, mid-session model/effort/MCP switches, missing cache-usage monitoring, TTL/idle-shape misfit, cache-busting compaction, and cache-economics misfit. Invoke when reviewing an agent loop / SDK integration / harness that assembles a repeated prompt prefix and calls a hosted model across many turns, to find cache-busting cost regressions. Skip when auditing a prose instruction file's content/attention (use audit-instruction-file), or a single-turn / cold-start / local-no-KV-reuse flow with no prefix to protect.
user-invocable: true
version: "0.4.0"
usage: /audit-prompt-cache-hygiene [path-to-harness-or-sdk-integration]
---

# Prompt-Cache Hygiene Audit

Prefix caching requires an **exact byte-level match**: on **Anthropic explicit caching**, cached reads cost ≈10% of base input and writes carry a 25–100% premium (OpenAI prompt caching is automatic for prompts ≥1024 tokens — no write premium, ≈50%+ read discount; [OpenAI — prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)), so one mutated byte *before the current turn* invalidates
everything after it — those tokens lose the ≈10% cached rate and re-bill at base input (re-written
breakpointed segments at the write premium); segments before the last surviving breakpoint still hit ([prompt-caching-architectural-discipline](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/);
[static-content-first-caching](https://agentpatterns.ai/context-engineering/static-content-first-caching/)).
The fix is **architectural** — immutable static prefix, dynamic tail, session boundaries — not a
comment telling it to "keep the prefix stable."

**Stance — detect and recommend; read-only audit; apply nothing without confirmation.**
- **Cache misses are SILENT.** They bill the full rate and never error — the Claude Code SDK
  `query()` bug busted the cache every call, costing up to **12× more input tokens** until
  fixed. The audit must inspect the *meter* (`cache_read_input_tokens` vs `cache_creation_input_tokens`,
  or the provider's equivalent — OpenAI `cached_tokens`, Google hit metadata),
  not just the layout ([caching discipline → SDK Cache Invalidation; Monitoring Cache Health](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/)).
- **Prefer deterministic controls.** Build `system`/`tools` once outside the call loop, sort tool
  defs by name, fix model/effort/MCP at session start, set `excludeDynamicSections` across a fleet.
  A natural-language "don't mutate the prefix" rule enforces nothing.
- **Guard the economics first.** Flagging cache discipline where it loses money (sub-minimum prefix,
  ≤2-turn / 5–10-call sessions, mostly-dynamic prefix) is a false positive — see PC-9.

## Input
- `path` (optional): a harness dir / SDK integration / agent loop. Default: scan prompt-assembly code
  (`compose_prompt`, `build_system_prompt`, `messages=`/`system=`/`tools=` construction, tool-list
  builders), SDK/CLI call sites + options objects, and cache config (`settings.json`,
  `cache_control` breakpoints, `excludeDynamicSections`, TTL flags).

## Scope
Audits the **harness assembly / SDK config** — where the prefix is built and the model called. **Out of scope:** a prose instruction file's content/attention (use `audit-instruction-file`;
its CE-9 already owns *volatile content in an always-loaded file's prefix* — do not re-flag it here);
single-turn / cold-start flows (no prefix to protect); local backends without cross-request KV reuse
(llama.cpp/Ollama — except the attribution-header case, PC-8/CLI); exfiltration/security architecture
(use `audit-lethal-trifecta`).

## Procedure
1. **Confirm the workload caches.** Multi-turn, hosted API, repeated prefix above the per-model minimum. If not → report "caching N/A here" and stop (PC-9; [When This Backfires](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/)).
   Done when a verdict is recorded: all three conditions confirmed (proceed) or the N/A report issued (stop).
2. **Locate the prefix boundary** — what is built once per session (`system`, `tools`) vs per call.
   Anything mutated before the current turn is a suspect.
   Done when every prompt layer is classified per-session or per-call — no blanks.
3. **Run the detectors** in [`checks.md`](checks.md) (PC-1…PC-11): prefix volatility, tool ordering,
   tool mutation, assembly order, per-call personalization, mid-session switch, monitoring, fleet
   per-machine context, TTL/idle-shape misfit (PC-10), cache-busting compaction (PC-11), economics. Load it when auditing.
   Done when every PC check has a recorded verdict — no blanks.
4. **Check the meter, not just the layout** — a harness that never reads `cache_read` vs
   `cache_creation` cannot detect a regression; flag the blind spot (PC-7).
   Done when the presence or absence of cache-metric reads is recorded.
5. **Recommend the cheapest deterministic fix per finding** (hoist out of loop, `sort_keys`,
   session-fixed model, `excludeDynamicSections`, `cache_control` on the static tail) and **estimate
   the bust frequency** (every call / on switch / per machine). Report with the template below.
   Done when every finding carries a fix, a bust frequency, and its resolved Fix → lesson link printed in the row, and the Prefix map and Findings are filled.

**Citations are printed in the report, not deferred.** Each Findings row's Fix → lesson cell must
carry the check's actual `learn.agentpatterns.ai` URL from `checks.md`, resolved by check ID right
there in the table — never left as a bare check ID or promised for the filed issue.

## Output template
```
# Prompt-cache hygiene audit — <target>

## Prefix map
| Layer (tools → system → messages) | Built | Stable across turns? |
|---|---|---|
| tools | per call, unsorted registry | NO — order varies |
| system | per call, datetime + user injected | NO — busts every call |
| messages | appended (full resend) | yes |

## Findings
| Severity | Check | Location | Busts | Deterministic fix | Fix → lesson |
|---|---|---|---|---|---|
| High | PC-2 tool order | call_model() L41 `load_tools_from_registry()` | every call | hoist + `sorted(..., key=name)` once per session | checks.md → PC-2 lesson URL |
| High | PC-5 per-call system | `build_system_prompt(user=...)` | every call | build once, no per-user injection | checks.md → PC-5 lesson URL |
| High | PC-1 volatile prefix | `f"...{datetime.now()}"` in system | every call | move timestamp to dynamic tail | checks.md → PC-1 lesson URL |
| Med | PC-7 no monitoring | no `usage` inspection | (silent) | log `cache_read` vs `cache_creation` (or provider equivalent) per session | checks.md → PC-7 lesson URL |

**Healthy baseline:** near-zero `cache_creation_input_tokens` after turn 1; a mid-session creation
spike = a prefix change.
**Smallest high-impact change:** <the one prefix mutation to remove first.>
```
Severity: **High** = prefix bust every call (full re-pay); **Medium** = per-event bust — per-switch /
per-machine / per-compaction (**PC-11**, the highest-cost single bust, at the moment the session is
longest) — or a silent monitoring gap; **Low** = economics/TTL tuning (PC-10) or least-cost cleanup.

## Related
- Sibling to **`audit-instruction-file`** — same family, different target: it audits instruction
  *prose* (and owns volatile-content-in-prefix via **CE-9**); this audits *harness code/config*.
  Boundary is reciprocal, written into that skill's Scope — do not overlap except at CE-9.
- Corpus: [dynamic-tool-fetching-cache-break](https://agentpatterns.ai/anti-patterns/dynamic-tool-fetching-cache-break/),
  [dynamic-system-prompt-composition](https://agentpatterns.ai/context-engineering/dynamic-system-prompt-composition/),
  [exclude-dynamic-system-prompt-sections](https://agentpatterns.ai/context-engineering/exclude-dynamic-system-prompt-sections/),
  [mid-session-config-cache-invalidators](https://agentpatterns.ai/anti-patterns/mid-session-config-cache-invalidators/).
- Auditor cluster of record: `prompt-cache-stability` (59 source-cited rules).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link, resolved by check ID via [`checks.md`](checks.md), **already printed in the Findings table row** (not deferred to filing time) — the filed issue body just copies that table.

## Critical rules (read last)
- **Exact prefix match or nothing** — one mutated byte before the current turn re-pays the whole
  prompt; the fix is layout/boundaries, not a prompt instruction.
- **Misses are silent** — always check `cache_read` vs `cache_creation` (or the provider's
  equivalent field); a creation spike mid-session is a prefix regression, not an error.
- **Guard the economics** — don't flag missing cache discipline on short/sub-minimum/mostly-dynamic
  workloads (PC-9). Read-only: recommend, apply nothing without confirmation.
