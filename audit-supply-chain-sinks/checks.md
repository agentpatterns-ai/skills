# audit-supply-chain-sinks — detectors

Loaded on demand by `audit-supply-chain-sinks`. For each execution path, enumerate the sinks that
consume LLM-emitted strings and classify each with the detectors below, then run the install-authority
and scoping checks. Each item: ID / **Flags** / **Why** (cited) / **Fix**. All detection is read-only.

The unifying rule (OWASP LLM05): trust does not transfer through a string boundary — validate at each
sink, not at the source ([improper-output-handling-downstream-sinks](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
Remediation lesson for SS-1/2/5/6/7/8: [the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

## Contents
- [SS-1 — Model string reaches `exec`/`eval`/shell (RCE)](#ss-1--model-string-reaches-execevalshell-with-no-per-sink-gate-rce)
- [SS-2 — SQL built by string-concatenating LLM output (SQLi)](#ss-2--sql-built-by-string-concatenating-llm-output-sqli-cve-2025-1793-class)
- [SS-3 — Install authority ungated (slopsquatting)](#ss-3--install-authority-ungated-by-lockfile--existence-check-slopsquatting)
- [SS-4 — Slopsquatting "defended" by edit-distance detection](#ss-4--slopsquatting-defended-by-edit-distance--typosquat-detection-mis-targeted-control)
- [SS-5 — Model output rendered without encoding/CSP (XSS + exfil)](#ss-5--model-output-rendered-in-an-auto-interpreting-surface-without-encodingcspfetch-gating-xss--image-exfil)
- [SS-6 — File path built from LLM output without canonicalisation](#ss-6--file-path-built-from-llm-output-without-canonicalisation--base-dir-constraint-traversal)
- [SS-7 — Tool/RAG output fed to a downstream LLM as trusted instructions](#ss-7--toolrag-output-fed-back-into-a-downstream-llm-as-trusted-instructions-re-injection-sink)
- [SS-8 — LLM05 conflated with LLM06 (false-safe scoping)](#ss-8--llm05-conflated-with-llm06-false-safe-scoping)
- [SS-9 — Anti-theatre / false-positive guard](#ss-9--anti-theatre--false-positive-guard-do-not-flag)

---

## SS-1 — Model string reaches `exec`/`eval`/shell with no per-sink gate (RCE)
- **Flags:** LLM-produced text passed to `eval`/`exec`, `subprocess`/`os.system`, or a shell with no
  command allowlist; a tool-executor that runs generated code verbatim.
- **Why:** the shell/`exec` sink is RCE; OWASP's control is an allowlist of permitted commands and
  refusing `eval` on any LLM-produced string ([LLM05:2025](https://genai.owasp.org/llmrisk/llm052025-improper-output-handling/) via [corpus](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
- **Fix:** allowlist permitted commands; never `eval` a model string; route intent through a
  schema-constrained object that a deterministic executor runs. Auditor: `agent-code-security-review`
  requires adversarial review for agent PRs touching exec/subprocess.
  Remediation: [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

## SS-2 — SQL built by string-concatenating LLM output (SQLi, CVE-2025-1793 class)
- **Flags:** `cursor.execute(f"… {llm_output}")`, ORM `.raw()`/`.extra()` with interpolated model
  text, the model writing a WHERE clause — no parameterised query / prepared statement.
- **Why:** the canonical instantiation is **CVE-2025-1793** in LlamaIndex (CVSS 9.8): vector-store
  integrations string-concatenated LLM-built SQL; fixed by parameterising ([corpus](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
- **Fix:** LLM emits a schema-constrained filter object (Pydantic/`Literal` fields); a deterministic
  executor builds the parameterised query — the model never writes SQL.
  Remediation: [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

## SS-3 — Install authority ungated by lockfile / existence check (slopsquatting)
- **Flags:** the agent can `uv add` / `npm install` / `pip install` an **agent-named** package with no
  lockfile-enforced resolution (`npm ci`, `uv pip sync`, `pip-compile --generate-hashes`) and no
  registry existence+provenance check; manifest edits not gated behind a reviewed PR.
- **Why:** LLMs hallucinate package names 5.2–21.7%; 43% persist across re-runs → enumerable →
  attackers pre-register them. Lanyado's `huggingface-cli` PoC drew >30k downloads ([slopsquatting](https://agentpatterns.ai/security/slopsquatting-hallucinated-package-names/);
  [Spracklen, USENIX Security 2025, arXiv:2406.10279](https://arxiv.org/abs/2406.10279)).
  Same training-distribution mechanism as agent-pinned vulnerable versions ([llm-pinned-vulnerable-versions](https://agentpatterns.ai/security/llm-pinned-vulnerable-versions/)).
- **Fix:** lockfile-enforced install path (fails closed on any unendorsed name); registry
  existence+provenance check at the install hook; remove the install leg and require a human-reviewed
  PR for manifest changes ([blast-radius-containment](https://agentpatterns.ai/security/blast-radius-containment/)).
  Lesson: [the-package-that-doesnt-exist](https://learn.agentpatterns.ai/security/the-package-that-doesnt-exist/).
  Auditor: `dependency-and-supply-chain` — pipe agent-generated requirements through lock-then-resolve.

## SS-4 — Slopsquatting "defended" by edit-distance / typosquat detection (mis-targeted control)
- **Flags:** the only install-side control is registry-side Levenshtein/typosquat heuristics keyed off
  small edit distance from popular names.
- **Why:** 48.6% of hallucinated names sit Levenshtein distance ≥6 from any real package — the
  detector looks for the wrong shape and misses the bulk of the surface ([slopsquatting](https://agentpatterns.ai/security/slopsquatting-hallucinated-package-names/)).
- **Fix:** verify **existence + provenance**, not edit distance; lockfile-enforced resolution. Treat
  this as Medium — a present-but-wrong control, distinct from no control. (Anti-control check.)
  Remediation: [learn — the-package-that-doesnt-exist](https://learn.agentpatterns.ai/security/the-package-that-doesnt-exist/).

## SS-5 — Model output rendered in an auto-interpreting surface without encoding/CSP/fetch-gating (XSS + image exfil)
- **Flags:** markdown/HTML in PR comments, chat, or email rendered without context-aware encoding, a
  CSP, or image-fetch gating — e.g. an auto-fetched remote `<img>`/`![](…)` URL in agent-authored text.
- **Why:** the render sink is XSS and exfiltration via auto-fetched images; agent-authored messages are
  a documented deferred-exfiltration channel ([LLM05 renderer row](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/);
  [agent-authored-message-rendered-image-exfiltration](https://agentpatterns.ai/security/agent-authored-message-rendered-image-exfiltration/)).
- **Fix:** context-aware HTML encoding; CSP; gate/strip outbound image fetches in the renderer
  (escape **at the sink**, not stream-stripping). Auditor: `network-egress-control` (rendered-image exfil).
  Remediation: [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

## SS-6 — File path built from LLM output without canonicalisation / base-dir constraint (traversal)
- **Flags:** an FS call (`open`/`write`/`read`) on a path interpolated from model output with no
  canonicalisation or base-directory constraint.
- **Why:** the file-path sink is path traversal ([LLM05 file-path row](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
- **Fix:** canonicalise and constrain to a base directory before any FS call.
  Remediation: [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

## SS-7 — Tool/RAG output fed back into a downstream LLM as trusted instructions (re-injection sink)
- **Flags:** untrusted tool/RAG output re-enters a model whose downstream **consequential actions
  are ungated** (no confirmation gate / allowlist on what the model can trigger next) and the
  untrusted span is not delimited from trusted instructions (spotlighting); or one LLM's output is
  piped into a downstream model's **system/instruction segment** rather than its data segment.
  *Not a flag:* an ordinary tool loop whose consequential actions are gated — tool results
  re-entering as data is how every ReAct loop works; Action-Selector is the strictest remediation,
  not a baseline requirement.
- **Why:** the downstream-LLM sink is re-injection — tool returns function as both data and
  instructions; spotlighting (delimiting trusted instructions from untrusted tool content) applies
  the separation, and the architectural fix is Action-Selector, where tool output never re-enters a
  model ([LLM05 downstream-LLM row](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/);
  [tool-invocation-attack-surface](https://agentpatterns.ai/security/tool-invocation-attack-surface/);
  [action-selector-pattern](https://agentpatterns.ai/security/action-selector-pattern/)).
- **Fix:** proportionate first — spotlight/delimit the untrusted span; move piped output out of the
  instruction segment into the data segment; gate consequential downstream actions
  (confirmation/allowlist). Strongest: LLM as intent decoder → deterministic executor
  (Action-Selector), or route writes through a validating MCP
  ([safe-outputs-pattern](https://agentpatterns.ai/security/safe-outputs-pattern/)).
  Remediation: [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

## SS-8 — LLM05 conflated with LLM06 (false-safe scoping)
- **Flags:** a permission-bounded agent assumed safe because "we limited the tools" — though its
  bounded actions still emit strings consumed unvalidated downstream.
- **Why:** OWASP draws the line: LLM06 is the agent *taking action*; LLM05 is a downstream consumer
  *mishandling the text it produced*. Bounding the **action** is not validating the **output** —
  orthogonal controls ([LLM05, *Distinction From LLM06*](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
- **Fix:** audit the output sink even on a permission-bounded path; don't accept tool-bounding as
  output validation. (Scoping check — prevents "we limited the tools, we're fine.")
  Remediation: [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/).

---

## SS-9 — Anti-theatre / false-positive guard (do NOT flag)
- **Do not fire when:**
  - **Per-sink controls already exist** — DB calls parameterised, HTML escaped, `eval` refused,
    install lockfile-gated. Confirm coverage of the new applicability surface; don't duplicate.
  - **No code-interpreting sink on the path** — an agent that drafts an email for human review or
    code a developer reads before commit has no LLM05 surface there; validation is theatre + latency.
  - **Structured outputs constrain the sink-reaching fields to bounded values** — fields typed as
    enum/`Literal`/clamped numerics and consumed by a deterministic executor leave the model no
    arbitrary string to smuggle; for those fields the schema *is* the validator. **This guard never
    covers free-text fields:** a schema-conformant `str` is still an arbitrary string — if it flows
    into a sink, fire the applicable sink check regardless of the schema — shell/SQL/render →
    SS-1/2/5, file path → SS-6, downstream LLM → SS-7.
  - **Lockfile-enforced / curated-mirror installs already in place** — `npm ci`/`uv pip sync` against
    a reviewed lockfile rejects the slopsquatted name before resolution; a second check is redundant.
- **Why:** mature teams gain nothing from a parallel LLM-specific scanner; output-stream stripping
  rejects legitimate code/links — prefer escape-at-sink ([LLM05](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)
  & [slopsquatting](https://agentpatterns.ai/security/slopsquatting-hallucinated-package-names/), *When This Backfires*).
- **Fix:** confirm the existing per-sink/lockfile/structured-output controls cover the surface, then
  don't fire — prefer escape-at-sink over a parallel scanner. Remediation:
  [learn — the-output-is-untrusted-too](https://learn.agentpatterns.ai/security/the-output-is-untrusted-too/)
  and [learn — the-package-that-doesnt-exist](https://learn.agentpatterns.ai/security/the-package-that-doesnt-exist/).
