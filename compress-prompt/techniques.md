# compress-prompt — techniques catalog & worked example

Loaded on demand by `compress-prompt` (the SKILL.md *Techniques* summary points here). The full
catalog with when-to-use, plus a worked before/after that shows ceremony cut while every constraint
survives.

## Techniques catalog

| Technique | What it does | When |
|-----------|--------------|------|
| **Tables over prose** | Rows carry contrast (do/don't, in/out) with zero explanation overhead | Any set of parallel cases or paired rules ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); Remediation: [learn — signal-per-token](https://learn.agentpatterns.ai/context-engineering/signal-per-token/)) |
| **Bullets over sentences** | One idea per line; drops transitional/connective filler | Lists of rules or steps buried in paragraphs ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); Remediation: [learn — signal-per-token](https://learn.agentpatterns.ai/context-engineering/signal-per-token/)) |
| **Rules over explanations** | State the rule; omit the *why* unless compliance depends on it | "It is important that you ensure X" → `Do X` ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); Remediation: [learn — signal-per-token](https://learn.agentpatterns.ai/context-engineering/signal-per-token/)) |
| **Negative constraints** | "Never X" is cheaper and harder to misread than enumerating valid options; greppable | Known failure modes with a clear boundary ([negative-space-instructions](https://agentpatterns.ai/instructions/negative-space-instructions/); Remediation: [learn — guardrails-beat-guidance](https://learn.agentpatterns.ai/prompt-engineering/guardrails-beat-guidance/)) |
| **Examples over descriptions** | One example collapses a pattern + its application into imitable text | Style/format guidance that's easier shown than described ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); Remediation: [learn — rules-or-examples](https://learn.agentpatterns.ai/prompt-engineering/rules-or-examples/)) |
| **Name, don't define** | A canonical name (`semver`, `Conventional Commits`) carries its full definition — replace a re-definition of a concept the model already knows with the bare name | In-prompt definitions of standard concepts/conventions — only where local usage matches the standard meaning; a project *re*definition ("a 'release' here means…") is a constraint, keep it ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); Remediation: [learn — signal-per-token](https://learn.agentpatterns.ai/context-engineering/signal-per-token/)) |
| **Front-load critical rules** | Highest-cost-to-violate rules first; attention is U-shaped, the middle is weak | Any file with a real dead middle ([prompt-compression](https://agentpatterns.ai/context-engineering/prompt-compression/); [Liu et al. 2023, arXiv:2307.03172](https://arxiv.org/abs/2307.03172); Remediation: [learn — lost-in-the-middle](https://learn.agentpatterns.ai/context-engineering/lost-in-the-middle/)) |
| **Structural compression** | Move task-specific content to on-demand/skill files instead of lexically squeezing it | Content that doesn't compress but isn't needed every task ([instruction-compliance-ceiling](https://agentpatterns.ai/instructions/instruction-compliance-ceiling/); Remediation: [learn — the-ceiling](https://learn.agentpatterns.ai/prompt-engineering/the-ceiling/)) |

Structural compression is the highest-leverage move: aggregate rule load (across all
always-loaded files), not file count, drives compliance degradation — past the ceiling
even frontier models hold only ~68% accuracy ([IFScale 2025, arXiv:2507.11538](https://arxiv.org/abs/2507.11538)).
Lexically squeezing a rule that doesn't belong in the always-loaded layer is the wrong fix.

## Worked before / after

Same constraints, far fewer tokens (lexical: rules over explanations, bullets over sentences):

```markdown
<!-- Before — ~70 tokens -->
## Testing Requirements
It is very important that you make sure all code changes are thoroughly tested
before submitting them for review. You should always write unit tests that cover
the main logic of any function you add or modify. Try to ensure that edge cases are
handled appropriately. Please do not submit code that has not been tested.
```

```markdown
<!-- After — ~28 tokens, 60% smaller -->
## Testing
- Write unit tests for every function added or modified
- Cover edge cases
- Do not submit untested code
```

Every constraint survives (unit tests, edge cases, no-untested-code); only ceremony
("It is very important that…", "Try to…", "Please…") is gone. Note what is *not* cut:
a conditional like `integration tests when the function touches the database, unit
tests otherwise` is meaning, not ceremony — compressing it away applies the rule
uniformly where it should apply selectively.
