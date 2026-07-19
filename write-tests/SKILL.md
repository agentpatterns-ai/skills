---
name: write-tests
description: Write tests that pin behaviour and constrain the implementation — derive each expected value from the stated specification rather than from the implementation it exercises, confirm the failing state before implementing, keep minimum-implementation and refactor work in separate passes, sharpen weak assertions, and cover the error paths and blind spots that let a green suite mislead. Invoke when asked to write, add, or strengthen tests for a function or feature, or to drive an implementation test-first (red-green-refactor). Skip when the ask is to review or flag existing tests — smells, testability seams, weak coverage on a diff — without writing new ones (use review-code), or auditing whether an agent harness's completion gate can be gamed (use audit-verification-gates).
user-invocable: true
version: "0.5.0"
usage: /write-tests [function/feature + its specification, or a path to the code to drive test-first]
---

# Write Tests

A test is only worth its runtime if it can **fail when the code is wrong** — and the cheapest way to
lose that property is to write it with the implementation on screen: across five models and three
benchmarks, spec-derived tests caught 25% of injected faults vs 14% once the model had seen the
implementation (~11.7pp, p<0.05; no prompting strategy recovered it)
([code-first-test-oracle-bias](https://agentpatterns.ai/anti-patterns/code-first-test-oracle-bias/)).
The suite still runs green — it simply agrees with the bug. Everything below keeps the oracle (the
expected value) sourced from the **specification**, never from the code under test.

**Stance — write tests, and say what they do not prove.** Emit test code plus the case list it came
from, and name the cases left uncovered — a passing suite is evidence about the cases it encodes and
nothing more ([trust-without-verify](https://agentpatterns.ai/anti-patterns/trust-without-verify/)).
- **The costliest mistake first: never read the implementation to decide what a test should expect.**
  Derive expectations from the spec, docstring, ticket, or caller's needs — a spec/code disagreement
  is a **finding to surface**, not a test to bend around it (TT-1).
- **Never weaken or delete a test to make a suite pass.** A failing test is a result; editing it to
  fit the code destroys the only independent signal in the loop
  ([anti-reward-hacking](https://agentpatterns.ai/verification/anti-reward-hacking/)).

## Input
- Required: the **unit under test** (function/feature/module) and its **specification** — a ticket,
  docstring, type signature, acceptance criteria, or a prose statement of required behaviour.
- Optional: existing suite + conventions to match, known edge cases, the bug a regression test pins.
- **No specification supplied?** Ask for one, or write the behaviour you infer as an explicit
  assumption list and get it confirmed **before** asserting on it — never infer it silently from the
  implementation.

## Scope
Authors **new test code** for one unit — test-first against code not yet written, or added to an
implementation that already exists — including the craft decisions authoring forces: which **test
double** each dependency gets, which **oracle** each test relies on, how the test is **structured**,
and where its **data** comes from (craft rows in [`checks.md`](checks.md)). **Out of scope:**
reviewing tests that already exist for smells, testability seams, or coverage gaps on a diff →
`review-code` (the declared detector pair: this skill **chooses** doubles while authoring;
`review-code` **flags** bad ones — over-mocking and the smell catalog — in existing tests); a
harness's completion-gate architecture, hooks, or graders → `audit-verification-gates`; choosing a
test framework or CI topology.

## Procedure
1. **State the specification as a case list before reading any implementation** — input → required
   output, from the spec/ticket/signature. If code is already in context, name each expectation's
   source (a spec line, never a code line) (TT-1).
   Done when every case names its required output and spec source, and none traces to the implementation.
2. **Enumerate past the happy path** — error paths, boundaries (empty, one, max, overflow), type
   edges: the paths agent-written code most often omits and breaks at once (TT-6).
   Done when the list holds an error-path and a boundary case per input, or states why the unit has none.
3. **Choose each test double by dependency kind before writing the arrange block** — mock only
   **unmanaged** out-of-process dependencies (third-party APIs, message buses); use the real thing or
   a **fake** for **managed** ones (your own DB); leave the unit's internal classes unmocked (craft:
   *Test-double selection*, *Mock roles, not objects*).
   Done when every double names its dependency's kind and the arrange block is not mostly mock setup.
4. **Write each test with an assertion that can fail** — Arrange-Act-Assert, one behaviour per test
   with a one-line Act, each test owning its own data, asserting the required value rather than
   liveness (`is not None`, `assert True`, bare no-throw). Name the oracle where it isn't the spec;
   where output is legitimately non-deterministic, assert the decision and end-state, not an exact
   string (TT-5, TT-8; craft: *Oracle taxonomy*, *AAA*, *Test-data ownership*).
   Done when every assertion would fail on a wrong-but-plausible value.
5. **Confirm red — watch each test fail for the reason you predicted** (TT-2). Test-first: run
   before implementing; a test that passes with no implementation is broken — fix or delete it.
   **Implementation already exists** (the add/strengthen lane, where every correct test passes first
   try): prove each new test can fail — assert a deliberately wrong value or temporarily invert the
   behaviour under test, watch the predicted failure, then restore both.
   Done when every new test has been observed failing for its predicted reason and — in the
   existing-code lane — passes against the restored, unmodified code.
6. **Test-first lane only: go green with the minimum implementation, tests frozen** — a pass separate
   from test authoring keeps the implementation out of the test logic (TT-3, TT-4). Tests added to
   existing code skip to step 7.
   Done when the full suite passes (not just the focal tests) and no test file changed in this step.
7. **Refactor against the green suite, then break the blind spot in proportion to the stake** — on a
   load-bearing suite, get variance from outside its author: an adversarial pass with fresh eyes, a
   property-based invariant, or a mutation run to find assertions that never fail (TT-7). Recommend
   that tooling; never require it on a unit whose assertions are already exact (TT-9).
   Done when the suite still passes post-refactor and a blind-spot probe has run or is named as a
   recommendation — or, on a trivial unit, one line says why none is warranted.

## Requirements (TT-1…TT-9)
Each — **Flags** / **Why** (cited corpus) / **Fix** (remediation lesson) — lives in
[`checks.md`](checks.md); load it when writing tests. Index: TT-1 oracle read from the code · TT-2
unobserved red · TT-3 phases mixed in one pass · TT-4 test edited to pass · TT-5 assertion that cannot
fail · TT-6 happy-path only · TT-7 shared blind spot · TT-8 exact-match on a non-deterministic unit ·
TT-9 ceremony out of proportion to the stake.

**Craft — external canon.** `checks.md` also carries a bounded *Craft* annex (test-double selection ·
mock roles, not objects · oracle taxonomy · AAA / one behaviour per test · test-data ownership) for
the classic craft the corpus deliberately lacks; each row cites its published canon inline, while the
TT-n rules keep both links. **Over-mocking is the dominant failure mode of generated tests** — make
the double a deliberate choice (Procedure step 3).

**TT-9 is the floor on the other eight** — apply a rule where it bites and stay silent where it
doesn't. A small pure function with a spec that disclaims validation gets the test it asked for plus
one line on what isn't covered: no mutation-tooling requirement, no TT-6/TT-7/TT-8 recital. Two
asymmetries it guards: a precondition the spec delegates to the caller is **not** a missing error
path — asserting on it invents a requirement the author disclaimed — and a mock of an unmanaged
out-of-process dependency is the **correct** double, not an over-mock to remove (craft).

## Output template
Spec: *"`apply_discount(cents, pct)` takes a whole-cent amount and an integer 0–100, returns the
discounted amount rounded half-up. Rejects a pct outside 0–100."*

```
## Cases (from spec; implementation not read — TT-1)
| Case | Input | Required output | Spec source | Rule |
|---|---|---|---|---|
| typical | (1000, 10) | 900 | "discounted amount" | TT-1 |
| rounds half-up | (999, 50) | 500 (499.5 → 500) | "rounded half-up" | TT-1 |
| boundary: 0 / 100 pct | (1000, 0) / (1000, 100) | 1000 / 0 | "0–100" | TT-6 |
| error: pct out of range | (1000, 101) / (1000, -1) | raises ValueError | "Rejects a pct outside 0–100" | TT-6 |

## Tests
def test_rounds_half_up():
    assert apply_discount(999, 50) == 500        # not `is not None` — TT-5

def test_rejects_pct_above_range():
    with pytest.raises(ValueError):
        apply_discount(1000, 101)

## Red confirmed (TT-2)
6/6 failed — NameError: apply_discount is not defined. Expected: no implementation yet.

## Not proven by this suite
- Negative `cents` — spec is silent. **Question for the author, not a guessed assert.**
- Blind-spot probe recommended: property test `0 <= out <= cents` over random inputs (TT-7).
```

## Related / pairing
- **Detector pair — `review-code`.** It reviews the tests that **already exist** (test smells,
  testability seams, characterization gaps, over-mocking), rewriting nothing, and by its own scope
  does not flag a missing test-first process — that lane is here. The pair splits by **tense and
  verb** — authoring a double here, flagging a bad one there — keep it reciprocal as both evolve.
- **`audit-verification-gates`** owns the harness's completion gate — whether "done" can be gamed;
  different target (harness config, not a unit under test). Cited catalog: [`checks.md`](checks.md).

## Critical rules (read last)
- **The oracle comes from the specification, never from the code under test.** A test written against
  an implementation inherits its faults and goes green anyway — TT-1 carries the measured detection
  drop. If spec and code disagree, that is a finding to raise — not a test to reshape.
- **Never weaken, skip, or delete a test to reach green.** Freeze the tests during the green pass; a
  failure is a result, not an obstacle.
- **Confirm red before green — in both lanes.** A test never seen failing has proven nothing; when
  the implementation already exists, force the failure, then restore (step 5).
- **Choose the double by the dependency's kind, not by habit** — mock **unmanaged** out-of-process
  dependencies only; real thing or fake for **managed** ones; internals stay unmocked by default
  (the annex's canon note names the dissenting school — match the codebase's). A mostly-mock arrange
  block tests the mocks, and mocks agree with whatever the code does.
- **Say what the suite does not prove.** Name the uncovered cases; a green suite is evidence about the
  cases it encodes and nothing more.
- **Fit the ceremony to the stake.** Apply each rule where it bites and stay silent where it doesn't —
  a catalog recited at a trivial unit is noise, and noise is how the finding that mattered gets missed.
