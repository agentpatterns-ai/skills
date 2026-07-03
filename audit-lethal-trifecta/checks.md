# audit-lethal-trifecta — detectors & attack routes

Loaded on demand by `audit-lethal-trifecta`. Classify each execution path on the three legs, then run
the attack-route checks. Each item: ID / **Flags** / **Why** (cited) / **Fix**. All detection is
read-only.

---

## Leg detectors — classify each path

### LT-PD — Private-data access
- **Flags:** the path can reach secrets, credentials, PII, or proprietary code — readable `.env` /
  `.dev.vars` / key files, DB connection tools, broad `Read`/filesystem scope over internal repos,
  injected API tokens.
- **Why:** private data is the payload an exfiltration chain steals ([threat model](https://agentpatterns.ai/security/lethal-trifecta-threat-model/)).
- **Fix (if this is the leg to remove):** exclude secret paths from agent-readable scope; tokenize
  PII (resolved in a trusted executor); inject short-lived, minimal-permission credentials at runtime
  ([secrets-management-for-agents](https://agentpatterns.ai/security/secrets-management-for-agents/),
  [pii-tokenization-in-agent-context](https://agentpatterns.ai/security/pii-tokenization-in-agent-context/)).
  Remediation: [learn — the-lethal-trifecta](https://learn.agentpatterns.ai/security/the-lethal-trifecta/).
- **Partial leg:** tokenized data is *not* full private-data access — but the token resolver becomes
  the new target. Mark partial, don't zero it.

### LT-UI — Untrusted input
- **Flags:** the path ingests content it did not author — `WebFetch` / web search, fetched pages,
  GitHub issue / PR / comment text, installed dependencies, file uploads, or outputs from external
  MCP servers.
- **Why:** LLMs can't separate injected from trusted instructions; untrusted input in context can
  steer tool calls ([prompt-injection-threat-model](https://agentpatterns.ai/security/prompt-injection-threat-model/)).
- **Fix (if this is the leg to remove):** restrict the path to operator-controlled content; or apply
  a pattern that quarantines untrusted content — Dual-LLM, Action-Selector, Plan-Then-Execute,
  Context-Minimization, Code-Then-Execute ([Beurer-Kellner 2025, arXiv:2506.08837](https://arxiv.org/abs/2506.08837)).
  Remediation: [learn — the-lethal-trifecta](https://learn.agentpatterns.ai/security/the-lethal-trifecta/).
- **Partial leg:** content from a trusted-but-fetched source is lower-risk than open web — note it.

### LT-EG — External egress
- **Flags:** the path can send data outside the sandbox — HTTP tools, `WebFetch` to broad domains,
  `Bash(curl|wget)`, `git push`, MCP servers with outbound calls, or a container without network
  restriction (`--network` not `none`).
- **Why:** egress is the exfiltration channel; removing it closes the path even when the other two
  legs remain ([threat model, key takeaways](https://agentpatterns.ai/security/lethal-trifecta-threat-model/)).
- **Fix (preferred — cheapest for coding agents):** default-deny outbound with an explicit allowlist;
  a deterministic sandbox the model can't override ([dual-boundary-sandboxing](https://agentpatterns.ai/security/dual-boundary-sandboxing/),
  [agent-network-egress-policy](https://agentpatterns.ai/security/agent-network-egress-policy/)).
  Remediation: [learn — the-lethal-trifecta](https://learn.agentpatterns.ai/security/the-lethal-trifecta/).
- **Partial leg:** "read-only" fetch with no POST/PUT and a strict domain allowlist is reduced, not
  removed — a TLS/hostname blind spot can still leak ([url-exfiltration-guard](https://agentpatterns.ai/security/url-exfiltration-guard/)). Mark partial.

> A path is a **trifecta** when LT-PD ∧ LT-UI ∧ LT-EG are all present (or ambiguously partial).
> Audit each agent and sub-agent as its own path — sub-agents inherit nothing unless granted.

---

## Attack-route checks — paths that *acquire* a leg

Default severity (the SKILL severity guide's own bands): **High** when the route completes a live
trifecta or is a config-write grant route; **Medium** when it leaves an ambiguous partial trifecta;
**Low** when it is least-privilege tightening with no live trifecta.

### LT-A1 — Poisoned dependency
- **Flags:** a path that can read untrusted package names (issues, deps) **and** install them **and**
  reach private data — install becomes egress, the package exfiltrates env vars.
- **Why:** documented chain: agent reads an issue naming a malicious package, installs it, it leaks
  secrets ([slopsquatting-hallucinated-package-names](https://agentpatterns.ai/security/slopsquatting-hallucinated-package-names/);
  [NVIDIA/Lynch 2025](https://developer.nvidia.com/blog/from-assistant-to-adversary-exploiting-agentic-ai-developer-tools/)).
- **Fix:** remove install-time egress; pin/allowlist dependencies; sandbox the install. Remediation:
  [learn — the-package-that-doesnt-exist](https://learn.agentpatterns.ai/security/the-package-that-doesnt-exist/).

### LT-A2 — Cross-agent privilege escalation
- **Flags:** any agent can **write executable config** another agent loads — `CLAUDE.md`/`AGENTS.md`,
  `.claude/settings.json`, `.cursor/rules`, MCP config — so it can drop a sibling's sandbox and grant
  all three legs.
- **Why:** one agent rewriting another's constraints is a documented escalation ([Embrace The Red 2025](https://embracethered.com/blog/posts/2025/cross-agent-privilege-escalation-agents-that-free-each-other/)).
- **Fix:** gate agent writes to executable config behind human approval / make it read-only
  ([gate-agent-writes-to-executable-config](https://agentpatterns.ai/security/gate-agent-writes-to-executable-config/)). Remediation:
  [learn — bound-the-blast-radius](https://learn.agentpatterns.ai/security/bound-the-blast-radius/).

### LT-A3 — MCP tool shadowing / poisoning
- **Flags:** an untrusted/third-party MCP server alongside private-data access and egress — it can
  shadow a trusted tool, read private context, and forward it out.
- **Why:** malicious MCP servers shadowing trusted tools is a documented exfil route ([mcp-runtime-control-plane](https://agentpatterns.ai/security/mcp-runtime-control-plane/);
  [Invariant Labs 2025](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)).
- **Fix:** restrict MCP server egress; vet/allowlist servers; isolate untrusted servers from
  private-data paths. Remediation: [learn — the-gateway-in-the-middle](https://learn.agentpatterns.ai/security/the-gateway-in-the-middle/).

### LT-A4 — Cross-session persistence (deferred payload / Trojan Hippo)
- **Flags:** a path that **auto-writes untrusted/attacker-influenced content into a persistent,
  cross-session store** — agent long-term memory, a written file, a tracked issue/comment, a RAG /
  vector DB, a team `CLAUDE.md` — that a **later** run reads back and treats as trusted instruction
  *without re-establishing provenance*. The three legs are **split across sessions/time**, not
  co-located: Session 1 = untrusted input + memory-write; Session N = private data + egress. A
  per-session audit passes each session and misses the pivot, so check the **write→read seam**, not
  just single-run paths. Especially acute when a memory backend auto-ingests tool returns / scraped
  text as "facts" or "preferences" with no provenance.
- **Why:** memory extends the LLM's inability to separate injected from trusted instructions *across*
  sessions; retrieved memory enters context with the same authority as live user input, so a dormant
  payload can survive 100+ benign sessions and activate on a sensitive topic to exfiltrate. The
  attack composes the lethal trifecta across two sessions — single-session injection resistance does
  not transfer ([trojan-hippo-memory-attack, *Cross-Session Lethal Trifecta*](https://agentpatterns.ai/security/trojan-hippo-memory-attack/)).
  *Seam:* LT-A4 owns the **cross-session write→read trifecta seam**; auditing the memory store itself
  (write guards, provenance tiers, retrieval filtering) is `audit-memory-retrieval-integrity` (MR-I3)
  — hand the store audit off.
- **Fix (deterministic, leg-removal):** only user-approved memory writes — never auto-write tool
  returns / fetched content (`deny_sources: email_body, web_fetch_content, mcp_tool_return`); or
  human-curated, version-controlled memory (PR-reviewed). Compose with default-deny egress and PII
  tokenization so an activated payload has no channel and no payload — match defenses to the task
  distribution, no single layer suffices ([trojan-hippo-memory-attack, *Defenses*](https://agentpatterns.ai/security/trojan-hippo-memory-attack/); hands-on: [the-payload-that-waits](https://learn.agentpatterns.ai/security/the-payload-that-waits/)).

---

## Leg → mitigation map (for recommendations)

| Pattern | Leg removed | Mechanism |
|---|---|---|
| Default-deny egress sandbox | egress | model-independent network block (preferred for coding agents) |
| Dual-LLM / Action-Selector / Plan-Then-Execute / Context-Min / Code-Then-Execute | untrusted input | quarantine or fix the action set before untrusted content is trusted |
| PII tokenization / scoped credentials / file exclusion | private data | strip sensitive data before it enters context |
| LLM Map-Reduce | private data | each instance sees only a partition |

Source: [threat model pattern table](https://agentpatterns.ai/security/lethal-trifecta-threat-model/); [Beurer-Kellner 2025](https://arxiv.org/abs/2506.08837); [CaMeL 2025](https://arxiv.org/abs/2503.18813).

## Sandbox controls checklist (when recommending hardening)
- **Egress:** default-deny + explicit allowlist.
- **Filesystem:** block writes outside the workspace.
- **Config protection:** prevent modification of `CLAUDE.md` / `.cursorrules` / MCP configs (LT-A2).
- **Secrets:** short-lived, minimal-permission injected tokens (LT-PD).

Source: [Harang/NVIDIA 2025](https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/).
