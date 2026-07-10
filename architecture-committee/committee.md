# architecture-committee — panel, biases & debate protocol

Loaded on demand by `architecture-committee`. The roster, the bias catalog, the round protocol, and
the convergence guard — each grounded in the same primary sources as the skill body.

---

## Default persona panel (adapt to the spec; drop/add voices the decision turns on)

| Persona | Optimises for | Reflex to watch |
|---|---|---|
| **Pragmatist / ship-fast** | time-to-first-value, reversibility | under-weights scale & ops debt |
| **Scale-ops** | throughput, failure isolation, observability | over-engineers for load that may never come |
| **Simplicity-minimalist** | fewest moving parts, least new tech | dismisses genuinely-needed capability |
| **Security** | attack surface, data boundaries, least privilege | blocks on risks cheaper to accept |
| **Cost** | infra + licensing + headcount spend | optimises pennies, ignores opportunity cost |
| **Long-term maintainer** | clarity, on-call load, upgrade path | resists change that pays off later |
| **Domain expert** | fit to the actual problem & users | over-fits to one prior project |

The panel is the diversity engine — **diversity + intrinsic reasoning strength are the dominant
drivers of debate quality** ([arXiv:2601.19921](https://arxiv.org/pdf/2601.19921)), and role framing
is what shifts which risks a voice surfaces
([committee review pattern](https://agentpatterns.ai/code-review/committee-review-pattern/)). A panel
where voices think alike adds rounds without adding signal; a committee is justified only because the
decision needs **multiple independent directions at once**
([when many agents beat one](https://learn.agentpatterns.ai/multi-agent/when-many-agents/)). Prefer
**breadth of persona over depth of iteration**
([arXiv:2502.10858](https://arxiv.org/abs/2502.10858)). **Composition is done when the panel holds
5–7 genuinely-distinct voices, each optimising for a different constraint the spec actually has** —
that beats 3 voices arguing for 6 rounds.

---

## Named-bias catalog (each persona calls out the bias it suspects, in cross-examination)

| Bias | Symptom in an opinion | Probe that exposes it |
|---|---|---|
| **Framework affinity** | reaches for a named framework before the constraints are stated | "What does the spec demand that *only* this provides?" |
| **Novelty bias** | favours the newest tech for its own sake | "Name a boring tool that meets the same constraint." |
| **Resume-driven design** | picks what is impressive to have built | "Who pays the operational cost, and do they want it?" |
| **Recency bias** | over-weights the last project / latest blog post | "Is this *this* problem, or the last one you saw?" |
| **Sunk-cost / familiarity** | defends the incumbent because it exists | "If we were greenfield today, would we pick it?" |
| **Default / popularity** | "everyone uses X" stands in for a reason | "Cite the constraint, not the popularity." |

These are the failure modes the committee exists to neutralise: a lone agent's recommendation is a
sample from its training-data priors; the debate's job is to make each prior *argue for itself
against an equally-strong opposite* and lose if it cannot.

---

## Debate-round protocol (≤3 rounds — hard cap)

1. **Round 0 — blind proposals.** Each persona, in its **own separate sub-agent context**, proposes
   an approach without seeing the others. This structural independence (real fan-out, not one agent
   role-playing a panel) is what produces genuine spread; a shared context lets the first stated
   framing anchor every later voice — the main source of groupthink in correlated-context setups
   ([opponent-processor debate](https://agentpatterns.ai/multi-agent/opponent-processor-debate/);
   [Du et al. 2023, arXiv:2305.14325](https://arxiv.org/abs/2305.14325)). **Diverge in the prompts,
   not a sampling knob** — an invoking agent has no temperature control over itself or its
   sub-agents; of the page's three documented diversity mechanisms (temperature, seed context, prompt
   emphasis), the one always available here is varied system-prompt emphasis: give each sub-agent a
   deliberately divergent persona framing and optimisation target (one voice for simplicity, one
   for failure isolation, one for cost, …), which realises the "diverge hot" half of
   diverge-hot/converge-cold ([EDT, arXiv:2403.14541](https://arxiv.org/abs/2403.14541))
   structurally ([fan-out synthesis](https://agentpatterns.ai/multi-agent/fan-out-synthesis/);
   [fan-out and synthesis lesson](https://learn.agentpatterns.ai/multi-agent/fan-out-and-synthesis/)).
   *Done when every persona has a proposal produced blind — no persona saw another's output.*
2. **Steelman pass.** Each persona writes the strongest case for its own pick, then the strongest
   case for **the opposite** of its pick — forcing escape from a single framing
   ([Exchange-of-Perspective, arXiv:2506.03573](https://arxiv.org/abs/2506.03573)). Arguing the
   strongest opposing case is role-work, not mood: an agent told to defend a decision surfaces
   different evidence than one told to challenge it
   ([opponent-processor debate](https://agentpatterns.ai/multi-agent/opponent-processor-debate/)).
   Each persona must **reframe** the spec from its lens, not merely re-answer the question.
   *Done when every dossier carries both cases.*
3. **Rounds 1–3 — cross-examination.** Aggregate all opinions; each persona challenges its own and
   others' assumptions, naming the suspected bias from the catalog. **Stop when opinions stop
   changing between rounds, or when the round-3 cap hits — whichever comes first.** Shrinking
   between-round diffs are the mechanical stop signal
   ([convergence detection](https://agentpatterns.ai/loop-engineering/convergence-detection/)); the
   hard cap is the fallback — two-to-three rounds covers most cases, and unbounded debate deadlocks
   or drifts rather than improving
   ([committee review pattern](https://agentpatterns.ai/code-review/committee-review-pattern/);
   [opponent-processor debate](https://agentpatterns.ai/multi-agent/opponent-processor-debate/);
   [why multi-agent systems fail](https://learn.agentpatterns.ai/multi-agent/why-multi-agent-fails/);
   convergence typically lands ≤3 rounds — [Du 2023](https://arxiv.org/abs/2305.14325)). If the cap
   hits with live disagreement, carry it into the output as *unresolved* — do not force consensus.
4. **Converge — strictly, via one synthesizer.** The "converge cold" half of
   diverge-hot/converge-cold ([EDT](https://arxiv.org/abs/2403.14541)), realised structurally: a
   **single synthesis step** that scores, selects, and merges **only from the proposals on the
   table** — deliberate assembly with per-decision attribution, never a vague summary and never a
   new proposal ([fan-out synthesis](https://agentpatterns.ai/multi-agent/fan-out-synthesis/)).
   Then run the convergence guard below.

---

## Convergence guard (run before presenting)

Convergence alone is not the goal — *diverse* convergence is. Before writing the recommendation:

- **Weigh cross-persona agreement, not a majority vote.** A point reached **independently by
  several distinct personas** is signal; the same point echoed by voices that share a prior is not
  ([Multi-Perspective Self-Consistency, arXiv:2309.17272](https://arxiv.org/abs/2309.17272)).
  Agreement across diverse sources raises confidence precisely because each source was likely to
  make different errors; disagreement marks the trade-offs worth surfacing, not noise to average away
  ([multi-model plan synthesis](https://agentpatterns.ai/multi-agent/multi-model-plan-synthesis/)).
  *Caveat:* same-model personas are **correlated** sources — that page's own backfire warning — so
  persona-level decorrelation rests on the role-shift result
  ([committee-review-pattern](https://agentpatterns.ai/code-review/committee-review-pattern/)); treat
  cross-persona agreement as moderate confidence, not the cross-model signal.
- **Reject rhetorical wins.** If an option leads only because one persona argued it forcefully —
  not because it fits the binding constraints — it has not won. Re-test it against the
  steelman-of-the-opposite.
- **Surface the unresolved.** If the right answer hinges on an unknown (expected load, team size,
  compliance), name that dependency in the output instead of papering over it with consensus.
- **Never collapse to a silent pick.** The output is the option set + the conditions each wins
  under; the human commits. (Skill `Stance`.)
