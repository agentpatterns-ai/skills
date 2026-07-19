# review-code — core check catalog

Loaded when reviewing (Procedure step 3). Every check is a **named canonical concept**; the finding
names the concept and its **named fix**. Attribution in parentheses is the primary source — the
published canon, not the agentpatterns corpus (this skill's declared evidence base). Apply with
judgment: the *Flags* column describes the defect, not a metric threshold.

## Contents
- [Structure & responsibility](#structure--responsibility)
- [Naming & comments](#naming--comments)
- [Error handling & robustness](#error-handling--robustness)
- [Testability & test smells](#testability--test-smells)
- [Security basics](#security-basics)
- [Performance basics](#performance-basics)
- [Tensions — where the canon disagrees](#tensions--where-the-canon-disagrees)
- [Routing — what not to review by hand](#routing--what-not-to-review-by-hand)

## Structure & responsibility

| Check (source) | Flags | Named fix |
|---|---|---|
| **Function does one thing** (Martin, *Clean Code*; SLAP) | a function mixing abstraction levels or doing several jobs; boolean/flag/selector arguments; command mixed with query (CQS violation) | Extract Function; Split Phase; split the flag into two functions; Separate Query from Modifier |
| **Large Class / God Class** (Fowler, *Refactoring* — Large Class; Martin — SOLID's SRP / *Clean Code*; Brown et al., *AntiPatterns* — God Class/God Object) | a class/module in the hunk carrying several **unrelated** responsibilities — more than one axis of change; unrelated field clusters each used by a different method group; a `Manager`/`Util`/`Helper` catch-all accreting jobs. This is the class-level counterpart of *Function does one thing*. Flag on **responsibility count, not size**: a large but *cohesive* (deep) module is not a defect — the seam vs *Deep modules* (large ≠ shallow: a deep module is good) and the *Function length* Tension both bind here, so this fires on responsibility spread, **never on line count** | Extract Class along the responsibility seam; Move Function/Field to the owner; split by axis of change |
| **Duplication — knowledge-level DRY** (Hunt & Thomas; Fowler Duplicated Code; Beck) | the same *knowledge* (rule, constant, shape) stated twice in the diff — not just similar text; third copy of an existing pattern (Rule of Three reached) | Extract Function/Class; Pull Up Method; for two-copy cases prefer AHA — wait for the third before abstracting |
| **Misplaced behaviour** (Fowler Feature Envy; GRASP Information Expert; Anemic Domain Model) | Feature Envy in the diff; domain logic in controllers/handlers/services while entities are bags of getters | Move Function/Field to the data's owner; push behaviour into the domain object |
| **Deep modules** (Ousterhout Shallow Module; Fowler Middle Man, Lazy Element) | a Shallow Module; a class/method that only delegates; a pass-through method; classitis | Inline Function/Class; Remove Middle Man; Collapse Hierarchy; merge into a deeper module |
| **Information hiding** (Ousterhout Information Leakage; Fowler Insider Trading; Spolsky Leaky Abstraction) | one design decision (a format, a schema, an ordering) visible in multiple modules; a module reaching into a sibling's internals; internals exposed "for convenience" | move the decision behind one owner; Move Function/Field; Encapsulate Variable/Record; Hide Delegate |
| **Law of Demeter** (Lieberherr et al. 1987; Martin, *Clean Code*; Fowler, *Refactoring* Message Chains) | `a.getB().getC().doThing()` chains | Hide Delegate; Tell, Don't Ask |
| **Polymorphism over conditionals** (Fowler Repeated Switches) | the *same* type-based switch/if-ladder appearing in more than one place, or a new arm added to an existing ladder | Replace Conditional with Polymorphism; Replace Type Code with Subclasses; a single, never-repeated switch is fine |
| **Substitutability / LSP** (Liskov 1987, *Data Abstraction and Hierarchy*; Liskov & Wing 1994 — signature variance; Martin — SOLID's L; Meyer, *Design by Contract* — the assertion axis) | in the hunk: an override that throws instead of implementing (`NotImplementedError`, `UnsupportedOperationException`); an override that **requires more** than its base (tightened precondition, narrowed argument type) or **promises less** (weakened postcondition, widened or newly-nullable return); a subclass a caller must `isinstance`/`instanceof` to use safely. Requiring *less* and promising *more* — a widened precondition, a covariant return — is substitutable, not a defect | Replace Inheritance with Delegation; Extract Interface to the contract the base honestly guarantees (split the hierarchy at the real seam); Introduce Special Case / Null Object where the "unsupported" arm is really an absent case |
| **Primitive Obsession & Data Clumps** (Fowler, *Refactoring* — smell catalog) | domain concepts passed as bare strings/ints (money, IDs, ranges); the same group of parameters travelling together | Replace Primitive with Object; Introduce Parameter Object; Extract Class |
| **Speculative Generality & dead code** (Fowler; YAGNI; Lava Flow; Clean Code Clutter) | hooks, parameters, or abstraction layers with a single concrete use and no second caller in sight; commented-out or unreachable code kept "just in case" | Remove Dead Code; Inline Function/Class; Collapse Hierarchy; delete — version control remembers |
| **Global & mutable shared data** (Fowler Global Data, Mutable Data) | writes to globals/singletons from deep in logic; data mutated far from where it's read | Encapsulate Variable; Split Variable; prefer immutability; pass state explicitly |
| **Right altitude** (Ousterhout — pull complexity downwards) | a special case patched onto shared infrastructure or into callers where the underlying mechanism should generalize; a workaround layered over the root cause | generalize the mechanism at the owning layer; Pull Complexity Downwards — absorb the complexity inside the module instead of exposing it |

## Naming & comments

| Check (source) | Flags | Named fix |
|---|---|---|
| **Names reveal intent** (Fowler Mysterious Name; Clean Code N-heuristics; Ousterhout Vague Name) | a name needing the body to understand it; disinformation (`list` that's a map); encodings; a name that doesn't say its side effects; naming *difficulty* itself — a hard-to-name unit is a design signal | Rename Variable/Function/Field; if the right name won't come, Extract/Split until units are nameable |
| **Comments describe what code cannot** (Ousterhout; Clean Code C-heuristics; Fowler Comments-as-deodorant) | a comment restating the line below it; obsolete/contradicting comments; a comment excusing bad structure; commented-out code | delete redundant/obsolete comments; Extract Function + Rename so code says it; keep comments for *why*, invariants, and units |
| **Repo conventions** (Clean Code G24 — Follow Standard Conventions) | a changed line clearly violating the repo's own stated rules (CLAUDE.md / CONTRIBUTING / style guide — rules a linter already enforces stay routed to tooling, step 2) — flag only when you can quote the exact rule and the exact line; no "spirit of the doc" inferences | follow the stated rule; cite the governing file and quoted rule in the finding |

## Error handling & robustness

| Check (source) | Flags | Named fix |
|---|---|---|
| **Fail fast, honest errors** (Hunt & Thomas Crash Early; Clean Code; Error Hiding anti-pattern) | empty catch blocks; broad `catch (Exception)` without rethrow/wrap/log; errors swallowed and execution continued in a corrupt state; exceptions used for control flow; error codes ignored | catch the specific expected type; Replace Error Code with Exception; handle-or-propagate — never neither; Fail Fast on impossibilities |
| **One null-safety strategy** (Clean Code — don't return/pass null; Fowler Introduce Special Case) | new code returning null where the codebase uses Optional/Result; null checks sprinkled instead of a boundary decision | Introduce Special Case (Null Object); Optional/Maybe or non-null contract — one strategy, applied consistently |
| **Resource safety** (Stroustrup — RAII; try-with-resources / defer idioms) | handles, connections, locks, temp files acquired without release on *all* paths, including the exception path | try-with-resources / using / defer / context manager; release in finally; pool clients, don't leak them |
| **Validate at trust boundaries** (McConnell barricades; allowlist over blocklist) | external input (user, network, file, env) used before validation/canonicalization; validation only client-side; blocklist filtering | validate + canonicalize at the boundary, trust inside the barricade; allowlist what's expected |
| **Assertions vs error handling** (McConnell; Design by Contract — Meyer) | assertions on user input; expected runtime failures handled by assert; impossible states silently tolerated | assertions for programmer-impossible states, error handling for expected bad input; state pre/postconditions |
| **Guard clauses over deep nesting** (Fowler Replace Nested Conditional with Guard Clauses; Beck *Tidy First?*) | arrow-shaped code — happy path buried 3+ levels deep; else-chains after early exits | Replace Nested Conditional with Guard Clauses; early return; Decompose Conditional |
| **Boundary correctness** (Clean Code G3/T5) | new loops/ranges/pagination/slicing without evident handling of empty, one, boundary, and overflow cases | fix the fencepost; add the boundary test alongside the fix |
| **Data & arithmetic honesty** (Goldberg, *What Every Computer Scientist Should Know About Floating-Point Arithmetic*; Fowler, *PoEAA* Money) | floats for money; hand-rolled date/timezone math; naive string comparison across locales/encodings; precision lost in serialization round-trips | decimal/integer minor units for money; store UTC + convert at edges via a library; explicit encodings |
| **Removed behavior** (Fowler, *Refactoring* — refactoring is behaviour-preserving by definition; Feathers, *WELC* characterization tests) | a deleted or replaced guard, validation, error path, or test whose invariant is not re-established anywhere in the new code — a removed check is a change in behavior, not a cleanup | restore the guard, or show (in the finding) where the invariant now lives; a deleted test needs a stated reason |
| **Language pitfalls** (Crockford, *JavaScript: The Good Parts*; Bloch, *Effective Java*; Slatkin, *Effective Python*) | the diff introduces a classic trap of its language — JS falsy-zero and `==` coercion, closure-captured loop variables, Python mutable default arguments and late-binding closures, Go nil-map writes and range-variable capture, float equality | the language's idiomatic safe form; add a regression test beside the fix |

## Testability & test smells

| Check (source) | Flags | Named fix |
|---|---|---|
| **Testability seams** (Feathers, *Working Effectively with Legacy Code*; Clean Code Inappropriate Static) | hidden `new` of collaborators deep in logic; static/global grabs; clock, filesystem, network, randomness read directly; real work in constructors | Parameterize Constructor/Method (inject the dependency); Extract Interface; Extract and Override Factory Method; inject clock/fs/network |
| **Characterization before change** (Feathers, *Working Effectively with Legacy Code*) | a PR editing gnarly untested code with no test pinning current behaviour first | add characterization tests, then change; Sprout Method/Class for new logic beside untested code |
| **Test smells** (van Deursen et al.; Meszaros, *xUnit Test Patterns*) | Assertion Roulette; Mystery Guest; Eager Test; Fragile/Erratic tests; Conditional Test Logic; assertion-free "coverage theatre" (Goodhart applied) | one behaviour per test, Arrange-Act-Assert; inline the fixture; split eager tests; make each assert say what failed |
| **Test the behaviour, not the implementation** (Meszaros; London-vs-Chicago debate; Khorikov, *Unit Testing Principles, Practices, and Patterns* — managed vs unmanaged dependencies) | tests asserting on private internals or mock call sequences that break on refactor; excessive mocking of value-bearing collaborators — a mocked **managed** dependency (the codebase's own DB/cache) or a mocked internal class; snapshot-test overuse | assert on observable behaviour; mock only at genuine boundaries (Test Double taxonomy: prefer fakes/stubs to interaction mocks; mock **unmanaged** out-of-process dependencies only). **Detector only — tests that already exist:** flag the over-mock and name the fix; *choosing* doubles while authoring new tests is `write-tests`' craft annex |
| **Deterministic tests** (Meszaros Erratic Test) | unmocked clock/now(), sleeps, real network, order-dependent tests, shared mutable fixtures (Test Run War) | inject time; isolate fixtures per test; remove order dependence |

## Security basics

In-diff checks only; a full security architecture review is a different engagement. CWE numbers are
cited plain for reference. These are ordinary application-source CWEs on ordinary code — the seam vs
the agent-harness security family: data-flow / egress-as-exfiltration architecture routes to
`audit-lethal-trifecta`; dependency-install / supply-chain sinks route to `audit-supply-chain-sinks`.

| Check (source) | Flags | Named fix |
|---|---|---|
| **Injection** (OWASP A03:2021; CWE-89, CWE-78, CWE-79, CWE-22) | SQL/shell/HTML/path built by string concatenation from external input; `eval` on input; un-encoded output into HTML/JS | parameterized queries; argument arrays, never shell strings; contextual output encoding; canonicalize + allowlist paths |
| **Unsafe deserialization** (CWE-502) | an unsafe deserializer run on external/untrusted bytes — `pickle.loads`, `yaml.load` (non-safe loader), Java `readObject`, Ruby `Marshal.load`, PHP `unserialize` (distinct from *Injection* above — no query/shell/HTML string is built) | a safe loader (`yaml.safe_load`, `json`, a schema-validated allowlist of types); never deserialize untrusted data into live objects |
| **SSRF** (OWASP A10:2021; CWE-918) | an outbound fetch whose URL/host derives from request input with no allowlist — webhook senders, image proxies, link-preview/URL-fetch, PDF renderers | allowlist the destination; resolve-and-pin the host; block internal/link-local ranges; disable auto-redirect following |
| **XXE** (CWE-611) | an XML/SAX/DOM parser on untrusted input with external-entity / DTD resolution left enabled | disable external entities + DTDs (defusedxml / `FEATURE_SECURE_PROCESSING` / equivalent) |
| **Secrets in code** (CWE-798; OWASP A02:2021) | credentials, tokens, keys hardcoded or logged; secrets in URLs or error messages | move to secret storage/env injection; redact logs; rotate anything already committed |
| **Crypto hygiene** (OWASP A02:2021; CWE-327, CWE-330) | home-rolled crypto; ECB mode; `random()` where security matters; string `==` on secrets; fast hashes for passwords | vetted library defaults; CSPRNG; constant-time comparison; argon2/bcrypt/scrypt for passwords |
| **Object-level authorization** (OWASP A01:2021; IDOR; mass assignment) | handlers loading records by client-supplied ID with no ownership check; request bodies bound wholesale onto ORM entities | check authorization per object, server-side; bind an explicit allowlist DTO |
| **Resource-consumption inputs** (ReDoS; zip bombs; CWE-400) | user-supplied regex or catastrophic-backtracking patterns on input; decompression/parse without size limits | bound the pattern (or use RE2-class engines); cap decompressed size and entity expansion |
| **Memory safety** (CWE Top 25 — CWE-787/CWE-125 out-of-bounds write/read, CWE-416 use-after-free, CWE-476 null dereference) | in unmanaged code (C/C++/`unsafe` Rust): unbounded copies (`strcpy`/`sprintf`/`gets`); manual pointer/index arithmetic with no bounds check; use or double-free after `free`; an allocation/lookup result dereferenced unchecked | bounded APIs (`snprintf`/`strncpy`, `std::span`/checked indexing); RAII/ownership types (smart pointers, Rust ownership) over manual `free`; check before dereference; run the language's sanitizers (ASan/UBSan) in CI |

## Performance basics

| Check (source) | Flags | Named fix |
|---|---|---|
| **I/O in loops** (the N+1 generalization; Fowler, *PoEAA* lazy-load hazard) | a query/HTTP call/file read per item inside a loop the diff introduces | batch/chunk the call; prefetch; move I/O out of the loop |
| **Sequential fan-out** (Amdahl's Law — the ceiling) | independent awaits/calls run one-by-one on a latency path | run concurrently (gather/Promise.all/errgroup) with a bound |
| **Unbounded growth** (Nygard Unbounded Result Sets) | reading a whole table/file/queue into memory; caches and collections with no eviction or cap | paginate/stream; LIMIT; bound the cache |
| **Construction in the hot path** (Bloch, *Effective Java* — avoid creating unnecessary objects) | per-request client/connection/regex/parser construction; string concatenation in loops | reuse clients and compiled objects; builders/join for strings |
| **Measured optimization** (Knuth premature optimization) | complexity added "for speed" with no measurement; micro-optimizing a cold path | require a benchmark/profile for optimization claims; keep the simple version otherwise |

## Tensions — where the canon disagrees

Named disagreements the reviewer must not flatten into mechanical flags (Procedure step 5 drops
violations of this section):

- **Function length** — Clean Code says small; Ousterhout says *depth* beats brevity — a long
  function with a simple interface can be the better design. Flag a function for mixing abstraction
  levels or jobs, **never for line count alone**.
- **Abstraction timing** — DRY vs AHA/Rule of Three: two similar copies may be cheaper than a hasty
  abstraction. Flag *knowledge* duplication confidently; flag two-copy text similarity only as Advisory.
- **Design patterns are fix vocabulary, not goals** (GoF; Golden Hammer) — recommend a pattern to
  name a fix ("this ladder is a Strategy"); adding patterns speculatively is itself Speculative
  Generality. A Builder for two fields, a Factory with one product, a Facade turned God Class — flags,
  not fixes.
- **TDD & test-first** (Beck vs Ousterhout design-first) — review the tests that exist; don't flag
  the absence of test-first process, only missing/weak tests for the changed behaviour (authoring
  new tests from a spec is `write-tests`). The same split governs **test doubles**: flag an
  over-mock **in a test the diff contains** (*Test the behaviour, not the implementation*), and
  route the *choice* of double for tests still to be written to `write-tests` — this skill is the
  detector, `write-tests` is the author. A legitimate mock of an **unmanaged** out-of-process
  dependency (third-party API, message bus) is not an over-mock; don't flag it.
- **Postel's Law** — "be liberal in what you accept" (Postel, RFC 761) is contested in modern
  practice (RFC 9413, *Maintaining Robust Protocols*); tolerant parsing at a public boundary
  deserves a note either way, not a mechanical flag.

## Routing — what not to review by hand

Deterministic tooling owns: formatting, import order, line/function-length and cyclomatic-complexity
thresholds, coverage numbers, known-CVE dependency and license scans (Procedure step 2 routes these
out). Whole-repo judgments — Shotgun Surgery, Divergent Change, cross-file duplication and dead code,
dependency cycles, Refused Bequest — are reliable only with supplied context (call graphs, usage
counts); without it they go under *Needs repo context*, never asserted as findings.

**Refused Bequest vs Substitutability / LSP** — same hierarchy, different evidence. *Substitutability*
is asserted only on what the hunk itself shows: an override that throws, a signature that requires
more or promises less, a caller type-checking a subclass. *Refused Bequest* is the whole-hierarchy
verdict — this subclass ignores most of what it inherits, so the base is the wrong parent — and needs
the base's other subclasses and its call sites to support. Without that context it stays here; don't
promote it by generalizing a single in-diff LSP violation into a claim about the hierarchy. When the
base's contract is not in the diff and not documented, only the self-evident arms hold — the throwing
override and the caller type-check; requires-more / promises-less needs the base contract to judge
against, and without it the finding goes under *Needs repo context* rather than asserting a breach of
a contract the reviewer inferred.
