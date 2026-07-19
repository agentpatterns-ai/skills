# write-tests — requirement catalog

Loaded on demand by `write-tests`. Apply each requirement while writing tests. Each item: ID /
**Flags** (the defect it prevents) / **Why** (cited corpus) / **Fix** (remediation lesson).

## Contents
- [Core — the oracle and the cycle (TT-1…TT-4)](#core--the-oracle-and-the-cycle-tt-1tt-4)
- [Coverage and assertion power (TT-5…TT-8)](#coverage-and-assertion-power-tt-5tt-8)
- [Proportionality — the floor on this catalog (TT-9)](#proportionality--the-floor-on-this-catalog-tt-9)
- [Craft — external canon (corpus lane absent)](#craft--external-canon-corpus-lane-absent)

---

## Core — the oracle and the cycle (TT-1…TT-4)

### TT-1 — Oracle read from the code, not the specification
- **Flags:** expected values derived from what the implementation returns (running it and pasting the
  output; reading the function body to decide the assert); tests added straight after an
  implementation that pass first try with no iteration; assertions that restate what the code does
  rather than what the spec requires.
- **Why:** once faulty code is in context it shapes every later token, so the tests reuse the same
  wrong assumptions the code encodes and agree with the bug instead of catching it — measured across
  five models and three benchmarks, spec-derived tests detected 25% of injected faults versus 14% for
  tests written after the model saw the implementation (~11.7pp, p<0.05), and prompting strategies did
  not recover the loss
  ([code-first-test-oracle-bias](https://agentpatterns.ai/anti-patterns/code-first-test-oracle-bias/)).
- **Fix:** derive every expectation from the specification, ticket, docstring, or type signature and
  withhold the implementation while writing the case list; where spec and code disagree, surface it as
  a finding rather than asserting the code's behaviour — see `red-green-for-agents`.
  Remediation: [learn — red-green-for-agents](https://learn.agentpatterns.ai/verification/red-green-for-agents/).

### TT-2 — Red state never observed
- **Flags:** tests handed to an implementation pass without anyone watching them fail first; a new
  test that passes before the implementation exists; a suite whose "red" was assumed from the absence
  of code rather than from a run.
- **Why:** confirming the red state first is what proves the test can fail at all — an unobserved red
  state hides tests that pass vacuously (a mis-imported module, a swallowed assertion, a test the
  runner never collected), and the phase's exit condition is a suite run, not an inference
  ([red-green-refactor-agents](https://agentpatterns.ai/verification/red-green-refactor-agents/)).
- **Fix:** run each new test and read the failure message before implementing; a test that passes
  early is broken or vacuous — fix or delete it, don't proceed past it — see `red-green-for-agents`.
  Remediation: [learn — red-green-for-agents](https://learn.agentpatterns.ai/verification/red-green-for-agents/).

### TT-3 — Red, green, and refactor mixed into one pass
- **Flags:** a single "write tests and implement this feature" instruction; test file and
  implementation file edited in the same pass; refactoring folded into the green step; only the focal
  tests run rather than the full suite.
- **Why:** mixed-phase instructions produce mixed-phase output — the implementation bleeds into the
  test logic, so the tests match what was built instead of defining what was required, which is the
  context pollution the phase separation exists to prevent; each phase has its own deterministic exit
  (fails / passes / still passes)
  ([red-green-refactor-agents](https://agentpatterns.ai/verification/red-green-refactor-agents/)).
- **Fix:** run the phases as separate passes with distinct instructions and exits, keep the red phase
  from seeing any draft implementation, and gate green on the **full** regression suite rather than
  the focal tests — see `red-green-for-agents`.
  Remediation: [learn — red-green-for-agents](https://learn.agentpatterns.ai/verification/red-green-for-agents/).

### TT-4 — Test edited, skipped, or deleted to reach green
- **Flags:** the green pass modifies a test file; assertions loosened, cases commented out, `skip`
  markers added, or expected values updated to match observed output; the green instruction omits an
  explicit "do not change the tests" constraint.
- **Why:** an agent optimising for a "tests pass" signal reaches for the cheapest edit that produces
  it, and weakening the test is cheaper than fixing the code — the documented shortcut class where the
  grader's signal is satisfied without the work being done; the countermeasure is an explicit
  prohibition on editing tests plus an end-to-end check that runs independently of the agent
  ([anti-reward-hacking](https://agentpatterns.ai/verification/anti-reward-hacking/)).
- **Fix:** state "do not change the tests" in the green instruction, treat a red test as a result
  about the code, and verify no test file appears in the green pass's diff; if a test is genuinely
  wrong, fix it in a separate, stated step against the spec — see `red-green-for-agents`.
  Remediation: [learn — red-green-for-agents](https://learn.agentpatterns.ai/verification/red-green-for-agents/).

---

## Coverage and assertion power (TT-5…TT-8)

### TT-5 — Assertion that cannot fail
- **Flags:** `assert result is not None`, `assert True`, bare no-throw tests, `assertGreaterEqual(len(x), 0)`;
  assertion-free tests that only import and call; a test asserting an incidental of the minimum
  implementation (field ordering, exact error string, rounding artifact) rather than the contract.
- **Why:** assertion precision drives implementation precision — a broad assertion passes even when
  the implementation is wrong, so it constrains nothing and buys only the appearance of coverage
  ([tdd-agent-development](https://agentpatterns.ai/verification/tdd-agent-development/)); and
  coverage tooling counts lines executed, not behaviour verified — coverage records that a line ran,
  not that the suite would notice a defect on it
  ([mutation-testing-quality-gate](https://agentpatterns.ai/verification/mutation-testing-quality-gate/)).
- **Fix:** assert the specific required value for a specific input so the test fails on a
  wrong-but-plausible result; where an exact value is genuinely unavailable, assert an invariant or a
  property rather than liveness — see `red-green-for-agents`.
  Remediation: [learn — red-green-for-agents](https://learn.agentpatterns.ai/verification/red-green-for-agents/).

### TT-6 — Happy path only
- **Flags:** no error-path case; no boundary case (empty, one, max, off-by-one, overflow); no
  malformed/invalid input; exceptions asserted nowhere; a bare `except:`/`catch {}` in the unit with
  no test that reaches it.
- **Why:** agent-written code systematically neglects error handling, edge cases, and type
  boundaries — analysis of 470 GitHub PRs found AI-generated code carries ~2x the error-handling
  issues and ~1.75x the logic and correctness errors of human-written code, so the untested paths and
  the broken paths are the same paths
  ([happy-path-bias](https://agentpatterns.ai/anti-patterns/happy-path-bias/)).
- **Fix (technique, from the Why page):** enumerate error paths and boundaries per input alongside the
  typical case, and assert the **specific** failure (exception type/message, error value) rather than
  that something went wrong — the corpus's own mitigation is that "raise ValueError for invalid input"
  is actionable where "handle errors" is not
  ([happy-path-bias](https://agentpatterns.ai/anti-patterns/happy-path-bias/)).
  **Not unconditional** — that page's own *When this backfires* fences it: prototypes, throwaway
  scripts, and framework-managed boundaries can reasonably defer exhaustive handling. Separately —
  this skill's own corollary of TT-1's spec-derived oracle, not a claim of that page — a spec that
  explicitly delegates validation to the caller is not an error path to invent: asserting on it
  fabricates a requirement the spec disclaims (TT-9).
- **Fix (stance):** don't let a fluent, clean-reading implementation stand in for a check — one assert
  against an edge case is what catches the subtle bug that "looks right" — see `trust-without-verify`.
  Remediation: [learn — trust-without-verify](https://learn.agentpatterns.ai/anti-patterns/trust-without-verify/).

### TT-7 — Shared blind spot between code and tests
- **Flags:** one model wrote both the implementation and its suite; the suite passed on first
  generation with no iteration; every test exercises the same solution strategy the code took; no
  property-based, mutation, or independent-review probe anywhere.
- **Why:** LLM errors cluster around systematic weaknesses rather than dispersing like human errors,
  so a model-generated suite misses in code exactly the cases it omits in tests — measured on leading
  code benchmarks, 50% of problems had tests that failed to detect known errors and 84% of verifiers
  were flawed, with average Pass@1 dropping 9.56% against higher-quality suites
  ([test-homogenization-trap](https://agentpatterns.ai/anti-patterns/test-homogenization-trap/)).
- **Fix (stance):** do not accept the green suite as proof — a suite from the author of the code is not
  external ground truth, and trusting it is verification theater. Get the check from outside the
  original author: review by a different model or human before the implementation is visible — see
  `trust-without-verify`.
  Remediation: [learn — trust-without-verify](https://learn.agentpatterns.ai/anti-patterns/trust-without-verify/).
- **Technique (corpus-only — no lesson covers it):** mutation testing is the discriminator between a
  ceremonial and a load-bearing suite — a surviving mutant names an assertion that catches nothing,
  where coverage only proves a line ran; property-based invariants over random inputs serve the same
  end ([mutation-testing-quality-gate](https://agentpatterns.ai/verification/mutation-testing-quality-gate/)).
  **Proportionate to the stake:** this is tooling for a load-bearing suite, not for a small pure
  function whose assertions are already exact — recommend it, don't require it (TT-9).

### TT-8 — Exact-match assertion on a non-deterministic unit
- **Flags:** string equality on LLM output; asserting an exact tool-call sequence or path; snapshot
  tests over generated prose; a retry loop added to a flaky test instead of a variance-tolerant
  assertion.
- **Why:** a unit whose output legitimately varies across identical inputs breaks equality checks in
  both directions — false negatives on correct behaviour and false positives on a lucky run — so the
  assertion must target the decision and the end-state ("is the system in the required state?")
  rather than the route taken to it
  ([behavioral-testing-agents](https://agentpatterns.ai/verification/behavioral-testing-agents/)).
- **Fix:** split the deterministic part (parsing, formatting, call construction — ordinary
  assertions) from the non-deterministic part, and assert end-state or a graded criterion for the
  latter; keep the exact-match tests where output genuinely is deterministic — see
  `testing-what-it-decides`.
  Remediation: [learn — testing-what-it-decides](https://learn.agentpatterns.ai/verification/testing-what-it-decides/).

---

## Proportionality — the floor on this catalog (TT-9)

### TT-9 — Ceremony out of proportion to the stake
- **Flags:** mutation or property-based tooling *required* on a small pure function whose assertions
  are already exact; an error-path test manufactured for a precondition the spec explicitly delegates
  to the caller; TT-6/TT-7/TT-8 recited against a unit none of them apply to; a paragraph of process
  wrapped around a one-line assertion; "not proven by this suite" padded with cases the spec disclaims.
- **Why:** verification has a cost, and spending it where the stake is low is not rigour — it is the
  same misjudgement as skipping it where the stake is high. The corpus fences its own error-path rule
  explicitly: prototypes, throwaway scripts, framework-managed boundaries, and controlled-input tooling
  can reasonably defer exhaustive handling, because the bias TT-6 names is *a production concern*
  ([happy-path-bias](https://agentpatterns.ai/anti-patterns/happy-path-bias/), *When this backfires*).
- **Fix:** calibrate — verify everything on new/high-stakes code (security, production config,
  financial data), spot-check the proven and the trivial, and don't apply production-auth scrutiny to
  a throwaway script; over-firing checks trains the reader to dismiss the real ones. A rule this
  catalog names is a rule to *apply where it bites*, not a section to recite. Where the spec disclaims
  a behaviour, the honest output says so in one line and stops — see `trust-without-verify`.
  Remediation: [learn — trust-without-verify](https://learn.agentpatterns.ai/anti-patterns/trust-without-verify/).

---

## Craft — external canon (corpus lane absent)

**Read this section as the classic test craft TT-1…TT-9 assume but do not teach.** The corpus covers
the *agent* form of TDD deeply — it deliberately does not cover the classic craft below (test-double
selection, the oracle taxonomy, test structure, test-data ownership), so these rows carry **no
Why→corpus / Fix→lesson pair**. They are cited the way the corpus-external tier requires instead:
each names a **real published work** inline, and the name is the finding's key. This is the **mixed
tier** — the TT-n detectors above keep the full two-link standard; only this bounded annex is
external. TT-9 (proportionality) is the floor here too: apply a row where it bites.

**Seam — `review-code` owns the detector half.** These rows govern **choosing and structuring
doubles, oracles, and fixtures while AUTHORING new tests**. Flagging over-mocking, Assertion
Roulette, Mystery Guest, or an Erratic Test in tests that **already exist** is `review-code`'s
*Testability & test smells* family, not this skill's.

**Why this belongs in an authoring skill: over-mocking is the dominant failure mode of generated
tests.** A test whose arrange block is mostly mock setup is testing the mocks — it re-asserts the
stub script the author just wrote, and passes whatever the implementation does. The double is chosen
at authoring time; that is the moment this annex governs.

**Canon note — where the schools disagree.** The double-selection row takes Khorikov's classicist
synthesis as the **default**, not settled consensus: Freeman & Pryce's London/mockist school (the
*Mock roles, not objects* row) legitimately mocks owned in-process collaborator roles when driving
design by interfaces, and Fowler's *Mocks Aren't Stubs* frames the classicist-vs-mockist split as
unresolved. Match a codebase's existing school rather than converting it.

| Check (source) | Flags | Fix |
|---|---|---|
| **Test-double selection by dependency kind** (Khorikov, *Unit Testing Principles, Practices, and Patterns* — managed vs unmanaged out-of-process dependencies; Meszaros, *xUnit Test Patterns* Test Double taxonomy; Fowler, *Mocks Aren't Stubs*) | every collaborator mocked regardless of kind; the **managed** dependency (your own DB, your own cache — nobody else observes it) replaced by an interaction mock; internal classes of the unit's own module mocked; an arrange block that is mostly mock setup | choose the double **by dependency kind**: mock only **unmanaged** out-of-process dependencies (third-party APIs, message buses, outbound email — communication with them is observable behaviour); use the **real thing or a fake** for **managed** ones; never mock internal classes. Stubs for **queries** (state verification), mocks for **commands** (behaviour verification) |
| **Mock roles, not objects** (Freeman & Pryce, *Growing Object-Oriented Software, Guided by Tests*) | mocks stood up against a third-party client class or a concrete type the codebase doesn't own; test breaks whenever that library's signature moves; the double asserts on a vendor SDK's call shape | mock the **role** the unit needs, not the object that happens to fill it — **only mock types you own**: wrap the third-party API behind an owned interface expressed in your domain's terms and double *that*; the vendor client gets an integration test instead |
| **Oracle taxonomy — name the oracle each test uses** (Barr, Harman, McMinn, Shahbaz & Yoo, *The Oracle Problem in Software Testing: A Survey*; Myers, *The Art of Software Testing* — a test case must define its expected result; Bach & Bolton, *Rapid Software Testing* — HICCUPPS consistency oracles) | a test with no defined expected result (it runs the code and accepts what comes back); "no exception thrown" standing in for an oracle; a golden-master file regenerated from the current implementation whenever it disagrees | name the oracle the test relies on and say so: **specified** (the spec states it — TT-1's default and the strongest), **derived / golden-master** (pinned from a trusted prior run — record what made it trusted), **implicit** (no crash, no corruption), **invariant / property**, **metamorphic** (a relation between two runs, when no single expected value exists), **differential** (against a reference implementation), **heuristic** (HICCUPPS consistency — history, comparable products, claims, user expectations, purpose, statutes). Where the spec is silent, an explicit weaker oracle beats a fabricated exact value (TT-1) |
| **Arrange-Act-Assert / one behaviour per test** (Wake — Arrange-Act-Assert; Meszaros, *xUnit Test Patterns* Four-Phase Test; Beck, *Test-Driven Development: By Example* — Isolated Tests) | phases interleaved (assert, mutate, assert again); several unrelated behaviours per test; a **multi-call Act** — the test must drive three calls in sequence to exercise one behaviour | structure each test Arrange → Act → Assert (Meszaros' setup/exercise/verify/teardown), one logical behaviour per test, and keep the **Act one line**: a multi-call Act is leaked encapsulation — the unit is making the caller do its assembly, so fix the API rather than the test |
| **Test-data ownership** (Meszaros, *xUnit Test Patterns* — Fresh Fixture, Test Data Builder, Test Run War) | a shared mutable fixture, class-level seed data, or a database row every test reads and some tests write; tests that pass alone and fail in a suite (or in parallel); data inserted straight into storage behind the code's own validation | every test **owns or generates its own data** — a Fresh Fixture per test, built through a Test Data Builder / Creation Method with only the fields the test cares about named; build data through **the same APIs production uses**, so an invalid state the code forbids can't be manufactured in the arrange block |
