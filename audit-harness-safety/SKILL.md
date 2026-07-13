---
name: audit-harness-safety
description: Audit an agent harness's setup/architecture for blast-radius containment and recovery — OS-level sandbox + least-privilege scope, hard deny floors, reversibility/idempotency, a kill path, and consumption bounds — bounding the *damage* if a control fails. Invoke when hardening a harness for destructive-action safety — before granting write/Bash tools, running an agent headless or in CI, or when it touches the native filesystem, real credentials, shared branches, production, or paid APIs; also when reviewing reversibility/kill-path/cost posture. Skip when the question is data exfiltration or whether a web/network-access grant is safe — the lethal trifecta (use audit-lethal-trifecta), whether an LLM-emitted string or an agent-run package install — even phrased as a sandbox question — can be trusted at the sink (use audit-supply-chain-sinks), workflow-YAML script-injection/token/pin risks (use audit-github-actions-security), or instruction-prose attention/density (use audit-instruction-file).
user-invocable: true
version: "0.3.0"
usage: /audit-harness-safety [path-to-agent-config-or-repo]
---

# Audit Harness Safety

Anthropic frames agent risk as `risk = likelihood × damage` and bounds **damage** with sandboxes,
scoped permissions, and egress controls — uniformly, even against a misbehaving model
([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/) / [Anthropic — How we contain Claude](https://www.anthropic.com/engineering/how-we-contain-claude)).
`audit-lethal-trifecta` works the likelihood/exfiltration side; **this skill works the damage side**:
when a control fails (injection lands, model misjudges scope, or it loops) — *how much state can it
wreck, can you undo it, stop it, and what does it cost?* Least-privilege, sandboxing, reversibility,
kill-path, and consumption bounds all bound that damage term.

**Stance — detect and recommend; read-only; apply nothing without confirmation. Deterministic guardrails over prompt guardrails.**
- **The OS-level sandbox is the costliest, outermost boundary — it holds when judgment fails.** Tool
  access is enforced at runtime before the model sees a request, so an injection cannot invoke a tool
  never wired in; a prompt cannot ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)). Enforce filesystem *and* network at the OS layer — filesystem-only still allows network exfil, network-only allows startup-script escalation ([dual-boundary-sandboxing](https://agentpatterns.ai/security/dual-boundary-sandboxing/)).
- **Scope-as-prose is not a boundary.** "Don't touch prod" in CLAUDE.md stops nothing: identical
  Sonnet-4.6 weights swing 1.1%→27.7% overeager on framework alone — the deny rule is the floor the
  classifier can't talk past ([permission-framework-over-model](https://agentpatterns.ai/security/permission-framework-over-model/) / [Qu et al. 2026, arXiv:2605.18583](https://arxiv.org/abs/2605.18583)).
- **Bound radius *and* duration.** Scoping caps per-action damage, not time-integrated — 60% of orgs
  can't terminate a misbehaving agent. Pair scope with a runtime kill path the agent can't block ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)).

## Input
- `path` (optional): an agent-config dir or repo. Default: scan the harness — `.claude/` (agent &
  sub-agent `tools:` lists, `settings.json` `permissions.allow`/`deny`, `defaultMode`,
  `--dangerously-skip-permissions`, skills); sandbox/runtime config (Dockerfile/compose, `bwrap`,
  Seatbelt, `docker sbx`, `--network`, devcontainer); loop/cost config (`maxTurns`, iteration/time
  caps, budgets, circuit breakers); CI/workflow files (`.github/workflows/*` for deploy
  sequences and branch/PR strategy only — token `permissions:` → `audit-github-actions-security`).

## Scope
Audits the **enforced setup/architecture** for damage containment and recovery. **Out of scope:**
data-exfiltration trifecta analysis (use `audit-lethal-trifecta`); instruction-prose attention/density
(use `audit-instruction-file`); CI workflow-file security — script injection, token `permissions:`,
action pins (use `audit-github-actions-security`); application source-code vulnerabilities (general
security review); live runtime monitoring (static config audit).

## Procedure
1. **Enumerate execution paths** — each agent / sub-agent / CI job. Record its tools, file scope,
   permission mode, network reach, loop caps, and state-changing commands it can run. Sub-agents
   inherit nothing unless granted; a missing/wildcard `tools:` inherits **everything** — audit each.
   Done when every path has a recorded row for each attribute — no blanks.
2. **Run the detectors** in [`checks.md`](checks.md) per path: containment (HS-1–4), recovery
   (HS-5–7), duration/cost (HS-8–9), advisory (HS-10). Classify, don't assume — mark partial states.
   Done when every path × HS check is marked present/partial/absent.
3. **Score severity** from the Detectors table below — its Sev column is the canonical map.
   Done when every finding carries one severity.
4. **Recommend the deterministic control** — sandbox / allowlist / deny rule / human gate / `maxTurns`
   / budget — over any prompt mitigation. For a **headless/CI** path, never offer "ask-to-continue":
   it can't pause ([permission-framework-over-model, *When This Backfires*](https://agentpatterns.ai/security/permission-framework-over-model/)).
   Done when every finding names its deterministic control.
5. **Report** with the template below; link each fix to its remediation lesson.
   Done when the Paths table, Findings, Safe paths, and Smallest high-impact change are filled.

## Detectors (full Flags / Why / Fix in [`checks.md`](checks.md))
| ID | Flags (one line) | Sev |
|---|---|:--:|
| HS-1 | Write/Edit/Bash on native FS with no OS sandbox (or `bypassPermissions`/`--dangerously-skip-permissions` unsandboxed) | High |
| HS-2 | Authorization only in CLAUDE.md/system prompt — no `permissions.allow`/`deny` block | Med |
| HS-3 | No hard `deny` for `Bash(rm *)`/force-push/prod-deploy and secret files (`.env*`,`*.pem`,`*.key`,`id_rsa*`,`.aws/credentials`,`.ssh/**`,`*.bak`) | High |
| HS-4 | Missing/wildcard `tools:` (inherits all) on an agent/sub-agent (workflow-file token scope → `audit-github-actions-security`) | High |
| HS-5 | Irreversible side effect (send/charge/deploy/force-push/delete) with no human gate and not on a branch/draft | High |
| HS-6 | Non-idempotent state mutation in a re-runnable/headless path (no check-before-act / upsert / unique key) | Med |
| HS-7 | State-changing setup/CI commands with no snapshot/checkpoint to revert; verifies on exit code alone | Med |
| HS-8 | No `maxTurns`/iteration cap/circuit breaker/timeout — no kill path for a stuck agent | High |
| HS-9 | Paid LLM/tool calls with no per-call / per-task / fan-out / velocity / per-day bound — denial-of-wallet via an injection-reachable cost loop | Med |
| HS-10 | Write-capable multi-file/unfamiliar work with no read-only plan phase (advisory) | Low |

## Output template
```
# Harness-safety audit — <target>

## Paths
| Path | Sandbox (fs/net) | Scope source | Deny floor | Reversible | Kill path | Cost bound | Verdict |
|---|:--:|---|:--:|:--:|:--:|:--:|:--:|
| deploy-agent | none/none | CLAUDE.md prose | none | no (prod deploy) | none | none | UNSAFE |
| research-bot | yes/none  | allowlist []    | n/a  | yes (read-only)  | maxTurns | n/a | safe |

## Findings
| Sev | Path | Detector | What it can wreck | Deterministic fix | Remediation |
|---|---|---|---|---|---|
| High | deploy-agent | HS-1,3,5,8 | unsandboxed prod deploy + `rm`, no undo, no stop | `--network none` + container; `deny:[Bash(rm *),Edit(.env*)]`; gate deploy; `maxTurns` | sandboxing-and-blast-radius (via `checks.md`) |

**Safe paths:** <which, and which control keeps them safe.>
**Smallest high-impact change:** <the one control to add first — usually the OS sandbox or the deny floor.>
```

## Related / pairing
- **Reciprocal sibling: `audit-lethal-trifecta`** — opposite half of `risk = likelihood × damage`.
  Trifecta asks *can data leak?* (one path holding all three legs), recommends leg-removal to break an
  exfiltration path; this asks *if it errs, how much damage and can you recover?* and recommends the
  same controls to **cap blast radius and ensure recovery**. A write-capable single-leg agent is out
  of scope there, in scope here. If a path is **both** (unsandboxed *and* a full trifecta), run both
  skills — do not collapse them.
- Corpus: [sandbox-runtime-comparison](https://agentpatterns.ai/security/sandbox-runtime-comparison/),
  [rollback-first-design](https://agentpatterns.ai/agent-design/rollback-first-design/),
  [idempotent-agent-operations](https://agentpatterns.ai/agent-design/idempotent-agent-operations/),
  [unbounded-consumption-resource-bounds](https://agentpatterns.ai/security/unbounded-consumption-resource-bounds/).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. State all three of title, label, and body **explicitly in the offer text itself** — do not leave the label unstated even when it's the default. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- **The sandbox + deny floor are the boundary; a prompt/CLAUDE.md rule is documentation, not enforcement** — recommend the deterministic control.
- **Bound radius *and* duration** — scope caps per-action damage; a runtime kill path (`maxTurns`/breaker) caps time-integrated damage. A long-running or headless agent needs both.
- **Rank every action on undo cost; gate the irreversible** — external side effects have no rollback, so the gate is the only defense. Read-only: apply nothing without confirmation.
