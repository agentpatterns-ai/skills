---
name: review-code
description: "Review a source-code diff, PR, or file against canonical review criteria — Fowler code smells with named refactorings, Clean Code / A Philosophy of Software Design red flags, error-handling/resource-safety gaps, testability seams and test smells, security/secrets basics, performance basics, plus concurrency, database/N+1, distributed-systems, API-contract, observability modules — each defect reported as a line-anchored finding with a named fix. Invoke when asked to review code, a diff, or a PR for quality, maintainability, or correctness, or to find code smells and refactoring opportunities in ordinary application source. Skip when the target is a GitHub Actions workflow (use audit-github-actions-security), a SKILL.md (use audit-skill-quality), an agent instruction file (use audit-instruction-file), an agent harness's security/data-flow architecture (use audit-lethal-trifecta) or its LLM-output sinks/install authority (use audit-supply-chain-sinks), or when writing new tests from a spec (use write-tests)."
user-invocable: true
version: "0.5.0"
usage: /review-code [diff | PR number/URL | file/dir path]
---

# Review Code

Review source changes by **named canonical concept**: every finding names the established concept it
instantiates (a Fowler smell, a Clean Code heuristic, an Ousterhout red flag, a test smell, a CWE)
and pairs it with the **named fix** (a Fowler refactoring, a Feathers technique, a known pattern).
Named concepts are the densest vocabulary a reviewer and an LLM share — the name carries the
definition, so the review spends its budget on *application*, not explanation. This is a
**corpus-external tier** skill: its evidence base is the published canon it names, not the agentpatterns corpus.

**Stance — findings report; read-only; precision over recall.** Recommend, never rewrite — apply a
fix only when the user asks. A clean hunk is a valid verdict; an unanchored or hedged finding is not
a finding. Route metric-detectable checks to deterministic tooling instead of restating them.

## Input
- A diff / PR (preferred — review the change, not the codebase) or file/dir paths. With a PR, read
  its description and commit messages for intent. No target given: review
  `git diff @{upstream}...HEAD` (fall back `main...HEAD` / `HEAD~1`) plus uncommitted `git diff HEAD`.

## Scope
Ordinary application/library source and its tests, any language. **Out of scope:** CI workflow
YAML (→ `audit-github-actions-security`); SKILL.md / instruction-file prose (→ `audit-skill-quality`
/ `audit-instruction-file`); agent-harness security architecture (→ `audit-lethal-trifecta` and
siblings); pure formatting/style owned by linters (route out, step 2).

## Procedure
1. **Establish intent and select modules.** State what the change is trying to do (from the PR/commit
   text or the code itself), the language/stack, and which [`domains.md`](domains.md) modules the
   diff touches (concurrency, database, distributed/messaging, API contract, observability).
   Done when intent, stack, and the selected module list are written down.
2. **Route to tooling.** List what deterministic tooling owns for this repo — the *Routing* list in
   [`checks.md`](checks.md) (formatting, complexity metrics, CVE scans, …) — and exclude those from
   agent findings. Done when the routed-out list is recorded (empty is a valid record for a repo
   with no tooling — then note the gap as one advisory finding).
3. **Core pass.** Read the diff hunk-by-hunk against [`checks.md`](checks.md), plus three diff-level
   moves: read the **enclosing function** of each hunk (defects on unchanged lines of a touched
   function are in scope); for every **deleted or replaced** line, name the invariant it enforced
   and find where the new code re-establishes it; when a changed function's contract shifts
   (signature, return shape, new exception/precondition), grep its **callers** — a broken call site
   is a finding. For each candidate finding, anchor it `file:line`, name the concept, and name the
   fix. Judge changed code plus what it directly touches — not the whole file's pre-existing debt
   (Boy Scout radius: note inherited debt once, in *Pre-existing*, don't fail the PR on it).
   Done when every hunk has a verdict — finding(s) or clean.
4. **Domain pass.** Run each module selected in step 1 from [`domains.md`](domains.md) over the
   relevant hunks. Done when every selected module has recorded verdicts.
5. **Precision pass.** Drop any finding that lacks a named concept, a line anchor, or a concrete
   named fix; a High (correctness/security) finding must also state its concrete failure scenario —
   the inputs or state that produce the wrong outcome; drop mechanical flags the canon itself disputes (see *Tensions* in
   [`checks.md`](checks.md) — e.g. function length alone is not a finding); move whole-repo
   hypotheses (the *Routing* section's whole-repo list — e.g. Shotgun Surgery) to *Needs repo
   context* unless the context was supplied. Assign severity: **High** = correctness, security, data
   loss, or resource leak; **Medium** = maintainability defect worth fixing in this PR; **Advisory**
   = optional tidying (Beck's *Tidy First?* scale). Done when every surviving finding carries
   concept + anchor + named fix + severity (a High also its failure scenario), and each dropped
   candidate has a one-line drop reason recorded.
6. **Report** with the output template, then offer next steps (apply fixes, file follow-ups) —
   read-only until the user confirms. Done when every template section is filled (with "none" where
   empty) and each finding row is complete.

## Checks
- [`checks.md`](checks.md) — the core catalog, loaded when reviewing: structure & responsibility,
  naming & comments, error handling & robustness, testability & test smells, security basics,
  performance basics, plus the *Tensions* the canon disagrees on.
- [`domains.md`](domains.md) — conditional modules loaded per step 1: concurrency & async, database
  & persistence, distributed systems & messaging, API & contracts, observability & operations.

## Worked example
A PR adds `OrderService.calculateShipping(customer)` that reads five fields off `Customer` and none
off `OrderService` → **Feature Envy (Fowler)** — Medium at `order_service.py:41` — fix: **Move
Function** onto `Customer` (or a `ShippingCalculator` taking a customer). Same PR wraps the DB call
in `try: … except Exception: pass` → **Error Hiding / empty catch** — High at `order_service.py:58`
— fix: catch the specific expected exception, handle or re-raise; **Fail Fast** for the rest.

## Output template
```
# Code review — <diff/PR> (<N> hunks, <files>)
Intent: <what the change does> · Modules: <selected domain modules> · Routed to tooling: <list/none>

## Findings
| Sev | Where | Finding (named concept + backing work) | Why it matters here | Named fix |
|---|---|---|---|---|
| High | order_service.py:58 | Error Hiding — empty catch (Hunt & Thomas, *The Pragmatic Programmer* — Crash Early) | swallows DB failures; order silently unsaved | catch specific exception, re-raise; Fail Fast |
| Med | order_service.py:41 | Feature Envy (Fowler, *Refactoring*) | shipping logic reads 5 Customer fields | Move Function to Customer |
| Adv | order_service.py:12 | Comment repeats code (Martin, *Clean Code* C2) | restates the assignment | delete; Extract Function if intent unclear |

**Clean:** <hunks/aspects that pass, and which checks they passed.>
**Highest-impact fix:** <the one change to make first.>
**Pre-existing (out of PR scope):** <inherited debt noted once, or none.>
**Needs repo context:** <cross-file hypotheses needing call graphs/usage data, or none.>
```

## Related
- The one shelf skill whose evidence base is the external canon (Fowler, Martin, Ousterhout,
  Feathers, Beck, McConnell, Hunt & Thomas, OWASP/CWE) rather than the corpus; sibling boundaries
  are in *Scope*.
- **Transform pair — `write-tests`.** It authors new tests from a specification and drives the
  red-green-refactor cycle; this skill reviews the tests that already exist (*Testability & test
  smells*) and, per *Tensions*, never flags a missing test-first process — that lane is `write-tests`.
  The pair splits by tense: this skill **flags** over-mocking and test smells in tests the diff
  **already contains**; **choosing** a test double while authoring new tests is `write-tests`.

## Critical rules (read last)
- **Every finding = named concept + its backing work + line anchor + named fix + severity** — the work
  is what a reader follows here (this skill cites canon, not corpus/lesson URLs); less gets dropped.
- **Precision over recall** — clean is a verdict; respect the *Tensions* section (never flag length
  or a missing pattern mechanically); whole-repo smells go to *Needs repo context*, not Findings.
- **Route metrics to tooling** — formatting, complexity thresholds, coverage, CVE scans are not
  agent findings.
- **Read-only** — recommend named fixes; apply nothing without confirmation.
