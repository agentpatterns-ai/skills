# audit-github-actions-security — detectors

Loaded on demand by `audit-github-actions-security`. For each workflow / action, run the detectors
below (GHA-1…GHA-10) plus the GHA-0 precision guard. Each item: ID / **Flags** / **Why** (corpus +
the authoritative OWASP/GitHub source) / **Fix** (deterministic) + remediation lesson. All detection
is read-only.

The basis is the **[OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)**
(`CICD-SEC-1…10`); every finding emits its check ID **and** its `CICD-SEC-N` category. The unifying
rule: a workflow runs privileged automation on every event, sometimes on attacker-influenced input —
so untrusted input must never reach a privileged runtime, and every trust + permission must be an
immutable, least-privilege, deterministic control.

## Contents
- [GHA-1 — Script injection via untrusted `${{ … }}` expansion](#gha-1--untrusted----expanded-into-a-run-shell-script-injection--cicd-sec-4)
- [GHA-2 — Pwn-request: privileged trigger builds untrusted code](#gha-2--privileged-trigger-checks-out--builds-untrusted-code-pwn-request--cicd-sec-4)
- [GHA-3 — Third-party action pinned to a mutable ref](#gha-3--third-party-action-pinned-to-a-mutable-ref-not-a-full-sha-dependency-chain-abuse--cicd-sec-39)
- [GHA-4 — `GITHUB_TOKEN` over-privileged / no `permissions:` block](#gha-4--github_token-over-privileged--no-permissions-block-iam--cicd-sec-25)
- [GHA-5 — Long-lived secrets instead of OIDC](#gha-5--long-lived-secrets--secret-leakage-instead-of-oidc-credential-hygiene--cicd-sec-6)
- [GHA-6 — Fork-PR / deploy runs without an approval gate](#gha-6--fork-pr--deploy-runs-without-an-approval-or-flow-control-gate-cicd-sec-1)
- [GHA-7 — Self-hosted runner reachable by untrusted code](#gha-7--self-hosted-runner-reachable-by-untrusted-code-insecure-config--cicd-sec-7)
- [GHA-8 — Ungoverned 3rd-party services / remote-script execution](#gha-8--ungoverned-3rd-party-services--remote-script-execution-cicd-sec-8)
- [GHA-9 — Artifacts built/published without integrity or provenance](#gha-9--artifacts-builtpublished-without-integrity-or-provenance-cicd-sec-9)
- [GHA-10 — Security-masking config / insufficient logging](#gha-10--security-masking-config--insufficient-logging--visibility-cicd-sec-10)
- [GHA-0 — Precision / anti-theatre guard](#gha-0--precision--anti-theatre-guard-do-not-flag-a-hardened-workflow)

---

## GHA-1 — Untrusted `${{ … }}` expanded into a `run:` shell (script injection · CICD-SEC-4)
- **Flags:** a `run:` (or `actions/github-script`) body that inlines `${{ github.event.* }}`,
  `github.head_ref`, `${{ github.event.pull_request.title/body }}`, `*.comment.body`, or any
  attacker-settable context directly into the shell command.
- **Why:** PR titles/bodies/branch names are attacker-controlled; GitHub Actions expands `${{ }}`
  **before** the shell runs, so the value becomes shell syntax → RCE on the runner with whatever
  token/secrets are in scope. This is the GitInject class — untrusted content meeting elevated CI
  permissions in one runtime ([ai-agents-in-ci-cd-with-elevated-permissions](https://agentpatterns.ai/anti-patterns/ai-agents-in-ci-cd-with-elevated-permissions/);
  GitHub *Understanding the risk of script injections*; OWASP CICD-SEC-4 Poisoned Pipeline Execution).
- **Fix:** bind the value to an `env:` variable on the step and reference it quoted (`"$TITLE"`) so it
  is data, never expanded into the command; for complex logic use `actions/github-script` with
  `context`. Remediation: [agents-in-ci-cd](https://learn.agentpatterns.ai/anti-patterns/agents-in-ci-cd/).

## GHA-2 — Privileged trigger checks out / builds untrusted code (pwn-request · CICD-SEC-4)
- **Flags:** `on: pull_request_target` or `workflow_run` (which run in the **base** repo context with
  secrets + write `GITHUB_TOKEN`) combined with **either** `actions/checkout` of the PR head
  (`ref: ${{ github.event.pull_request.head.sha }}`) and build/test/lint steps on it, **or**
  `download-artifact` of an artifact the untrusted run uploaded whose content is then **executed,
  installed, or interpreted as code** (`run` on a contained script, `npm ci`/`pip install` against
  it, sourcing it) — artifact poisoning, the same pwn-request class without a checkout. Downloading
  and *reading an artifact as data* (parsing metrics/text with no exec path) is the safe consumer
  shape — do not flag the unzip alone.
- **Why:** the privileged context exposes secrets and a write token; executing the fork's code —
  checked out **or smuggled in via an uploaded artifact** — runs attacker-controlled code with that
  privilege; the canonical "pwn request"
  ([ai-agents-in-ci-cd-with-elevated-permissions](https://agentpatterns.ai/anti-patterns/ai-agents-in-ci-cd-with-elevated-permissions/);
  GitHub `pull_request_target` docs / GHSL *preventing pwn requests*; OWASP CICD-SEC-4).
- **Fix:** do not build untrusted code in a privileged trigger; use `pull_request` for build/test, or
  split into a label-gated two-workflow pattern where the privileged half never checks out head code
  and consumes artifacts **only as validated data** (parsed, schema-checked — never executed,
  installed, or sourced). Remediation: [agents-in-ci-cd](https://learn.agentpatterns.ai/anti-patterns/agents-in-ci-cd/).

## GHA-3 — Third-party action pinned to a mutable ref, not a full SHA (dependency chain abuse · CICD-SEC-3/9)
- **Flags:** a third-party `uses: org/action@v4` / `@main` / `@<branch>` / `@<short-sha>` — any ref
  that is not a **full 40-character commit SHA** (first-party `actions/*` and local `./` actions are
  lower risk but still SHA-pin for tamper-evidence).
- **Why:** a tag or branch is a mutable pointer; a compromised or malicious upstream can repoint it
  into your runner with your token's blast radius — the dependency-chain-abuse surface that mutable
  version refs widen ([agent-emitted-dependency-ranges](https://agentpatterns.ai/security/agent-emitted-dependency-ranges/);
  GitHub *pin actions to a full-length commit SHA*; OWASP CICD-SEC-3 / artifact integrity CICD-SEC-9).
- **Fix:** pin every third-party `uses:` to a full commit SHA (keep the version in a trailing comment)
  and let Dependabot bump it via reviewed PRs. Remediation: [the-package-that-doesnt-exist](https://learn.agentpatterns.ai/security/the-package-that-doesnt-exist/)
  (*nearest lesson — teaches the same fail-closed exact-pin control via lockfile-enforced installs;
  the action-SHA mechanics are in the GitHub pin-to-full-SHA doc above*).

## GHA-4 — `GITHUB_TOKEN` over-privileged / no `permissions:` block (IAM · CICD-SEC-2/5)
- **Flags:** no top-level `permissions:` key (the token defaults to the broad repo/org default, often
  read-write), `permissions: write-all`, or a job granted `contents: write` / `id-token` / `packages`
  it does not use.
- **Why:** the default token scope exceeds least privilege; a single compromised step (see GHA-1/3)
  then inherits write/push/release power — over-broad standing privilege is the blast radius to bound
  ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/);
  GitHub *Controlling permissions for GITHUB_TOKEN*; OWASP CICD-SEC-5 Insufficient PBAC / CICD-SEC-2 IAM).
- **Fix:** set `permissions: {}` at the top level (default-deny) and grant each job only the scopes it
  needs (`contents: read`, etc.). *Seam:* GHA-4 owns workflow-file token scope — `audit-harness-safety`
  (HS-4) defers it here; the agent harness's own `tools:`/permission grants stay with HS-4. Remediation: [bound-the-blast-radius](https://learn.agentpatterns.ai/security/bound-the-blast-radius/).

## GHA-5 — Long-lived secrets / secret leakage instead of OIDC (credential hygiene · CICD-SEC-6)
- **Flags:** static long-lived cloud keys in `secrets.*` for deploy/registry auth (vs OIDC
  `id-token: write` federation), `${{ secrets.* }}` echoed/printed in a `run:`, or any secret reachable
  by a fork-PR build or an unpinned 3rd-party step.
- **Why:** long-lived credentials maximise the window and blast radius of a leak; short-lived federated
  tokens are the hardened control, and a secret printed or handed to untrusted code is exfiltrated
  ([workload-identity-federation-for-agents](https://agentpatterns.ai/security/workload-identity-federation-for-agents/);
  GitHub *OIDC hardening*; OWASP CICD-SEC-6 Insufficient Credential Hygiene).
- **Fix:** use OIDC federation (`permissions: id-token: write` + cloud trust policy) for short-lived
  tokens; never echo secrets; scope secrets to protected environments, not fork builds. *Seam:* a
  static **model-API key** (`ANTHROPIC_API_KEY` etc.) is `audit-secret-exposure`'s lane (SE-4 — WIF
  posture, shadowing edge); GHA-5 owns every other repo/workflow secret. Remediation:
  [keys-that-expire-in-minutes](https://learn.agentpatterns.ai/security/keys-that-expire-in-minutes/).

## GHA-6 — Fork-PR / deploy runs without an approval or flow-control gate (CICD-SEC-1)
- **Flags:** privileged or deploy jobs that auto-run on `pull_request` from forks with no
  `environment:` protection rule / required reviewer, no "require approval for outside collaborators"
  posture, and no manual gate before a release/deploy step. *Disposition:* a missing `environment:` block /
  manual gate is **YAML-verdictable — fire on it**; only the settings-level conjuncts (the "require
  approval for outside collaborators" posture, environment required-reviewers) are repo/org settings —
  append "UNVERIFIED — confirm in repo settings (or `gh api`)" for those, never suppress the finding.
- **Why:** without a flow-control chokepoint, a single PR or push drives a privileged action straight
  to production with no human/approval barrier — insufficient flow control
  ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/);
  GitHub *approving workflow runs from public forks* / environment protection rules; OWASP CICD-SEC-1).
- **Fix:** gate privileged/deploy jobs behind an `environment:` with required reviewers; require
  approval for first-time/outside-collaborator runs; separate build from deploy. Remediation:
  [bound-the-blast-radius](https://learn.agentpatterns.ai/security/bound-the-blast-radius/).

## GHA-7 — Self-hosted runner reachable by untrusted code (insecure config · CICD-SEC-7)
- **Flags:** `runs-on: self-hosted` (or a self-hosted label) on a workflow triggered by
  `pull_request`/`pull_request_target` in a **public** repo, or a non-ephemeral runner shared across
  jobs/repos. *Disposition:* `runs-on: self-hosted` + a PR trigger is **YAML-verdictable — fire on it**
  (severity per the guide); repo visibility and runner ephemerality refine it but are not in the
  YAML — when unknown, append "UNVERIFIED — confirm repo visibility / runner config" to the finding,
  never suppress it.
- **Why:** persistent self-hosted runners run fork-submitted code on your infrastructure with whatever
  state/network the host carries — untrusted execution without an isolation boundary
  ([dual-boundary-sandboxing](https://agentpatterns.ai/security/dual-boundary-sandboxing/);
  GitHub *Self-hosted runner security*; OWASP CICD-SEC-7 Insecure System Configuration).
- **Fix:** never run public-fork PRs on self-hosted runners; use ephemeral, network-isolated,
  single-job runners (e.g. fresh VM/container per run). Remediation:
  [pick-your-sandbox](https://learn.agentpatterns.ai/security/pick-your-sandbox/).

## GHA-8 — Ungoverned 3rd-party services / remote-script execution (CICD-SEC-8)
- **Flags:** `curl … | bash` / `wget … | sh`, downloading and executing a remote script, installing a
  tool from an unverified URL, or calling an external service with no pin/checksum/egress control.
- **Why:** piping a remote script into a shell trusts whatever the endpoint serves at run time —
  ungoverned third-party trust with an open egress path
  ([agent-network-egress-policy](https://agentpatterns.ai/security/agent-network-egress-policy/);
  OWASP CICD-SEC-8 Ungoverned Usage of 3rd Party Services).
- **Fix:** pin the installer to a version + verify a checksum/signature before executing; prefer a
  SHA-pinned action; constrain runner egress to an allowlist. Remediation:
  [the-model-is-not-the-firewall](https://learn.agentpatterns.ai/security/the-model-is-not-the-firewall/).

## GHA-9 — Artifacts built/published without integrity or provenance (CICD-SEC-9)
- **Flags:** build/release/publish steps that emit artifacts (images, packages, release assets) with
  no signing, no build-provenance attestation, no checksum on release assets, or a cache restored from
  an untrusted key into a privileged build.
- **Why:** an unsigned, un-attested artifact can't be verified downstream — the integrity gap that
  supply-chain compromises (SolarWinds/Codecov class) exploit
  ([tool-signing-verification](https://agentpatterns.ai/security/tool-signing-verification/);
  SLSA / GitHub artifact attestations `actions/attest-build-provenance`; OWASP CICD-SEC-9).
- **Fix:** generate build-provenance attestations and sign artifacts/images; publish + verify
  checksums; scope cache keys so untrusted PRs can't poison a trusted build. Remediation:
  [the-provenance-blind-model](https://learn.agentpatterns.ai/security/the-provenance-blind-model/)
  (*nearest lesson — teaches the same failure class, a consumer that cannot verify where its input
  came from; the signing/attestation mechanics are in the SLSA / GitHub attestation docs above*).

## GHA-10 — Security-masking config / insufficient logging & visibility (CICD-SEC-10)
- **Flags:** `continue-on-error: true` on a security/test gate (failure silently passes),
  `set -x`/verbose shell tracing in a secret-bearing `run:` step (log masking redacts registered
  secrets, not values derived or transformed from them), over-broad triggers (`on: [push]` with
  no path/branch scope on a privileged workflow), or no audit-log retention for privileged runs
  (*`ACTIONS_STEP_DEBUG` — enabled via a repo secret/variable or the re-run UI — and audit-log
  retention are **repo/org settings**, not workflow YAML: when only files are available, emit
  "UNVERIFIED — confirm in repo/org settings", not a finding*).
- **Why:** a masked failure or a blind pipeline removes the chokepoint where a divergence between
  intended and actual action would be caught — the audit/visibility gap
  ([action-audit-divergence-taxonomy](https://agentpatterns.ai/security/action-audit-divergence-taxonomy/);
  OWASP CICD-SEC-10 Insufficient Logging and Visibility).
- **Fix:** don't mask security gates with `continue-on-error`; keep `set -x`/debug tracing out of
  secret-bearing steps; scope triggers tightly; retain and review audit logs for privileged runs. Remediation:
  [the-log-is-the-truth](https://learn.agentpatterns.ai/observability/the-log-is-the-truth/).

---

## GHA-0 — Precision / anti-theatre guard (do NOT flag a hardened workflow)
- **Flags (suppress findings when present):** third-party `uses:` already SHA-pinned; a top-level
  least-privilege `permissions:` block; no untrusted-code checkout under a privileged trigger; OIDC
  used instead of long-lived secrets; no secret exposed to fork builds; security gates not masked.
- **Why:** a security audit that flags already-correct config is noise that trains reviewers to ignore
  it — confirm clean controls, don't duplicate them ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)).
- **Fix:** report these in the **Clean** section with the control that holds them; raise a finding only
  on a real, exploitable gap. Remediation: [bound-the-blast-radius](https://learn.agentpatterns.ai/security/bound-the-blast-radius/).
