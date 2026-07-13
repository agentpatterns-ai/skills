# audit-secret-exposure — detectors

Loaded on demand by `audit-secret-exposure`. Scan each agent-readable surface with these detectors.
Each item: ID / **Flags** / **Why** (cited) / **Fix** (deterministic). All detection is read-only;
**redact every literal** in output — report the shape and location, not the value.

---

## Credential-format regex (SE-1 / SE-6 matchers)

| Class | Pattern (illustrative) |
|---|---|
| Anthropic / OpenAI key | `sk-ant-[A-Za-z0-9_-]+`, `sk-proj-[A-Za-z0-9_-]+`, `sk-[A-Za-z0-9]{20,}` |
| Stripe-style live key | `sk_live_…` / `sk-live-…` (both separators occur in the wild) |
| GitHub token | `ghp_[A-Za-z0-9]{36}`, `gho_…`, `ghs_…` |
| AWS access key | `AKIA[0-9A-Z]{16}` |
| Bearer / basic in a request | `Authorization: Bearer <token>`, `https://user:pass@host/` |
| Credential-shaped key=value | `api_key:`, `*_API_KEY=`, `*_TOKEN=`, `password:` followed by a literal |
| Env-var default leak | `API_KEY=${API_KEY:-sk-live-xyz}` |
| High-entropy fallback | Shannon entropy ≥ 4.0 **and** length ≥ 20, sitting next to a credential-shaped key |

Entropy alone over-fires; gate it on proximity to a credential-shaped key. A placeholder
(`$VAR`, `<token>`, `sk-…`) is **not** a finding — it signals substitution is required.

---

## Detectors

### SE-1 — Hardcoded credential literal in an agent-readable surface
- **Flags:** a credential-format match or gated high-entropy string in `CLAUDE.md`/`AGENTS.md`, a
  system/sub-agent prompt, a `SKILL.md`, `.mcp.json`/`settings.json`, or a code comment the agent reads.
- **Why:** once a secret enters the context window you lose control of it; it is copied into the API
  request, session logs, and generated files ([secrets-management-for-agents — *Never Store Secrets in
  Agent-Readable Files*](https://agentpatterns.ai/security/secrets-management-for-agents/)).
- **Fix:** remove and **rotate** (assume leaked); inject as an env var or behind a wrapper script.
  Remediation: [learn — keep-the-keys-out](https://learn.agentpatterns.ai/security/keep-the-keys-out/). **Severity High.**

### SE-2 — Secret supplied via prompt / tool-call argument (not env-or-wrapper)
- **Flags:** a key pasted into a prompt, or passed as a literal tool-call argument, instead of
  injected as an env var before process start or hidden behind a wrapper that accepts intent and
  returns results.
- **Why:** pasting a key into a prompt sends it to the model API and risks the agent echoing it back;
  agents need results, not credentials ([secrets-management-for-agents — *Environment Variable
  Injection* / *Wrapper Scripts*](https://agentpatterns.ai/security/secrets-management-for-agents/)).
- **Fix:** env injection (`direnv`/`.envrc` or shell pre-load) or a wrapper script
  (`scripts/query-db.sh` reads `$DATABASE_URL`, returns only output). Remediation:
  [learn — keep-the-keys-out](https://learn.agentpatterns.ai/security/keep-the-keys-out/).
  **Severity High** (live literal) / Medium (pattern with placeholder).

### SE-3 — Credential-bearing file is agent-readable (no mechanical block)
- **Flags:** `.env*`, `~/.aws/credentials`, `*.pem`/`*.key`, `service-account.json`, `secrets/**`
  present with **no** `permissions.deny` entry **and no** `PreToolUse` Read hook; or reliance on
  `respectGitignore` alone.
- **Why:** agents read whatever they encounter; advisory "don't read `.env`" is unreliable
  (instruction-following degrades multiplicatively). `permissions.deny` has filed enforcement-failure
  bugs (treat as first layer), and gitignore gates *discovery* — an explicit read still succeeds
  ([protecting-sensitive-files](https://agentpatterns.ai/security/protecting-sensitive-files/)).
- **Fix:** `permissions.deny` on the paths **plus** a `PreToolUse` Read hook (the more reliable layer)
  — or the target harness's equivalent deny mechanism for non-Claude setups. *Neither stops a spawned
  subprocess* reading the file (`cat .env` via Bash): the cited page's sandbox-layer control
  (`sandbox.credentials`) closes it **for sandboxed subprocesses — it requires sandbox mode on**
  (pair with the harness's OS sandbox); with sandboxing off, mark the surface partially open.
  Verify the agent cannot echo the value. Remediation:
  [learn — keep-the-keys-out](https://learn.agentpatterns.ai/security/keep-the-keys-out/). **Severity High.**

### SE-4 — Long-lived / static credential where a short-lived token is available
- **Flags:** a static `sk-ant-…` or long-lived PAT in CI secrets / container env / shell profile with
  no OIDC federation, when an ambient workload identity exists. Edge: a leftover `ANTHROPIC_API_KEY`
  (even `=""`) silently **shadows** federation.
- **Why:** a static API key is the highest-blast-radius credential — leakable from logs, hooks,
  transcripts, with rotation that never matches incident timelines; WIF removes the key for a
  short-lived OIDC token. `ANTHROPIC_API_KEY` outranks the federation env vars, and `=""` still wins —
  **unset, do not blank** ([workload-identity-federation-for-agents](https://agentpatterns.ai/security/workload-identity-federation-for-agents/)).
- **Fix:** WIF / OIDC-federated short-lived token; **unset** any leftover key (don't blank); confirm
  with `ant auth status`. *Seam:* SE-4 covers the **agent's own model-API credential** wherever it
  lives; the generic workflow-secrets-vs-OIDC posture of a CI pipeline is
  `audit-github-actions-security`'s lane. Remediation:
  [learn — keys-that-expire-in-minutes](https://learn.agentpatterns.ai/security/keys-that-expire-in-minutes/). **Severity Medium.**

### SE-5 — Secret / user data carried in a URL (query-string exfiltration channel) *(judgment-tier)*
- **Flags (config-observable):** redirect-following left on, embedded resources auto-fetched, with a
  tool/skill that constructs or follows URLs built from context. Whether credentials or user-specific
  data are *placeable* in the query string is a data-flow judgment — this is the catalog's one
  declared judgment-tier check, not part of the deterministic regex/entropy/config scan.
- **Why:** the URL is itself a data channel — the leak is in the *request*, before any response is
  read; redirect chains bypass a domain allow-list (trusted domain → attacker domain), so an
  allow-list answers the wrong question ([url-exfiltration-guard](https://agentpatterns.ai/security/url-exfiltration-guard/)).
- **Fix:** `follow_redirects=False` and re-check the redirect target; gate on a public-web index
  property (was this URL independently observable with no user data?), not a domain allow-list
  ([url-fetch-public-index-gate](https://agentpatterns.ai/security/url-fetch-public-index-gate/)).
  Remediation: [learn — the-url-is-the-leak](https://learn.agentpatterns.ai/security/the-url-is-the-leak/). **Severity Medium.**

### SE-6 — Inline credential in a skill example / endpoint; skill dirs uncovered by scanning
- **Flags:** `curl -H "Authorization: Bearer ghp_…"`, `api_key: sk-live-…`,
  `https://user:pass@host/`, or `API_KEY=${API_KEY:-sk-live-xyz}` defaults inside a `SKILL.md`; **and**
  no secret-scanner path rule of **any** scanner (e.g. `betterleaks`/`gitleaks`/`trufflehog`/
  `detect-secrets`) covering `.claude/skills/`, `skills/` — the requirement is coverage, not a brand.
- **Restraint:** a missing scanner-coverage rule alone, with no literal secret-shaped value in any
  skill example, is **not** an SE-6 finding — never report "no scanner config visible" as a finding
  by itself. SE-6 fires on the literal; the scanning-gap note only attaches to a literal already found.
- **Why:** skills are reusable artifacts that ship a working example from the author's environment;
  they leak via publication, git history, and verbatim reproduction — a runtime-only secrets control
  does not cover the file itself ([credential-hygiene-agent-skills](https://agentpatterns.ai/security/credential-hygiene-agent-skills/)).
- **Fix:** replace with `$VAR` / `<token>` placeholders; route the skill through a wrapper script that
  reads the secret from the env; **extend secret-scanning to skill directories** — a fast edge scanner
  ([`betterleaks`](https://github.com/betterleaks/betterleaks), the maintained successor to the now
  feature-complete `gitleaks`) at commit/CI, plus `trufflehog` over full history; rotate the leaked
  value. Remediation: [learn — keep-the-keys-out](https://learn.agentpatterns.ai/security/keep-the-keys-out/).
  **Severity High** (live literal) / Low (scanning gap only).

### SE-7 — System prompt used as a secret / control store (OWASP LLM07)
- **Flags:** credentials/connection strings, internal business rules (transaction/loan limits),
  filtering criteria, or role/permission tables placed in a system prompt and relied on as confidential.
- **Enumerate every instance:** when one cited block carries multiple distinct control-logic-in-prompt
  items (e.g. several dollar-limit rules, a permission-tier flag, a cross-user-refusal/authorization
  rule), the finding must name **all** of them, not just the one or two most obvious — a partial list
  under-reports the exposure even when the line range cited is correct.
- **Why:** the system prompt is recoverable input — there is no privilege boundary inside the context
  window; multi-turn extraction reaches 84–92% ASR and a tool-invocation channel recovered it at 0.997
  similarity. The bug is the design choice to trust prompt confidentiality, not the leak that follows
  ([system-prompt-not-a-secret-store — OWASP LLM07:2025](https://agentpatterns.ai/security/system-prompt-not-a-secret-store/)).
- **Fix:** externalise — credentials to env-injected/tool-resolved storage; limits and authorization
  to deterministic checks in the API/tool layer, enforced independent of the prompt. Lesson-thin;
  closest remediation: [learn — keep-the-keys-out](https://learn.agentpatterns.ai/security/keep-the-keys-out/)
  ("never paste a key into a prompt"). **Severity Medium.**

---

### SE-8 — Internal hostname / staging URL disclosure (optional, auditor-only)
- **Flags:** internal hostnames / staging URLs / private subdomains leaked into instruction files
  (`.local`, `.internal`, `.corp`, `staging-*`, `dev-*`, `jumpbox`, `bastion`).
- **Why:** a hostname alone grants no access, but it names a target — the same naming patterns are
  literal entries in mainstream reconnaissance wordlists, and exposed repos are documented to leak
  "infrastructure intel" (hostnames, IPs, ports) as a category distinct from credentials
  ([internal-hostname-disclosure-agent-context](https://agentpatterns.ai/security/internal-hostname-disclosure-agent-context/)).
- **Fix:** extend the credential-format regex sweep to internal-naming patterns; replace a real
  hostname in a committed instruction file with a placeholder and inject it at runtime. Remediation:
  [learn — keep-the-keys-out](https://learn.agentpatterns.ai/security/keep-the-keys-out/). **Severity Low** — informational, not a normative High/Medium finding.

---

## Severity → fix summary
| Sev | Triggers | Deterministic control |
|---|---|---|
| High | SE-1, SE-2 (literal), SE-3, SE-6 (literal) | env injection / wrapper / `permissions.deny` + `PreToolUse` hook / placeholders — **and rotate** |
| Medium | SE-4, SE-5, SE-7, SE-2 (pattern) | short-lived WIF token / `follow_redirects=False` + index gate / externalise from prompt |
| Low | SE-6 (scanning gap), SE-8 (recon note) | extend secret-scanning to skill dirs; informational recon note |

A High finding **halts** the rest of an agent-readiness assessment until the secret is rotated
(auditor `secrets-and-credentials` cluster: secret-exposure is the first audit to run).
