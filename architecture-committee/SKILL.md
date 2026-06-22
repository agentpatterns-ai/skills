---
name: architecture-committee
description: Convene a debiased multi-persona committee to vet an architecture / design decision for a feature, spec, or PRD — distinct personas independently propose, steelman their pick AND its opposite, cross-examine across capped debate rounds, then converge on the approach that fits THIS spec rather than the model's default priors. Invoke when asked to "form/convene a committee", weigh architecture or technology options, or pressure-test a design choice before committing. Skip when you are auditing or detecting problems in an existing instruction file / config (use audit-instruction-file or lethal-trifecta-audit), compressing a prompt (use compress-prompt), or when the decision is trivial / already made and only execution remains.
user-invocable: true
version: "0.1.0"
usage: /architecture-committee [goal | spec | PRD | feature description]
---

# Architecture Committee

A single agent's architecture recommendation inherits that agent's built-in priors — reflexive
reaches for familiar frameworks, languages, and "default" solutions regardless of fit. The committee
structure exists to **surface and neutralise those biases** so the recommendation follows from the
spec, not from what is popular in the training data. The mechanism is empirically backed: *N*
independent instances that answer, then read and critique each other and revise, converge to better
answers — typically within **≤3 rounds** — with real gains in reasoning and factual validity
([Du et al. 2023, arXiv:2305.14325](https://arxiv.org/abs/2305.14325), "society of minds"). The
active ingredient is **diversity, not repetition**: group diversity and intrinsic reasoning strength
are the dominant drivers of debate success ([arXiv:2601.19921](https://arxiv.org/pdf/2601.19921)).

**Stance — the committee vets; the human decides. It never silently picks.** Output is advisory:
viable options, each with its steelman, its strongest counter, the assumptions it rests on, and the
conditions under which it wins or breaks. Three guardrails are structural, not requests:
- **Genuine independence, deterministically.** Generate each persona's first opinion in a *separate,
  blind context* (sub-agent fan-out), not by asking one agent to role-play disagreement — shared
  context anchors and collapses the diversity that does the work ([Du 2023](https://arxiv.org/abs/2305.14325)).
- **Cap the rounds.** ≤3 debate rounds; deeper single-thread iteration plateaus while *breadth*
  (more diverse personas) keeps paying ([arXiv:2502.10858](https://arxiv.org/abs/2502.10858)). Spend
  budget on personas, not rounds.
- **Diverse convergence only.** Two agents that think alike agreeing tells you nothing; weigh
  cross-persona agreement, and reject a "winner" that won on rhetoric rather than fit (below).

## Input
- A goal, PRD, spec, or loose feature description — what is to be built and any hard constraints.
- Optional: a roster hint (e.g. "include a security and a cost voice"). Absent that, the default
  panel in [`committee.md`](committee.md) is used and adapted to the spec.

## Scope
Decides **how to build** a stated thing: architecture, technology, framework, and approach trade-offs.
**Out of scope:** auditing an existing instruction file or harness (use `audit-instruction-file` /
`lethal-trifecta-audit`), compressing a prompt (`compress-prompt`), writing the implementation, and
making the final call — the committee recommends; it does not commit.

## Procedure (the vetting loop)
1. **Convene personas** — pick distinct, opinionated viewpoints the decision actually turns on
   (pragmatist/ship-fast, scale-ops, simplicity-minimalist, security, cost, long-term-maintainer,
   domain-expert). Roster adapts to the spec; default panel in `committee.md` is the fallback.
2. **Independent opinions** — each persona proposes an approach **blind to the others**, in its own
   context, to avoid anchoring and preserve diversity ([Du 2023](https://arxiv.org/abs/2305.14325)).
   Sample this divergent step at a higher / entropy-aware temperature for genuine spread, then
   converge at low temperature ([EDT, arXiv:2403.14541](https://arxiv.org/abs/2403.14541)).
3. **Steelman own choice** — each makes the strongest good-faith case for its pick.
4. **Steelman the opposite** — each *also* makes the strongest case for the opposite of its own
   recommendation, to break the fixed mindset of one framing ([Exchange-of-Perspective, arXiv:2506.03573](https://arxiv.org/abs/2506.03573)).
5. **Cross-examine (≤3 rounds)** — aggregate the opinions; personas challenge **both their own and
   each other's** assumptions, naming the bias each suspects (framework affinity, novelty bias,
   resume-driven design — catalog in `committee.md`).
6. **Converge** — discuss toward the approach that best fits *this* spec, weighing cross-persona
   agreement, not a majority vote ([MPSC, arXiv:2309.17272](https://arxiv.org/abs/2309.17272)).
   Apply the convergence guard before presenting.

## Personas, biases & the convergence guard
The default persona panel, the named-bias catalog, the debate-round protocol, and the
rhetoric-vs-fit convergence guard live in [`committee.md`](committee.md) — load it when convening.

## Output template
```
# Architecture committee — <decision>

## Panel & spec read
Personas convened: <pragmatist, scale-ops, security, …> — and why this spec needs them.
Constraints that bind the decision: <latency, team size, budget, compliance, …>

## Options on the table
| Option | Champion(s) | Wins when… | Breaks when… | Key assumption |
|---|---|---|---|---|
| Modular monolith | pragmatist, maintainer | small team, unproven domain | >N services need independent scaling | traffic stays single-region |
| Event-driven microservices | scale-ops | load is spiky & partitionable | team can't staff the ops surface | org can run distributed tracing |

## Per-option dossier
### Modular monolith
- **Steelman:** <strongest good-faith case.>
- **Strongest counter:** <the best argument against — its own steelman-of-the-opposite.>
- **Rests on:** <explicit assumptions.>  **Bias check:** <e.g. is this resume-driven? recency?>

## Where the committee converged (and where it didn't)
Diverse agreement: <points >1 distinct persona reached independently.>
Unresolved / depends-on: <the open question whose answer flips the recommendation.>

## Recommendation to the human (advisory)
Best fit for THIS spec: <option> — because <fit to the binding constraints>. Conditions that would
change it: <…>. **The committee does not commit this; you make the call.**
```

## Principles (the cited why)
- **Debate beats a lone pass** — independent instances catch each other's errors and converge ≤3
  rounds ([Du 2023](https://arxiv.org/abs/2305.14325)); the multi-agent generalisation of
  self-consistency ([Wang et al. 2022, arXiv:2203.11171](https://arxiv.org/abs/2203.11171)).
- **Diversity is the lever, not round count** — diversity + reasoning strength dominate
  ([arXiv:2601.19921](https://arxiv.org/pdf/2601.19921)), breadth > depth ([arXiv:2502.10858](https://arxiv.org/abs/2502.10858)), and each persona must *reframe* the spec, not restate it ([EoP, arXiv:2506.03573](https://arxiv.org/abs/2506.03573)) — convergence then weighs cross-perspective agreement ([MPSC, arXiv:2309.17272](https://arxiv.org/abs/2309.17272)).

## Critical rules (read last)
- **The human makes the final call** — present vetted options; never silently pick or auto-commit.
- **Independence is structural** — blind, separate contexts; role-played disagreement in one context
  is not a substitute and collapses the diversity that is the whole point.
- **Cap at 3 rounds; spend budget on diverse personas.** Reject a winner that won on rhetorical
  strength rather than fit to the spec.
