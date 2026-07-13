---
name: audit-secret-exposure
description: Scan an agent's context surfaces — instruction files, skill files, harness config, tool defs, and credential-bearing files — for credentials that landed where the model can read them — hardcoded keys, secrets pasted into prompts/tool args, agent-readable .env/key files with no mechanical block, long-lived keys on harness/context surfaces where short-lived tokens exist (workflow-file OIDC posture — use audit-github-actions-security), URL query-string leak channels, secrets baked into skill examples, and control logic stored in a system prompt. Invoke when granting or hardening an agent's repo/harness and you need a deterministic literal-secret content scan. Skip when auditing whether private-data + untrusted-input + egress co-occur on one execution path (use audit-lethal-trifecta) or auditing dependency/install sinks (use audit-supply-chain-sinks).
user-invocable: true
version: "0.3.0"
usage: /audit-secret-exposure [path-to-agent-config-or-repo]
---

# Audit Secret Exposure

Once a secret enters the model's context window you lose control of it — it is copied into the model
API request, session logs, and any file or comment the agent writes next
([secrets-management-for-agents](https://agentpatterns.ai/security/secrets-management-for-agents/)).
This audit is a **deterministic content scan** of every *agent-readable surface* for credentials
literally present, or long-lived keys used where a short-lived token exists — with **one declared
judgment-tier exception**: SE-5's URL data-flow check (see `checks.md`). Run it **first** when
hardening a harness: a high-severity finding halts the rest of the assessment until the secret is
**rotated** (exposure ≠ removal — assume it already leaked).

**Stance — detect, redact, and recommend; read-only; apply nothing.**
- **Redact every literal.** Report `sk-ant-…` / `ghp_…` shape and location, never the full value —
  echoing it re-exposes it ([secrets-management-for-agents](https://agentpatterns.ai/security/secrets-management-for-agents/)).
- **A found secret is compromised — rotate, don't just delete.** Already in git history, publication,
  and logs ([credential-hygiene-agent-skills](https://agentpatterns.ai/security/credential-hygiene-agent-skills/)).
- **Mechanical enforcement, not advisory prose.** "Don't read `.env`" stops nothing —
  instruction-following degrades multiplicatively under many rules. Recommend `permissions.deny` + a
  `PreToolUse` Read hook ([protecting-sensitive-files](https://agentpatterns.ai/security/protecting-sensitive-files/)).

## Input
- `path` (optional): an agent-config dir or repo. Default: scan the agent-readable surface —
  `CLAUDE.md`/`AGENTS.md`, system & sub-agent prompts, `.claude/skills/**/SKILL.md` & `skills/**`,
  `.claude/settings.json`/`settings.local.json`, `.mcp.json`/MCP config, `.cursor/rules`, tool
  definitions & example tool-calls, code comments, and any credential-bearing file the agent can read
  (`.env*`, `~/.aws/credentials`, `*.pem`/`*.key`, `service-account.json`, `secrets/**`).

## Scope
Audits **content**: is a credential literally in a surface the model reads, or a long-lived key used
where a short-lived one exists? **Out of scope:** whether private-data + untrusted-input + egress
co-occur on one path (architecture — use `audit-lethal-trifecta`); dependency/install poisoning
(supply-chain); generic CI/CD workflow secret posture — cloud-deploy OIDC, workflow hardening (use
`audit-github-actions-security`; SE-4 keeps the agent's own model-API credential wherever it lives);
app source-code vulnerabilities (general security review). They touch only at the
exfil edge — a leaked secret is the *payload* a trifecta exfiltrates.

## Procedure
1. **Enumerate surfaces** (the Input list). Treat every file the agent can read as in-context.
   Done when every Input surface is listed — no blanks.
2. **Run the detectors** in [`checks.md`](checks.md) — credential-format regex + the gated
   Shannon-entropy fallback (thresholds in `checks.md`) + config-presence (deny rules / hooks / WIF).
   Done when every surface has a per-detector verdict recorded.
3. **Redact and severity-rank** each hit (table below). A live literal or an unblocked credential
   file is **High** — halt and flag for rotation before continuing.
   Done when every hit is redacted and carries a severity.
4. **Recommend the deterministic fix** per finding (env injection / wrapper script / `permissions.deny`
   + `PreToolUse` hook / placeholder + skill-dir scanning / short-lived WIF token / externalise from
   the prompt). Prefer the control the model cannot override.
   Done when every finding names one deterministic fix.
5. **Report** with the template below. **Hard requirement:** every finding's Why→corpus and
   Fix→lesson citations must be resolved and printed in the report itself (Citations section,
   below) — never deferred to "resolved when filed."
   Done when the filled template covers every finding and the Citations section resolves every
   check ID used, by real URL, from `checks.md`.

## Detectors
The seven detectors (SE-1…SE-7) — flags, cited why, and the deterministic fix each maps to — plus the
credential-format regex table and an optional auditor-only recon check live in
[`checks.md`](checks.md). Load it when auditing.

## Output template
```
# Secret-exposure audit — <target>

## Findings
| Sev  | Check | Surface (file:loc)            | What (redacted)            | Deterministic fix |
|------|-------|-------------------------------|----------------------------|-------------------|
| High | SE-1  | CLAUDE.md:14                  | OPENAI_API_KEY="sk-…2 chars shown" | env-inject; ROTATE now |
| High | SE-3  | .env (no deny rule)           | agent-readable secrets file | permissions.deny + PreToolUse hook |
| High | SE-6  | skills/deploy/SKILL.md:22     | curl -H "Authorization: Bearer ghp_…" | $VAR placeholder + wrapper; scan skill dirs; ROTATE |
| Med  | SE-4  | ci.yml:8                      | static ANTHROPIC_API_KEY where OIDC exists | short-lived WIF token; UNSET don't blank |

## Citations
One row per check ID used above — required in every report, not only in the filed issue.
| Check | Why → corpus | Fix → lesson |
|-------|--------------|--------------|
| SE-1  | [secrets-management-for-agents](https://agentpatterns.ai/security/secrets-management-for-agents/) | checks.md → SE-1 lesson URL |
| SE-3  | [protecting-sensitive-files](https://agentpatterns.ai/security/protecting-sensitive-files/) | checks.md → SE-3 lesson URL |
| SE-6  | [credential-hygiene-agent-skills](https://agentpatterns.ai/security/credential-hygiene-agent-skills/) | checks.md → SE-6 lesson URL |
| SE-4  | [workload-identity-federation-for-agents](https://agentpatterns.ai/security/workload-identity-federation-for-agents/) | checks.md → SE-4 lesson URL |

**Halt-and-rotate:** <every High literal — already in logs/history; rotate, don't just delete.>
**Clean surfaces:** <which, and the control that keeps them clean — env injection / deny rule / placeholders.>
**Smallest high-impact change:** <the one control to add first.>
```
Severity: **High** = a live credential literal in an agent-readable surface (SE-1/2/6) or an
agent-readable credential file with no mechanical block (SE-3). **Medium** = long-lived key where a
short-lived one exists (SE-4), a URL query-string leak channel (SE-5), or control logic in a system
prompt (SE-7). **Low** = hardening with no live secret (e.g. add skill-dir scanning pre-emptively).

## Related / pairing
- Sibling to **`audit-lethal-trifecta`** — it audits security *architecture* (legs co-located on a
  path); this audits *content* (a secret literally in a readable surface). Same family, different
  target; reciprocal Skip clauses.
- Corpus: [secrets-management-for-agents](https://agentpatterns.ai/security/secrets-management-for-agents/),
  [protecting-sensitive-files](https://agentpatterns.ai/security/protecting-sensitive-files/),
  [workload-identity-federation-for-agents](https://agentpatterns.ai/security/workload-identity-federation-for-agents/),
  [credential-hygiene-agent-skills](https://agentpatterns.ai/security/credential-hygiene-agent-skills/),
  [url-exfiltration-guard](https://agentpatterns.ai/security/url-exfiltration-guard/),
  [system-prompt-not-a-secret-store](https://agentpatterns.ai/security/system-prompt-not-a-secret-store/).
- Remediation lessons: `keep-the-keys-out`, `keys-that-expire-in-minutes`, `the-url-is-the-leak`.

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- **A secret found in context is already compromised — rotate it, don't just delete it**, and never
  echo the literal value (redact).
- **Enforce mechanically, not by prose** — `permissions.deny` + a `PreToolUse` hook over an advisory
  "don't read `.env`"; gitignore gates *discovery*, not explicit reads, so it is not sufficient.
- Read-only: detect and recommend; apply nothing without confirmation.
