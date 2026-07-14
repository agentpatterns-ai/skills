---
name: audit-supply-chain-sinks
description: Audit an agent harness's output boundary and dependency-install authority — every place an LLM-emitted string crosses into a code-interpreting sink (shell/exec/eval, SQL, HTML/markdown render, file path, package manager, a downstream LLM) and every place the agent can install an agent-named dependency — and flag the unvalidated sinks (OWASP LLM05) and slopsquatting install routes. Invoke when reviewing or hardening an agent that can execute generated code, run shell, build SQL, render model output, write files, or run package installs — including auto-running package installs inside a sandbox — or before granting an agent code-execution or install authority. Skip when auditing input-side data-flow architecture — whether one path co-holds private-data + untrusted-input + egress (use audit-lethal-trifecta), credential/secret hygiene in context (use audit-secret-exposure), or bounding what an already-granted action can destroy — sandbox, reversibility, kill path (use audit-harness-safety).
user-invocable: true
version: "0.3.1"
usage: /audit-supply-chain-sinks [path-to-agent-config-or-repo]
---

# Audit Supply-Chain Sinks

Trust does not transfer across a string boundary. Input validation checked the user prompt, not the
model response — so an LLM-emitted string reaching a **code-interpreting sink** is unscrutinised text.
OWASP **LLM05:2025**: validate output **per sink**, at each downstream consumer
([improper-output-handling-downstream-sinks](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
The package-manager sink adds a twist: coding LLMs hallucinate package names (5.2–21.7%; 43%
persist, so they are enumerable and pre-registerable), so an ungated `uv add`/`npm install` of an
agent-named dependency is **slopsquatting**
([slopsquatting-hallucinated-package-names](https://agentpatterns.ai/security/slopsquatting-hallucinated-package-names/);
[Spracklen et al., USENIX Security 2025, arXiv:2406.10279](https://arxiv.org/abs/2406.10279)).

**Stance — detect and recommend; read-only; apply nothing without confirmation.**
- **Gate the sink, not the model.** Fix with a per-sink control (parameterise, escape-at-sink,
  command allowlist, lockfile-enforced install) — a deterministic boundary the model can't override,
  not a prompt asking it to emit safe strings ([LLM05](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
- **Escape at the sink, never strip the stream.** Context-aware encoding at the consumer beats
  pattern-stripping the LLM output, which rejects legitimate code/links ([LLM05, *When This Backfires*](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).
- **Gate install authority, fail closed.** Lockfile-enforced resolution (`npm ci`, `uv pip sync`,
  `pip-compile --generate-hashes`) refuses any name the lockfile doesn't endorse — not edit-distance
  typosquat detection, which misses 48.6% of hallucinated names ([slopsquatting](https://agentpatterns.ai/security/slopsquatting-hallucinated-package-names/)).

## Input
- `path` (optional): an agent-config dir or repo. Default: scan the harness for sinks — tool-executor
  code, command/shell allowlists, SQL/ORM construction sites, render/template/email config (CSP,
  image-fetch gating), file-path construction, install hooks (`uv add`/`npm install`/`pip install`),
  lockfile presence + CI manifest gate, and any structured-output (Pydantic/JSON-Schema) boundary.

## Scope
Audits the **output boundary** (does each downstream consumer of LLM output validate?) and
**install authority** (is dependency install lockfile-gated?). **Out of scope:** input-side data-flow —
whether a path co-holds private-data + untrusted-input + egress (use
`audit-lethal-trifecta`); cross-session memory poisoning (Trojan Hippo) and retrieval authorization
(separate audits); credential/secret hygiene in context (use `audit-secret-exposure`); resource/cost
bounds (use `audit-harness-safety`). A path can be trifecta-safe and still fail LLM05 — a
permission-bounded agent emitting unvalidated SQL.

## Procedure
1. **Enumerate sinks.** Per execution path, list every place an LLM-emitted string is consumed:
   shell/`exec`/`eval`, SQL, render surface, file path, package manager, downstream LLM.
   Done when every execution path has its sink list recorded — no blanks.
2. **Classify** each sink via the detectors in [`checks.md`](checks.md): is there a per-sink control,
   or does raw model output reach the interpreter? Mark the install hook against lockfile presence.
   Done when every sink has a per-detector verdict recorded.
3. **Flag** every ungated sink and every ungated install leg. Apply the anti-theatre guard (SS-9):
   do **not** flag a path with no code-interpreting sink (draft-for-human-review), a sink already
   parameterised/escaped, or installs already lockfile-gated — confirm, don't duplicate.
   Done when every sink is flagged or confirmed clean — none skipped.
4. **Recommend** the per-sink deterministic control (fix column in `checks.md`), preferring
   escape-at-sink over stream-stripping, and a schema-constrained object → deterministic executor over
   the model writing the sink string. Remediation lessons resolve per check ID via
   [`checks.md`](checks.md) — the links' canonical home.
   Done when every finding names its per-sink control.
5. **Report** with the template below.
   Done when the filled template covers every finding.

## Detectors
Per-sink detectors (SS-1 RCE, SS-2 SQLi, SS-3 ungated install, SS-4 mis-targeted typosquat control,
SS-5 render/XSS+image-exfil, SS-6 path traversal, SS-7 downstream-LLM re-injection, SS-8 LLM05↔LLM06
false-safe) plus the anti-theatre guard (SS-9) live in [`checks.md`](checks.md) — load when auditing.

## Worked example (SS-2, the SQL sink)
**Before — model writes the WHERE clause (CVE-2025-1793 class):**
`cursor.execute(f"SELECT * FROM docs WHERE {llm_output}")` → a prompt rephrased to
`1=1; DROP TABLE docs --` executes. **Finding:** High, SQL sink ungated.
**After (the recommended fix):** the LLM emits a schema-constrained `Filter` object (Pydantic
`Literal` fields); a deterministic executor builds `execute("… WHERE %s %s %s", (f.field, f.op,
f.value))`. The model never writes SQL ([LLM05 example](https://agentpatterns.ai/security/improper-output-handling-downstream-sinks/)).

## Output template
```
# Supply-chain sink audit — <target>

## Sinks
| Path | Sink | LLM output reaches it? | Per-sink control | Verdict |
|---|---|:--:|---|:--:|
| query-agent  | SQL (cursor.execute) | Yes (f-string)      | none              | UNGATED |
| query-agent  | package install      | Yes (`uv add`)      | no lockfile        | UNGATED |
| query-agent  | render (PR comment)  | Yes (markdown)      | CSP + escape@sink  | gated   |

## Findings
| Check | Severity | Path | Sink | Why (source) | Fix (deterministic) |
|---|---|---|---|---|---|
| SS-2 | High | query-agent | SQL string-concat | CVE-2025-1793 SQLi class — model writes the WHERE clause [LLM05] | schema object (Pydantic) → parameterised `execute(sql, params)` |
| SS-3 | High | query-agent | ungated `uv add`  | slopsquatting: agent-named dep, no lockfile [arXiv:2406.10279] | `uv pip sync`/`npm ci` against committed lockfile; manifest change via reviewed PR |

**Gated sinks:** <which, and the control that holds them.>
**Smallest high-impact change:** <the one ungated sink to close first.>
```
Severity: **High** = raw model output reaches an executing sink (RCE/SQLi) or an ungated install leg;
**Medium** = render/path sink without encoding, or a mis-targeted control (SS-4); **Low** =
least-privilege hardening where no live sink is exposed.

**Findings → backlog (default).** After the report, **offer** to file the findings as one tracking issue in your backlog tracker (issue tracker) — title `<skill-name>: <one-line>`, label `enhancement`, body = the findings table; interactive: confirm first (never auto-file); autonomous: self-file. Each finding carries its **Fix → lesson** link in both the report and the filed issue, resolved by check ID via [`checks.md`](checks.md).

## Critical rules (read last)
- An ungated code-interpreting sink needs a **deterministic per-sink control** — a prompt asking the
  model to emit safe strings is not a fix.
- **Escape at the sink; gate install authority with a lockfile (fail closed)** — not stream-stripping,
  not edit-distance typosquat detection (misses 48.6%).
- Read-only: recommend, apply nothing without confirmation. A trifecta-safe path can still fail LLM05.
