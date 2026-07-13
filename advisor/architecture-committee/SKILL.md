---
name: architecture-committee
description: Convene a debiased multi-persona committee to vet an architecture / design decision for a feature, spec, or PRD — distinct personas independently propose, steelman their pick AND its opposite, cross-examine across capped debate rounds, then converge on the approach that fits THIS spec rather than the model's default priors. Invoke when asked to "form/convene a committee", weigh architecture or technology options, or pressure-test a design choice before committing. Skip when you are auditing or detecting problems in an existing instruction file / config (use audit-instruction-file or audit-lethal-trifecta), compressing a prompt (use compress-prompt), or when the decision is trivial / already made and only execution remains.
user-invocable: true
version: "0.3.0"
usage: /architecture-committee [goal | spec | PRD | feature description]
---

# Architecture Committee

A lone agent's architecture pick inherits its built-in priors — reflexive reaches for familiar
frameworks, languages, "default" solutions regardless of fit. The committee **surfaces and
neutralises those biases** so the recommendation follows from the spec, not from what's popular in
training data. Empirically backed for **high-stakes, hard-to-reverse decisions**: *N* independent
instances answer, then critique each other and revise, converging — typically **≤3 rounds** — with
gains in reasoning and factual validity ([Du et al. 2023, arXiv:2305.14325](https://arxiv.org/abs/2305.14325), "society of
minds"); at equal compute on routine calls, debate often does **not** beat a strong single pass, and
majority pressure can converge agents on a confident wrong answer ([opponent-processor-debate,
*Empirical caveats*](https://agentpatterns.ai/multi-agent/opponent-processor-debate/)) — hence the Skip-when boundary. The active ingredient is **diversity, not repetition**: group diversity and intrinsic
reasoning strength dominate debate success ([arXiv:2601.19921](https://arxiv.org/pdf/2601.19921)),
and independent attempts explore trade-offs no single pass reaches
([fan-out synthesis](https://agentpatterns.ai/multi-agent/fan-out-synthesis/)).

**Stance — the committee vets; the human decides. It never silently picks.** Output is advisory:
viable options, each with its steelman, its strongest counter, the assumptions it rests on, and when
it wins or breaks. Three guardrails are structural, not requests:
- **Genuine independence, deterministically.** Generate each persona's first opinion in a *separate,
  blind sub-agent context*, not by role-playing disagreement in one agent — a shared context lets
  the first stated framing anchor every later voice
  ([opponent-processor debate](https://agentpatterns.ai/multi-agent/opponent-processor-debate/);
  [Du 2023](https://arxiv.org/abs/2305.14325)).
- **Cap the rounds.** ≤3 debate rounds; deeper single-thread iteration plateaus while *breadth*
  (more diverse personas) keeps paying ([arXiv:2502.10858](https://arxiv.org/abs/2502.10858)). Spend
  budget on personas, not rounds.
- **Diverse convergence only.** Two agents that think alike agreeing tells you nothing; weigh
  cross-persona agreement, and reject a "winner" that won on rhetoric rather than fit.

## Input
- A goal, PRD, spec, or loose feature description — what to build and any hard constraints.
- Optional: a roster hint (e.g. "include a security and a cost voice"). Absent that, the default
  panel in [`committee.md`](committee.md) applies, adapted to the spec.

## Scope
Decides **how to build** a stated thing: architecture, technology, framework, approach trade-offs.
**Out of scope:** auditing an existing instruction file or harness (use `audit-instruction-file` /
`audit-lethal-trifecta`), compressing a prompt (`compress-prompt`), writing the implementation, and
making the final call — the committee recommends; it does not commit.

## Procedure (the vetting loop)
The full protocol — persona table, named-bias catalog, round mechanics, convergence guard — lives in
[`committee.md`](committee.md); load it when convening. The operational sequence:

1. **Convene personas** — pick the distinct, opinionated viewpoints this decision turns on;
   `committee.md`'s default panel is the fallback. *Done when 5–7 genuinely-distinct voices are
   named, each optimising for a different constraint the spec actually has.*
2. **Blind independent proposals** — fan out one sub-agent per persona, each prompted with a
   deliberately divergent persona framing, none seeing another's output (protocol Round 0).
   *Done when every persona has proposed an approach produced in its own separate context.*
3. **Steelman both ways** — each persona writes the strongest good-faith case for its own pick
   **and** for the opposite of its pick (protocol steelman pass). *Done when every dossier carries
   both cases and the opposite-case reframes the spec rather than re-answering it.*
4. **Cross-examine (≤3 rounds)** — aggregate the opinions; personas challenge their own and each
   other's assumptions, naming the suspected bias from `committee.md`'s catalog. *Done when
   opinions stop changing between rounds or the 3-round cap hits — whichever comes first.*
5. **Converge** — one synthesizer assembles the recommendation from the proposals on the table
   (no new options), then runs `committee.md`'s convergence guard. *Done when the guard passes:
   diverse agreement weighed over vote-count, rhetorical wins rejected, unresolved dependencies
   named.*
6. **Present** — fill the output template below. *Done when every option carries its steelman,
   strongest counter, assumptions, and win/break conditions — and the final call is explicitly
   left to the human.*

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

## Related / pairing (the cited why)
- **Siblings:** detecting problems in an existing file/config → `audit-instruction-file` /
  `audit-lethal-trifecta`; compressing a prompt → `compress-prompt` (this skill only *decides*).
- **Debate beats a lone pass** (on high-stakes calls — see the intro's equal-compute caveat) —
  independent instances catch each other's errors and converge ≤3
  rounds ([Du 2023](https://arxiv.org/abs/2305.14325)); the multi-agent generalisation of
  self-consistency ([Wang et al. 2022, arXiv:2203.11171](https://arxiv.org/abs/2203.11171)).
- **Diversity is the lever, not round count** — diversity + reasoning strength dominate
  ([arXiv:2601.19921](https://arxiv.org/pdf/2601.19921)), breadth > depth ([arXiv:2502.10858](https://arxiv.org/abs/2502.10858)), and each persona must *reframe* the spec, not restate it ([EoP, arXiv:2506.03573](https://arxiv.org/abs/2506.03573)) — convergence then weighs cross-perspective agreement, where independent sources agreeing is the confidence signal ([MPSC, arXiv:2309.17272](https://arxiv.org/abs/2309.17272); [multi-model plan synthesis](https://agentpatterns.ai/multi-agent/multi-model-plan-synthesis/)).

## Critical rules (read last)
- **The human makes the final call** — present vetted options; never silently pick or auto-commit.
- **Independence is structural** — blind, separate contexts; role-played disagreement in one context
  is not a substitute and collapses the diversity that is the whole point.
- **Cap at 3 rounds; spend budget on diverse personas.** Reject a winner that won on rhetorical
  strength rather than fit to the spec.
