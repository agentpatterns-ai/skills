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
drivers of debate quality** ([arXiv:2601.19921](https://arxiv.org/pdf/2601.19921)). A panel where
voices think alike adds rounds without adding signal. Prefer **breadth of persona over depth of
iteration** ([arXiv:2502.10858](https://arxiv.org/abs/2502.10858)): 5–7 genuinely-distinct voices
beats 3 voices arguing for 6 rounds.

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

1. **Round 0 — blind opinions.** Each persona, in its **own separate context**, proposes an approach
   without seeing the others. This structural independence (real fan-out, not one agent role-playing
   a panel) is what produces genuine spread; shared context anchors every voice to the first one
   stated ([Du et al. 2023, arXiv:2305.14325](https://arxiv.org/abs/2305.14325)). Sample this step at
   a **higher / entropy-aware temperature** for divergence ([EDT, arXiv:2403.14541](https://arxiv.org/abs/2403.14541)).
2. **Steelman pass.** Each persona writes the strongest case for its own pick, then the strongest
   case for **the opposite** of its pick — forcing escape from a single framing
   ([Exchange-of-Perspective, arXiv:2506.03573](https://arxiv.org/abs/2506.03573)). Each persona must
   **reframe** the spec from its lens, not merely re-answer the question.
3. **Rounds 1–3 — cross-examination.** Aggregate all opinions; each persona challenges its own and
   others' assumptions, naming the suspected bias from the catalog. Stop at round 3, or earlier once
   opinions stop changing — convergence typically lands ≤3 rounds ([Du 2023](https://arxiv.org/abs/2305.14325)).
4. **Converge at low temperature** for a disciplined synthesis ([EDT](https://arxiv.org/abs/2403.14541)).

---

## Convergence guard (run before presenting)

Convergence alone is not the goal — *diverse* convergence is. Before writing the recommendation:

- **Weigh cross-persona agreement, not a majority vote.** A point reached **independently by
  several distinct personas** is signal; the same point echoed by voices that share a prior is not
  ([Multi-Perspective Self-Consistency, arXiv:2309.17272](https://arxiv.org/abs/2309.17272)).
- **Reject rhetorical wins.** If an option leads only because one persona argued it forcefully —
  not because it fits the binding constraints — it has not won. Re-test it against the
  steelman-of-the-opposite.
- **Surface the unresolved.** If the right answer hinges on an unknown (expected load, team size,
  compliance), name that dependency in the output instead of papering over it with consensus.
- **Never collapse to a silent pick.** The output is the option set + the conditions each wins
  under; the human commits. (Skill `Stance`.)
