# audit-prompt-cache-hygiene — detectors

Loaded on demand. Map the prefix (tools → system → messages), then run each check on the
prompt-assembly code / SDK config. Each item: ID / **Flags** / **Why** (cited) / **Fix**. All
detection is read-only and static. PC-1…PC-8 are the core detector set; **PC-9 guards against false
positives** — flagging cache discipline where the economics say skip it.

> Remediation lessons (framing only, never a sole cite): `learn/context-engineering/caching-static-first`,
> `learn/context-engineering/assembling-the-prompt`, `learn/prompt-engineering/the-production-stack`.

---

### PC-1 — Volatile content in the static prefix
- **Flags:** timestamps, `cwd`, config, request IDs, or per-call personalization placed *before*
  stable content — e.g. `f"...{datetime.now()}"` or live config inlined into `system`.
- **Why:** mutating the prefix to convey state busts the cache on every call; variable state belongs
  in the dynamic tail ([caching discipline → Three Rules That Break Caching](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/);
  [static-content-first → What Breaks Cache Hits, *Prefix mutation*](https://agentpatterns.ai/context-engineering/static-content-first-caching/)).
- **Fix:** move volatile data to the dynamic tail (first user message / per-turn content); keep the
  prefix byte-identical across turns. Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

### PC-2 — Non-deterministic tool enumeration
- **Flags:** tool defs serialized in unstable order — unsorted registry, dict/key-order randomization,
  `json.dumps` without `sort_keys`, MCP tools listed in arrival order.
- **Why:** a different byte prefix every call → near-zero hit rate; the documented Codex MCP bug
  ([static-content-first → Non-deterministic tool enumeration](https://agentpatterns.ai/context-engineering/static-content-first-caching/);
  [dynamic-tool-fetching-cache-break → Non-Deterministic Serialization](https://agentpatterns.ai/anti-patterns/dynamic-tool-fetching-cache-break/)).
- **Fix:** enumerate/serialize tool defs in a deterministic order — `sorted(tools, key=lambda t: t["name"])`,
  `sort_keys=True`. Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

### PC-3 — Tool definitions mutated / rebuilt mid-session
- **Flags:** tools added, removed, or re-fetched per agent step (`load_tools()` inside the loop);
  tool list changing between calls.
- **Why:** tool defs sit at the top of the prefix (`tools → system → messages`); changing them
  invalidates the entire cache ([caching discipline → Adding or removing tools mid-session](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/);
  [dynamic-tool-fetching-cache-break → Why It Fails](https://agentpatterns.ai/anti-patterns/dynamic-tool-fetching-cache-break/)).
- **Fix:** fix a stable core tool set at session start; use `defer_loading: true` for rarely-used
  tools rather than swapping the list. Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

### PC-4 — Cache-hostile assembly order (dynamic content early)
- **Flags:** a conditionally-included or runtime section placed *early* in priority order; tool
  results left inside the cached region.
- **Why:** an early dynamic section invalidates the cache for every token after it — defeating the
  composition caching was meant to enable ([dynamic-system-prompt-composition → Cache invalidation
  from over-modulation](https://agentpatterns.ai/context-engineering/dynamic-system-prompt-composition/);
  [static-content-first → What Goes Where](https://agentpatterns.ai/context-engineering/static-content-first-caching/)).
- **Fix:** order static-first, dynamic-last; reserve conditional/runtime sections for the tail; be
  selective about caching volatile tool-result blocks. Remediation: [learn — assembling-the-prompt](https://learn.agentpatterns.ai/context-engineering/assembling-the-prompt/).

### PC-5 — Per-call mutation/personalization of the system prompt
- **Flags:** `system` built with per-user or per-call injection inside the call loop —
  `build_system_prompt(user=current_user)`.
- **Why:** system instructions cannot be personalized per-call — any change busts the prefix
  ([static-content-first → System instructions cannot be personalized per-call](https://agentpatterns.ai/context-engineering/static-content-first-caching/)).
- **Fix:** build `system` once outside the loop with no per-call injection; put per-user context in
  the dynamic tail. Remediation: [learn — the-production-stack](https://learn.agentpatterns.ai/prompt-engineering/the-production-stack/).

### PC-6 — Mid-session model / effort / MCP switch
- **Flags:** `model=` swapped mid-loop, effort/plan-mode toggled (opusplan), MCP servers added or
  reconnected partway through a session.
- **Why:** model-specific instructions live in the prefix; a swap is a full cache invalidation
  ([caching discipline → Switching models](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/);
  [static-content-first → Model switching](https://agentpatterns.ai/context-engineering/static-content-first-caching/);
  [mid-session-config-cache-invalidators](https://agentpatterns.ai/anti-patterns/mid-session-config-cache-invalidators/)).
- **Fix:** fix model/effort/MCP at session start; treat a switch as a context boundary; escalate via
  a subagent, not a parent model switch. Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

### PC-7 — No cache-health monitoring (silent miss)
- **Flags:** the harness never inspects `cache_read_input_tokens` vs `cache_creation_input_tokens`;
  no per-session hit-rate metric.
- **Why:** cache misses are silent — they bill full rate without erroring, so a mid-session prefix
  regression (the 12× SDK bug) is invisible without the meter ([caching discipline → Monitoring Cache
  Health; SDK Cache Invalidation](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/)).
- **Fix:** log `cache_read` / total per session; a healthy session shows near-zero
  `cache_creation_input_tokens` after turn 1 — alert on a mid-session creation spike. Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

### PC-8 — Per-machine context in the system prompt across a fleet
- **Flags:** `cwd` / OS / shell / memory paths inlined in the system prompt while the same harness
  runs across many machines / CI runners; `excludeDynamicSections` not set.
- **Why:** the per-session preset forks the prefix per machine/directory; the cache needs a 100%
  identical prefix ([exclude-dynamic-system-prompt-sections](https://agentpatterns.ai/context-engineering/exclude-dynamic-system-prompt-sections/)).
  (Local-inference variant: a varying `CLAUDE_CODE_ATTRIBUTION_HEADER` shifts the prefix and causes
  ~90% slower inference — set it to `0` ([kv-cache-invalidation-local-inference](https://agentpatterns.ai/context-engineering/kv-cache-invalidation-local-inference/)).)
- **Fix:** set `excludeDynamicSections: true` (SDK) / `--exclude-dynamic-system-prompt-sections`
  (CLI) so identical configs share one cache entry; keep per-machine context in the dynamic tail. Remediation: [learn — the-production-stack](https://learn.agentpatterns.ai/prompt-engineering/the-production-stack/).

### PC-9 — Cache-economics misfit *(false-positive guard — do NOT flag missing discipline here)*
- **Flags (suppress a finding when present):** prefix below the per-model cache minimum; ≤2-turn or
  short 5–10-call sessions paying the write premium without amortizing reads; a mostly-dynamic prefix
  that stabilizes for only a few turns; local backend with no cross-request KV reuse.
- **Why:** caching can lose money — the 25–100% write premium is paid repeatedly without enough reads
  to amortize it; for short sessions the optimization isn't worth it ([caching discipline → When This
  Backfires; Cache Economics](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/);
  [static-content-first → Tradeoffs](https://agentpatterns.ai/context-engineering/static-content-first-caching/)).
- **Fix:** report "caching N/A / not cost-effective here" instead of a cache-discipline finding;
  audit the hit-rate trace first — if reads don't dominate writes after a few turns, the cost isn't
  paid back. Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).

---

## Cache-safe compaction (when recommending history compression)
Naive compaction rebuilds the prompt and loses the cached prefix. **Fork instead:** keep the identical
prefix, append a summary of prior history as a *new* user message, then continue — the prefix cache
carries over ([caching discipline → Cache-Safe Forking for Compaction](https://agentpatterns.ai/context-engineering/prompt-caching-architectural-discipline/)).
Remediation: [learn — caching-static-first](https://learn.agentpatterns.ai/context-engineering/caching-static-first/).
