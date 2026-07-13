---
name: audit-lethal-trifecta
description: Audit an agent / sub-agent / MCP setup for the lethal trifecta — private-data access + untrusted-content exposure + external egress on one execution path — and flag the prompt-injection and exfiltration routes it opens. Invoke when reviewing or hardening an agent harness's security architecture (sub-agent tool allowlists, MCP servers, permissions, sandbox/egress config) or before deciding if granting an agent new tools, data, or web/network access is safe. Skip when the ask is literal secrets in context (use audit-secret-exposure), LLM-output sinks or agent-run installs (use audit-supply-chain-sinks), blast-radius/reversibility containment (use audit-harness-safety), retrieval-boundary authorization (use audit-memory-retrieval-integrity), instruction-file content/attention (use audit-instruction-file), or reviewing application source code for vulnerabilities (use a general security review).
user-invocable: true
version: "0.3.0"
usage: /audit-lethal-trifecta [path-to-agent-config-or-repo]
---

# Lethal Trifecta Audit

An execution path holding **all three** legs — private-data access + untrusted input + external
egress — is an exploitable exfiltration surface: LLMs can't separate trusted instructions from ones
injected via untrusted content, so injected input can steer tool calls to leak private data
([Willison 2025](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)). The fix is
architectural — **remove at least one leg from every path** — not a prompt asking the model to behave.

**Stance — detect and recommend; read-only; apply nothing without confirmation.**
- **Audit per execution path, not per agent.** One path with three "Yes" demands mitigation even when
  the agent looks safe in aggregate ([lethal-trifecta-threat-model](https://agentpatterns.ai/security/lethal-trifecta-threat-model/)).
- **Prefer removing egress, deterministically.** Most coding tasks need no network; a default-deny
  sandbox is a control the model can't override (Lockdown Mode caps egress with no AI in the loop)
  ([Willison 2026](https://simonwillison.net/2026/Jun/5/lockdown-mode/); [Harang/NVIDIA 2025](https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/)).
- **A natural-language rule is not enforcement.** "Don't exfiltrate" stops nothing — prompt guardrails
  are bypassed by injection ([defense-in-depth-agent-safety](https://agentpatterns.ai/security/defense-in-depth-agent-safety/); [Li et al. 2026, arXiv:2603.12230](https://arxiv.org/abs/2603.12230));
  recommend a sandbox / allowlist / deny rule that mechanically enforces leg removal.

## Input
- `path` (optional): agent-config dir or repo. Default: scan the harness — `.claude/` (agent/sub-agent
  defs, `settings.json` permissions, skills), `.mcp.json` / MCP config, `.cursor/rules`, `.github/`
  agent files, and any sandbox / egress config.

## Scope
Audits the **setup / architecture**: per-path data reach, tool allowlists, MCP servers, permission
modes, sandbox / egress controls. **Out of scope:** application source-code vulnerabilities (general
security review), instruction-prose attention / density (`audit-instruction-file`), live runtime
monitoring (static config audit).

## Procedure
1. **Enumerate paths.** Each agent/sub-agent is a path; record its tool allowlist, data reach, and
   egress surfaces. Sub-agents inherit nothing unless granted — audit each separately.
   Done when every agent/sub-agent has a recorded path row — no blanks.
2. **Classify the three legs per path** via detectors in [`checks.md`](checks.md): private data?
   untrusted input? egress? Mark **partial-leg** states (read-only egress, tokenized data) — do not
   collapse an ambiguous leg to a false "No" ([threat model, *When This Backfires*](https://agentpatterns.ai/security/lethal-trifecta-threat-model/)).
   Done when each path × leg is marked Yes/No/Partial.
3. **Flag every path holding all three** (or an ambiguous partial trifecta).
   Done when every path carries a TRIFECTA/safe verdict.
4. **Recommend the cheapest leg to remove** — egress-first for coding agents — preferring a
   deterministic control over a prompt mitigation (mitigation map in `checks.md`). Leg removal
   *migrates* risk, so name the new high-value target that must then be hardened. If the target
   already carries a prose-only safety line for this exact risk (e.g. "never exfiltrate secrets",
   "don't leak data") — name it explicitly and state it is not enforcement (per the Stance above),
   not just imply the deterministic fix is better.
   Done when every flagged path names the leg to remove, the target the risk migrates to, and any
   existing prose-only guardrail for that risk is named as non-enforcing.
5. **Report** with the template below.
   Done when the paths table, Findings, Safe paths, and Smallest high-impact change are filled.

## Detectors & attack routes
Per-leg detectors + injection/exfiltration route checks (poisoned dependency, cross-agent config
rewrite, MCP tool shadowing, cross-session memory/persistence) and the leg→mitigation map live in
[`checks.md`](checks.md) — load it when auditing. One seam to check explicitly: the trifecta can
complete **across sessions** — write attacker-influenced content to a persistent store now, read it
back as trusted instruction later; a per-session pass misses it (LT-A4).

## Output template
```
# Lethal-trifecta audit — <target>

## Execution paths
| Path (agent/sub-agent) | Private data | Untrusted input | Egress | Verdict |
|---|:--:|:--:|:--:|:--:|
| deploy-agent           | Yes (.env)   | Yes (repo cfg)  | Yes (push) | TRIFECTA |
| code-reviewer          | Yes          | Yes (PR diff)   | No        | safe     |
| research (web)         | No           | Yes (web)       | Yes       | safe     |

## Findings
| Severity | Path | Legs | Cheapest fix (deterministic) | Risk it migrates to |
|---|---|---|---|---|
| High | deploy-agent | PD+UI+EG | default-deny egress sandbox (`--network none`) | sandbox-escape — harden the boundary |

**Safe paths:** <which, and which leg keeps them safe.>
**Smallest high-impact change:** <the one leg to remove first.>
```
Severity: **High** = path with all three legs (or a config-write route that can grant them);
**Medium** = ambiguous partial trifecta; **Low** = least-privilege tightening with no live trifecta.

## Related / pairing
- Sibling to **`audit-instruction-file`** — that audits instruction *prose*; this audits security
  *architecture*. Same audit family, different target.
- Corpus: [prompt-injection-threat-model](https://agentpatterns.ai/security/prompt-injection-threat-model/),
  [defense-in-depth-agent-safety](https://agentpatterns.ai/security/defense-in-depth-agent-safety/),
  [dual-boundary-sandboxing](https://agentpatterns.ai/security/dual-boundary-sandboxing/),
  [url-exfiltration-guard](https://agentpatterns.ai/security/url-exfiltration-guard/).
- Mitigation patterns: [Beurer-Kellner et al. 2025, arXiv:2506.08837](https://arxiv.org/abs/2506.08837); [CaMeL, arXiv:2503.18813](https://arxiv.org/abs/2503.18813).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- A path with all three legs needs an **architectural** fix — a prompt instruction is not a fix.
- Audit **per execution path**; aggregate-safe agents hide single unsafe paths.
- Recommend **deterministic** controls (sandbox / allowlist / deny rule); read-only — apply nothing
  without confirmation.
