# audit-harness-safety — detectors

Loaded on demand by `audit-harness-safety`. Run per execution path. Each item: ID / **Flags** /
**Why** (cited) / **Fix** (deterministic, with remediation lesson). All detection is read-only.
Group A = containment (cap the radius), B = recovery (undo the damage), C = duration/cost (stop it).

## Contents
- [A — Containment (HS-1…HS-4)](#a--containment-cap-the-blast-radius)
- [B — Recovery (HS-5…HS-7)](#b--recovery-undo-the-damage)
- [C — Duration & cost (HS-8…HS-9)](#c--duration--cost-stop-it)
- [Advisory (HS-10)](#advisory)

---

## A — Containment: cap the blast radius

### HS-1 — Unsandboxed write/exec agent
- **Flags:** a path with `Write`/`Edit`/`Bash` on the native filesystem and **no** OS-level boundary
  (no `--network none` + container/microVM, no bubblewrap/Seatbelt); or `bypassPermissions` /
  `--dangerously-skip-permissions` / `defaultMode: bypassPermissions` outside a sandbox.
- **Why:** tool access is enforced at the runtime layer before the model sees a request, so isolation
  is structural, not probabilistic — an injection can't invoke a tool the sandbox blocks, a prompt
  can't ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)).
  Filesystem-only containment still allows network exfil; network-only allows startup-script privilege
  escalation — enforce **both** at the OS level ([dual-boundary-sandboxing](https://agentpatterns.ai/security/dual-boundary-sandboxing/)).
- **Fix:** add an OS sandbox matched to the threat — microVM for untrusted/multi-tenant, container or
  bubblewrap/Seatbelt for trusted single-host ([sandbox-runtime-comparison](https://agentpatterns.ai/security/sandbox-runtime-comparison/)).
  Remediation: [sandboxing-and-blast-radius](https://learn.agentpatterns.ai/harness-engineering/sandboxing-and-blast-radius/), [pick-your-sandbox](https://learn.agentpatterns.ai/security/pick-your-sandbox/).

### HS-2 — Scope encoded as prose, not allow/deny rules
- **Flags:** authorization expressed only in CLAUDE.md / system prompt ("do not delete files outside
  `src/`", "don't touch prod") with no `permissions.allow` / `deny` block.
- **Why:** the permission framework drives overeager-action rates more than the model — identical
  Sonnet-4.6 weights span 1.1%→27.7% across harnesses; stripping the scope block moves the rate
  11.9–17.2 pts. A prompt is not a boundary ([permission-framework-over-model](https://agentpatterns.ai/security/permission-framework-over-model/) / [Qu et al. 2026, arXiv:2605.18583](https://arxiv.org/abs/2605.18583)).
- **Fix:** encode scope as a deterministic narrow `permissions.allow` allowlist (or an ask-to-continue
  checkpoint — **not** for headless paths, which can't pause; headless per the evidence bar, tail
  note). Remediation:
  [permissions-and-safety-boundaries](https://learn.agentpatterns.ai/harness-engineering/permissions-and-safety-boundaries/), [the-framework-is-the-knob](https://learn.agentpatterns.ai/security/the-framework-is-the-knob/).

### HS-3 — No hard deny under destructive / secret surfaces
- **Flags:** missing `deny` rules for `Bash(rm *)`, force-push, prod deploy, and `Edit`/`Read` of
  `.env*`, `*.pem`, `*.key`, `id_rsa*`, `.aws/credentials`, `.ssh/**`, `*.bak`.
- **Why:** the deny rule is the floor the classifier "can't talk its way past" — even the framework's
  classifier still misses ~17% on intent, so the deterministic deny is the backstop
  ([permission-framework-over-model](https://agentpatterns.ai/security/permission-framework-over-model/), deny-block example; [blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)).
  Deterministic auditor rule (project-auditor rulebook — provenance in the source-map): deny read+edit of
  `.env`, `.env.*`, `*.pem`, `*.key`, `id_rsa*`, `.aws/credentials`, `.ssh/**`.
- **Fix:** add the `deny` block for destructive verbs and secret paths. Remediation:
  [permissions-and-safety-boundaries](https://learn.agentpatterns.ai/harness-engineering/permissions-and-safety-boundaries/).
  *Caveat:* command-pattern denies (`Bash(rm *)`) are prefix-matched and bypassable (`/bin/rm`,
  `find -delete`, `git clean`, `cd x && rm`) — parser-bypass CVEs are documented
  ([safe-command-allowlisting, *Parser-bypass exposure*](https://agentpatterns.ai/security/safe-command-allowlisting/)).
  File-path denies are the solid part of the floor; for command execution the hard floor is the HS-1
  sandbox, not the denylist.
  *Restraint:* once a deny floor covers the standard credential/key patterns (`.env*`, `*.pem`,
  `*.key`, `id_rsa*`, `.aws/credentials`, `.ssh/**`, `*.bak`) alongside a sandbox and an explicit tool
  allowlist, do not fire HS-3 over incremental glob-coverage gaps — HS-3 is for **no** deny floor or a
  real structural gap, not for second-guessing an already-adequate one.
  *No-deny-block exception:* an **absent** `deny` block does not automatically mean "no floor" — check
  whether the `allow` list itself is narrowly path/command-scoped (e.g. `Read(./src/**)`,
  `Bash(npm test)` only, not bare `Read`/`Edit`/`Bash`) alongside a network-isolated sandbox. A narrow
  allowlist that structurally excludes credential paths and destructive commands **is** the floor —
  don't fire HS-3 for a missing explicit `deny` block on top of it. Fire HS-3 when the allow list is
  broad/unscoped (bare `Read`, `Edit`, `Bash` with no path/command restriction) AND there is no deny
  block — that combination has no containment at all.

### HS-4 — Over-broad / inherited tool grants
- **Flags:** any agent or sub-agent with a **missing or wildcard `tools:`** field (inherits
  everything) or unscoped `permissions`. *Seam:* CI-workflow `GITHUB_TOKEN` scope (`permissions:` in
  `.github/workflows/*`) is owned by `audit-github-actions-security` (GHA-4) — flag only the agent
  harness's **own** grants here.
- **Why:** damage is bounded by what you grant — scope four dimensions (tool / file / mode / repo) per
  agent and decompose a broad agent into a narrow chain; an injection can't invoke a tool never wired
  in ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)).
  Deterministic auditor rules (rulebook — provenance in the source-map): explicit
  minimal `tools` list per sub-agent (no wildcard inherit); GH Actions `read-all` + per-step write.
- **Fix:** declare an explicit minimal `tools:` list per agent (workflow-file token scope →
  `audit-github-actions-security`). Remediation: [bound-the-blast-radius](https://learn.agentpatterns.ai/security/bound-the-blast-radius/).

---

## B — Recovery: undo the damage

### HS-5 — Irreversible action with no human gate
- **Flags:** the path can send email/Slack/webhook, charge a card, deploy to prod, force-push, or
  delete an external resource — and is **not** on a branch / draft-PR / staging and has **no**
  confirmation gate.
- **Why:** choose the undo before the do; rank each action on undo cost — external side effects have
  no rollback, so the gate is the only defense ([rollback-first-design](https://agentpatterns.ai/agent-design/rollback-first-design/), undo-cost spectrum).
  Auditor rules (rulebook — provenance in the source-map):
  place human approval gates before irreversible actions (merge/publish/delete/deploy); show exact
  verbatim data; default to `[y/N]`.
- **Fix:** move the action onto a reversible primitive (branch / draft PR / staging) or add a
  human-approval gate showing the verbatim effect. Remediation:
  [reversibility-and-idempotency](https://learn.agentpatterns.ai/harness-engineering/reversibility-and-idempotency/), [snapshot-and-roll-back](https://learn.agentpatterns.ai/workflows/snapshot-and-roll-back/).
- **Headless carve-out:** before ruling out an inline ask-to-continue / human-approval gate, apply
  the **headless-evidence bar** (tail note below) — prose self-description and permission settings
  are not evidence of headlessness.

### HS-6 — Non-idempotent state mutation in a re-runnable path
- **Flags:** a workflow that creates a branch / comment / PR / external resource with no
  check-before-act, upsert, or unique-key marker, and can be re-run (headless / CI / retry).
- **Why:** agents fail mid-task and re-run with no memory; re-runs must converge to the same state via
  check-before-act, upsert, or unique idempotency keys, else they produce duplicate branches/PRs/
  comments or compounded errors ([idempotent-agent-operations](https://agentpatterns.ai/agent-design/idempotent-agent-operations/) / [AWS Builders' Library](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)).
- **Fix:** add check-before-act / upsert / a unique key; for ops that can't be made idempotent, log +
  gate. Remediation: [reversibility-and-idempotency](https://learn.agentpatterns.ai/harness-engineering/reversibility-and-idempotency/).

### HS-7 — State-changing commands with no snapshot to revert
- **Flags:** a setup / CI agent runs state-modifying commands (`pip install`, migrations, package
  installs) with no `docker commit` / checkpoint before, and verifies success on exit code alone.
- **Why:** snapshotting before each state-changing command turns environment pollution into a
  one-command revert; exit-code success ≠ feature success (SetupX; Repo2Run 86% vs 6/9/22%)
  ([experiential-setup-agents-snapshot-rollback](https://agentpatterns.ai/workflows/experiential-setup-agents-snapshot-rollback/)).
- **Fix:** snapshot/checkpoint before each state-changing command; verify the outcome, not just exit
  code. Remediation: [snapshot-and-roll-back](https://learn.agentpatterns.ai/workflows/snapshot-and-roll-back/).

---

## C — Duration & cost: stop it

### HS-8 — Bounded radius without bounded duration (no kill path)
- **Flags:** no `maxTurns` / iteration cap / circuit breaker / supervisor heartbeat / orchestrator
  timeout — a stuck or misbehaving agent can't be stopped.
- **Why:** scoping caps per-action damage, not time-integrated damage; 60% of orgs can't terminate a
  misbehaving agent — pair scope with a termination path the agent itself cannot block, and prefer a
  runtime stop (`maxTurns`) over an instruction stop the model can ignore ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/); [unbounded-consumption-resource-bounds](https://agentpatterns.ai/security/unbounded-consumption-resource-bounds/)).
- **Fix:** add `maxTurns` / iteration cap / circuit breaker / orchestrator timeout. Remediation:
  [sandboxing-and-blast-radius](https://learn.agentpatterns.ai/harness-engineering/sandboxing-and-blast-radius/), [cost-controls-and-circuit-breakers](https://learn.agentpatterns.ai/harness-engineering/cost-controls-and-circuit-breakers/).

### HS-9 — Missing consumption bounds (denial-of-wallet / DoS)
- **Flags:** requires **both** conditions together — (a) **none** of the five bounds present (per-call
  token cap, per-task iteration cap, fan-out concurrency cap, cost-velocity breaker, per-day dollar
  budget) **and** (b) the cost loop is injection-reachable (untrusted input drives the retry/fan-out
  loop that spends). Partial bound coverage (e.g. `maxTurns` alone) with no injection-reachable loop
  does **not** fire HS-9.
- **Why:** LLM call cost is variable and attacker-influenceable; no single bound covers the cost
  dimension — real incidents reached $46K/day and $82K/48h with no per-app detection firing
  ([unbounded-consumption-resource-bounds](https://agentpatterns.ai/security/unbounded-consumption-resource-bounds/) / [OWASP LLM10:2025](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/)).
  Treat the bill as an attack surface, not an accident: an adversary *deliberately* runs up cost via
  an injection-reachable retry loop or amplification (one study showed 658× cost by coercing verbose
  multi-turn chains past a 4K per-call cap), and denial-of-wallet leaves availability metrics healthy
  while the wallet drains — the runaway loop is a security control to bound, not a finance preference.
- **Fix:** add the five bounds (per-call, per-task, fan-out, velocity, per-day dollar) as
  deterministic counters — a brittle classifier in the enforcement path becomes its own DoS.
  Remediation: [cost-controls-and-circuit-breakers](https://learn.agentpatterns.ai/harness-engineering/cost-controls-and-circuit-breakers/), [the-bill-is-the-attack](https://learn.agentpatterns.ai/security/the-bill-is-the-attack/).

---

## Advisory

### HS-10 — Write-capable exploration with no read-only / plan phase
- **Flags:** multi-file or unfamiliar-codebase changes where the agent writes immediately, with no
  read-only / plan phase.
- **Why:** fixing a plan costs minutes; fixing a bad diff costs context, tokens, and reverts — a
  read-only phase surfaces wrong approaches cheaply. Auto-plan via system prompt is instruction-based;
  pair it with `--permission-mode plan` for a runtime boundary ([plan-first-loop](https://agentpatterns.ai/workflows/plan-first-loop/)).
- **Fix:** add a plan-first phase and enforce it with `--permission-mode plan`. Remediation:
  [plan-mode-and-plan-first](https://learn.agentpatterns.ai/harness-engineering/plan-mode-and-plan-first/).

---

> **Partial / careful classification.** A read-only fetch, a branch-scoped write, a tokenized
> credential, or a denylist-only sandbox is *reduced*, not *removed* — mark it partial, don't zero it.
> OS boundaries still leak (CVE escapes, denylist-reasoning bypass) — defense-in-depth: assume every
> layer fails and the sandbox is the outermost one that holds ([defense-in-depth-agent-safety](https://agentpatterns.ai/security/defense-in-depth-agent-safety/)).

> **Headless-evidence bar** — governs *every* branch on "headless": HS-2's Fix, HS-5's gate choice,
> HS-6's re-runnable flag, and Procedure step 4. Don't take a claimed "headless" / "unattended"
> self-description at face value — and don't treat a tool-permission setting as proof of it either.
> A prose line saying the agent runs unattended is not evidence of headlessness. Neither is
> `permissions.defaultMode: bypassPermissions` / `acceptEdits` or any other tool-approval-skipping
> setting — those control whether Claude Code's own tool-call prompts appear, not whether a
> human/stdin is present at all; a `bypassPermissions` run can still be sitting in front of an
> attended terminal. The ONLY evidence that genuinely establishes headlessness is an actual
> automation trigger: a CI job/cron/webhook invocation, a scheduler, systemd/supervisor management,
> or a non-interactive CLI flag (`--headless`, `-p`/print-mode with no TTY). Absent one of those, an
> ask-to-continue / human-approval gate can still apply — recommend it as a valid option alongside
> any external gate, rather than asserting inline approval categorically "can't" work.
