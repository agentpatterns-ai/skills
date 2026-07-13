# audit-multi-agent-orchestration — detectors

Loaded on demand by `audit-multi-agent-orchestration`. Run per agent boundary, shared surface, gate,
and loop. Each item: ID / **Flags** / **Why** (cited) / **Fix** (enforced, not prompt). All detection
is read-only. Remediation links point to the matching `learn` lesson for step-order.

> **Ceremony guard (run first).** Structured handoffs and gates add overhead that is *not* always
> justified — they "pay off only when work genuinely crosses agent boundaries"
> ([agent-handoff-protocols, *When This Backfires*](https://agentpatterns.ai/multi-agent/agent-handoff-protocols/);
> [lead-teammate, solo-task backfire](https://agentpatterns.ai/multi-agent/lead-teammate-plan-approval-handshake/)).
> On a solo, single-stage, low-stakes task, a heavy schema + plan-approval gate is **ceremony, not a
> violation** — record it as a non-finding and do not demand more structure.

---

## Handoffs (MA-H*)

### MA-H1 — Untyped / prose handoff
- **Flags:** an upstream agent feeds a downstream agent with free prose and no declared output schema
  — no `## Returns`/`## Output` section, no JSON/YAML block, no `returns:`/`output_schema:`
  frontmatter. Field extraction then depends on the receiver parsing natural language.
- **Why:** structured handoffs make field extraction deterministic; "most multi-agent failures stem
  from missing structure at handoff points, not model capability," and every boundary should be
  validated — no untyped data crosses ([agent-handoff-protocols](https://agentpatterns.ai/multi-agent/agent-handoff-protocols/);
  [typed-schemas-at-agent-boundaries](https://agentpatterns.ai/multi-agent/typed-schemas-at-agent-boundaries/)).
- **Fix:** declare an explicit output schema at the boundary — at minimum the four fields
  done / found / needs-attention / unresolved
  ([learn: handoffs-and-contracts](https://learn.agentpatterns.ai/multi-agent/handoffs-and-contracts/)).

### MA-H2 — Raw-transcript forwarding
- **Flags:** a previous agent's full output / conversation transcript is passed as the next agent's
  prompt instead of distilled conclusions — inflates the receiver's context with reasoning it did not
  produce.
- **Why:** "Summarize, Don't Forward" — raw transcript forwarding is a named anti-pattern; the parent
  should receive only the final result ([agent-handoff-protocols](https://agentpatterns.ai/multi-agent/agent-handoff-protocols/)).
- **Fix:** pass conclusions, not transcripts; condense sub-agent output at the boundary
  ([learn: sub-agents-and-orchestration](https://learn.agentpatterns.ai/harness-engineering/sub-agents-and-orchestration/)).

---

## State & topology (MA-S*)

### MA-S1 — Shared mid-run state between workers
- **Flags:** workers coordinate or write a shared mutable surface (scratch file, shared doc,
  message-passing loop) *during* execution rather than only returning results to the orchestrator.
- **Why:** "Workers do not coordinate with each other … Workers returning results to the orchestrator
  is the only coordination point. **Any state sharing between workers during execution is a design
  smell**" — it couples workers and opens races on each other's side effects
  ([orchestrator-worker](https://agentpatterns.ai/multi-agent/orchestrator-worker/)).
- **Fix:** workers return to the orchestrator only; if a shared surface is unavoidable, assign
  non-overlapping file sets or serialize via the coordination point
  ([learn: the-orchestrator](https://learn.agentpatterns.ai/multi-agent/the-orchestrator/)).

### MA-S2 — Synthesis is concatenation, not reasoning
- **Flags:** the merge step concatenates/aggregates worker outputs — `"\n".join(...)`, a bare vote,
  or a summary that does **not** reconcile conflicts or weight reliability — instead of producing a
  unified result. (A synthesis step that summarizes *while reconciling* is the prescribed shape — do
  not flag the word "summary" alone.)
- **Why:** "Synthesis is not aggregation … If the orchestrator simply concatenates worker outputs,
  the pattern adds latency without improving quality" — and multi-agent already costs ~15x the tokens
  of chat ([orchestrator-worker](https://agentpatterns.ai/multi-agent/orchestrator-worker/)).
- **Fix:** make synthesis a reasoning step — reconcile conflicts, weight worker reliability, produce
  one result ([learn: fan-out-and-synthesis](https://learn.agentpatterns.ai/multi-agent/fan-out-and-synthesis/)).

### MA-S3 — Uniform fan-out (no engineered diversity → conformity bias)
- **Flags:** a fan-out/synthesis topology spawns N workers with **identical instructions AND
  identical context** and **no engineered spread** — no temperature variation, no per-worker
  seed/reference-context variation, no system-prompt-emphasis variation. The workers differ only in
  count, so the fan-out collapses to conformity rather than genuine diversity. (Independent contexts
  alone do not satisfy this — the page's own example varies temperature *and* prompt emphasis per
  agent.)
- **Why:** fan-out's value is ensemble variance — "The key condition is genuine diversity: if agents
  converge, there is nothing to exploit"; "Identical instructions do not guarantee identical outputs"
  so you must "maximize spread" by varying temperature, seed context, or system-prompt emphasis. Left
  uniform, **"conformity bias collapses diversity"** — agents given the same prompt converge on the
  same approach, paying N× compute for one effective sample
  ([fan-out-synthesis, *Why It Works / Diversity Mechanisms / When This Backfires*](https://agentpatterns.ai/multi-agent/fan-out-synthesis/)).
- **Fix:** engineer spread across the N workers — vary temperature between instances, give each a
  different seed/reference context, or differentiate system-prompt emphasis (brevity / robustness /
  edge-case coverage) — so synthesis has genuinely different approaches to exploit; otherwise drop to
  a single attempt and save the cost
  ([learn: fan-out-and-synthesis](https://learn.agentpatterns.ai/multi-agent/fan-out-and-synthesis/)).

---

## Gates & verification (MA-V*)

> **Discriminator (run first, mutually exclusive).** Search for a distinct critic/verifier
> **agent definition** — its own dedicated file/prompt (e.g. a `critic.md` alongside `worker.md`), not
> just a call that reuses the generic worker spawn path with a different role-name string or prompt
> argument. **An ad-hoc `spawn_worker("critic", ...)`-style call with no separate agent definition
> file does NOT count as a verifier existing** — it's the same producer machinery relabeled, still
> self-administered. **Nothing found ⇒ MA-V1**, full stop — regardless of how the producer's own
> completion call looks (`mark_done()`, a self-set status flag, a loop that only breaks on that
> relabeled call's verdict, etc.); there is no verifier to be mis-wired. **MA-V2 applies only when a
> genuinely separate verifier agent/role definition exists** and is wired wrong (its verdict isn't
> checked, it isn't independent, it's off the critical path, or it lacks precision evidence). Never
> label a no-dedicated-verifier-definition case MA-V2.

### MA-V1 — No independent verifier gate on "done" (producer self-declares)
- **Flags:** the agent that did the work also declares it complete; no separate read-only verifier on
  the critical path, no fail-closed default, no packetized admission record.
- **Why:** LLMs prefer their own generations when grading them and self-refinement amplifies the bias
  ([Xu et al. 2024, arXiv:2402.11436](https://arxiv.org/abs/2402.11436)); a separate read-only
  verifier on the critical path, with fail-closed defaults and packetized records, breaks the loop
  ([verify-gated-completion](https://agentpatterns.ai/multi-agent/verify-gated-completion-admission-control/)).
- **Fix:** add a read-only verifier as completion authority; ambiguous ⇒ reject; record each decision
  ([learn: verify-gated-completion](https://learn.agentpatterns.ai/multi-agent/verify-gated-completion/)).

### MA-V2 — Verifier present but mis-wired
- **Flags:** a gate exists but fails one of the four conditions — not independent of the producer
  (same model class admits the same hallucinations), sits as a sidecar **off the critical path**, has
  no ground truth, or is **enforcing with no discoverable precision evidence** — no eval record /
  blocked-precision metric in the repo *and* no documented reference to an external measurement
  (dashboard, eval-run id). Evidence anywhere documented passes; this is the statically-checkable
  form of "promoted without evidence". Also flag bypass paths (direct file writes around the gate). Do **not** pass it as
  "has a gate."
- **Why:** all four conditions must hold; the cited deployment had 98.58% rule agreement but only
  **0.39% blocked precision** — almost every rejection a false positive — so an enforcing gate without
  precision evidence blocks more valid work than invalid, and bypass routes make the gate a suggestion
  ([verify-gated-completion, *When This Applies/Backfires*](https://agentpatterns.ai/multi-agent/verify-gated-completion-admission-control/)).
- **Fix:** isolate the verifier's session from the producer's reasoning; keep it on-path; keep it
  advisory until blocked precision is measured; close bypass routes
  ([learn: verify-gated-completion](https://learn.agentpatterns.ai/multi-agent/verify-gated-completion/)).

### MA-V3 — No plan-approval gate before first write, or a prompt-level pseudo-gate
- **Flags:** in a lead-and-teammates topology, teammates begin editing before lead approval, OR the
  "gate" is a prompt instruction ("please get approval first") rather than enforced by the permission
  model (plan mode).
- **Why:** what makes it a gate not a suggestion is that plan mode is enforced by the permission model
  — the edit tools are structurally unavailable until the lead approves; prompt-level "get approval
  first" instructions are routinely violated under load
  ([lead-teammate-plan-approval-handshake](https://agentpatterns.ai/multi-agent/lead-teammate-plan-approval-handshake/)).
- **Fix:** hold the teammate in read-only plan mode until lead approval; put approval criteria in the
  lead's prompt ([learn: handoffs-and-contracts](https://learn.agentpatterns.ai/multi-agent/handoffs-and-contracts/)).

---

## Loops (MA-L*)

### MA-L1 — Unbounded refinement / debate loop
- **Flags:** a generator-critic (evaluator-optimizer) or debate loop with no hard round cap — exits
  only on PASS (`while True:`). Edge sub-check: the loop runs on an already-strong baseline.
- **Why:** Anthropic's reference implementation ships an unbounded `while True:` that only exits on
  PASS, so "production callers must impose their own cap … a starting limit of 3 is common"; and on a
  task the generator already scores ~98%, a self-critique loop dropped accuracy to ~57%
  ([evaluator-optimizer](https://agentpatterns.ai/agent-design/evaluator-optimizer/)).
- **Fix:** impose a hard `max_rounds` (start at 3) with deadlock detection; skip the loop entirely on
  a strong baseline ([learn: evaluator-optimizer](https://learn.agentpatterns.ai/harness-engineering/evaluator-optimizer/)).

---

## Cost & scaling (MA-C*)

### MA-C1 — Hard-coded worker count / no effort-scaling in the prompt (over-spawning)
- **Flags:** a fixed agent count baked into harness code, or no effort-scaling heuristics in the
  orchestrator's *prompt* — the most common failure is over-spawning on simple queries. *(No tension
  with MA-L1: that check requires a code-level **max cap** — an upper bound; the smell here is a
  code-level **fixed count** — an exact number that ignores task complexity.)*
- **Why:** effort-scaling rules (1 simple / 2-4 medium / 10+ complex) "belong in the orchestrator's
  system prompt, not in code"; without effort boundaries a system "spawned 50 subagents for simple
  queries," and multi-agent costs ~15x tokens so the prompt's effort budget is the primary cost
  control ([orchestrator-worker](https://agentpatterns.ai/multi-agent/orchestrator-worker/);
  [emergent-behavior-sensitivity](https://agentpatterns.ai/multi-agent/emergent-behavior-sensitivity/)).
- **Fix:** move effort-scaling heuristics into the orchestrator prompt as a framework (principles +
  budgets), not a fixed count in code
  ([learn: the-orchestrator](https://learn.agentpatterns.ai/multi-agent/the-orchestrator/)).
