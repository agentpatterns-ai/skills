# skills

Portable, project-agnostic agent skills, grouped into three category folders by archetype —
`audit/`, `transform/`, `advisor/` (see Skills below). Each skill is a directory holding a
`SKILL.md` with load-bearing frontmatter (`name`, `description` with *invoke when / skip when*).
Most skills progressively disclose their detail (a `checks.md` detector catalog, a worked
template) so the always-loaded surface stays small. All skills ship at one uniform shelf
version. The category folder is organizational only — it is not part of a skill's `name` and
does not affect dispatch.

## Install / update

Copy the skill directories you want into your harness's skill path (e.g. `~/.claude/skills/`).
To **update**, sync against [`MANIFEST`](MANIFEST) — one skill package path per line, e.g.
`audit/audit-lethal-trifecta` — copy every listed dir and **delete any previously-installed
shelf skill dir that is no longer listed** (e.g. `rsync -a --delete` per dir, or prune by
diffing your install against MANIFEST). A plain re-copy without pruning strands stale copies
when a skill is **renamed or moved to a different category**: the old dir keeps dispatching
under its old name while every sibling's Skip-clause handoff points at the new one. Renames
are called out (`old-name` → `new-name`) in each release's notes. This repo's history is
incremental (one commit per publish, tagged `vX.Y.Z` per release) — `git clone` (or
`git clone --depth 1` for a shallow install) both give a correct, current install; pin
`--branch vX.Y.Z` (or `<skill>@vX.Y.Z` with `gh skill install`, below) for a reproducible one.


## "Audit my agent harness" — the run order

No single skill owns the generic ask; each audits one facet. For a full harness audit, run them in
this order (each hands findings to the next; stop early where a High finding demands a fix first):

1. **audit-secret-exposure** — literal credentials in agent-readable surfaces (a High here halts the
   rest until rotated).
2. **audit-lethal-trifecta** — private data × untrusted input × egress per execution path.
3. **audit-harness-safety** — blast radius: sandbox, deny floors, reversibility, kill path, budgets.
4. **audit-supply-chain-sinks** — LLM output crossing into shell/SQL/render/path/install sinks.
5. **audit-prompt-hook-placement** — which rules belong in hooks vs prose; hook soundness.
6. **audit-verification-gates** — can the agent's "done" be trusted.
7. **audit-observability-setup** — can you see the run (telemetry, breakers, trace identity).
8. **audit-prompt-cache-hygiene** — cost: prefix stability, TTL fit, monitoring.

Adjacent, run when the facet exists: **audit-mcp-server** / **audit-tool-definition** (tool surfaces),
**audit-memory-retrieval-integrity** (RAG/memory), **audit-multi-agent-orchestration** (>1 agent),
**audit-github-actions-security** (CI workflows), **audit-instruction-file** +
**conflicting-instruction-detector** (instruction prose), **audit-self-improvement-protocol**
(learning loops), **audit-geo-content** (published docs).


## Skills

### Harness security & blast-radius audits

- **[audit-lethal-trifecta](audit/audit-lethal-trifecta/SKILL.md)** — audit an agent / sub-agent /
  MCP setup for the lethal trifecta (private-data access + untrusted-content exposure +
  external egress on one execution path) and flag the prompt-injection and exfiltration
  routes it opens.
- **[audit-harness-safety](audit/audit-harness-safety/SKILL.md)** — audit a harness's setup for
  blast-radius containment and recovery: OS sandbox + least-privilege scope, hard deny
  floors, reversibility/idempotency, a kill path, and consumption bounds.
- **[audit-secret-exposure](audit/audit-secret-exposure/SKILL.md)** — scan an agent's context
  surfaces (instruction/skill files, harness config, tool defs, credential files) for
  secrets that landed where the model can read them, via a deterministic literal-secret scan.
- **[audit-supply-chain-sinks](audit/audit-supply-chain-sinks/SKILL.md)** — audit the output
  boundary where an LLM-emitted string crosses into a code-interpreting sink (shell, SQL,
  HTML render, file path, package manager, downstream LLM) plus slopsquatting install routes.
- **[audit-memory-retrieval-integrity](audit/audit-memory-retrieval-integrity/SKILL.md)** — audit a
  RAG / vector-store / agent-memory layer for the retrieval-boundary authz-and-integrity gap:
  cross-tenant chunk bleed, agent-supplied tenant identity, untokenized PII, and retrieved
  content trusted as fact-or-instruction with no provenance tier.
- **[audit-github-actions-security](audit/audit-github-actions-security/SKILL.md)** — audit GitHub
  Actions workflows against the OWASP Top 10 CI/CD security risks: script injection,
  pwn-requests, mutable-tag actions, over-broad `GITHUB_TOKEN`, long-lived secrets, and more.

### Agent design & coordination audits

- **[audit-tool-definition](audit/audit-tool-definition/SKILL.md)** — audit one or many individual
  tool/function definitions (name, description, schema, error contract, MCP annotations) for
  the ergonomics failures that make an agent mis-select or misuse a tool.
- **[audit-mcp-server](audit/audit-mcp-server/SKILL.md)** — audit an MCP server's own declaration:
  primitive choice (tool/resource/prompt), naming, schemas, behavior annotations, error
  channels, transport, server instructions, and tool-count/token budget.
- **[audit-multi-agent-orchestration](audit/audit-multi-agent-orchestration/SKILL.md)** — audit a
  running multi-agent harness's coordination mechanics: typed vs prose handoffs, shared
  mid-run state, verifier-independent-of-producer "done", and loop bounds.
- **[audit-verification-gates](audit/audit-verification-gates/SKILL.md)** — audit the
  completion-gating architecture (hooks, ledgers, graders, red-green constraints) and flag
  every gate that trusts self-report, grades the execution path, or is otherwise gameable.
- **[audit-observability-setup](audit/audit-observability-setup/SKILL.md)** — audit whether a long /
  autonomous / multi-agent run is legible enough to confirm it worked: OTel exporters, maxTurns
  + loop-detection circuit breakers, `agent_id` trace propagation, a wired regression gate.
- **[audit-prompt-cache-hygiene](audit/audit-prompt-cache-hygiene/SKILL.md)** — audit a harness's
  prompt assembly, tool serialization, and SDK/CLI cache config for cache-busting cost
  regressions: volatile content in the static prefix, mutated tool enumeration, mid-session
  model switches.
- **[audit-prompt-hook-placement](audit/audit-prompt-hook-placement/SKILL.md)** — audit the
  enforcement-medium decision: for each rule, does it belong in prose (judgment) or relocated
  to a deterministic gate (PreToolUse hook / CI / tool-restriction)?
- **[audit-self-improvement-protocol](audit/audit-self-improvement-protocol/SKILL.md)** — audit
  whether an agent's always-loaded base instruction file carries a well-formed self-improvement
  protocol (spot opportunities, gate changes behind approval, route learnings to a backlog,
  protect the core), then install or repair it.

### Instruction & content quality

- **[audit-instruction-file](audit/audit-instruction-file/SKILL.md)** — audit a single agent
  instruction file (`AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `copilot-instructions.md`,
  `GEMINI.md`, …) against context-engineering principles and recommend the smallest fix.
- **[conflicting-instruction-detector](audit/conflicting-instruction-detector/SKILL.md)** — surface
  semantic conflicts across two or more instruction files / a skill library — rules that
  contradict in effect though they share no wording — as candidate conflicts for human review,
  never auto-resolved.
- **[audit-geo-content](audit/audit-geo-content/SKILL.md)** — audit a docs/content corpus for AI
  answer-engine retrieval and citability (GEO): chunkability (answer-first, atomic, descriptive
  headings), assertion density, and machine-readable signals (`llms.txt`, schema).

### Authoring & transforms

- **[write-tool-description](transform/write-tool-description/SKILL.md)** — write the agent-facing
  description, parameter docs, and schema constraints for ONE tool so the model selects and
  calls it correctly: positive selection signal, sibling discriminator, return shape, example
  values, poka-yoke schema.
- **[compress-prompt](transform/compress-prompt/SKILL.md)** — compress any prompt or instruction file to
  maximize signal per token: prose → tables/bullets/rules, front-load critical constraints,
  semantics-preserving transforms. The fix-side complement to `audit-instruction-file`'s
  `CE-10` detector.

### Advisory

- **[architecture-committee](advisor/architecture-committee/SKILL.md)** — convene a debiased
  multi-persona committee to vet an architecture / design decision: distinct personas
  independently propose, steelman their pick AND its opposite, cross-examine across capped
  rounds, then converge on the approach that fits THIS spec rather than the model's defaults.

## Using a skill

Each skill is tool-agnostic Markdown. Install one into your agent with
[`gh skill`](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli)
(GitHub CLI v2.90+):

```sh
# Install one skill for Claude Code, user-wide (available in every project)
gh skill install agentpatterns-ai/skills audit-instruction-file --agent claude-code --scope user

# Or browse this repo's skills interactively
gh skill install agentpatterns-ai/skills --agent claude-code

# Pin to a release tag for reproducible CI / provisioning
gh skill install agentpatterns-ai/skills audit-instruction-file@v0.1.0 --agent claude-code --scope user
```

`--scope user` installs into your per-user agent config (available in every project);
`--scope project` (the default) writes the skill into the current checkout so teammates pick
it up on `git pull`. `--agent` also targets `cursor`, `codex`, `gemini`, and many other agents,
and `--pin <tag|sha>` (or `<skill>@<tag|sha>`) locks an install to an exact version for
reproducible CI/provisioning.

Once installed, invoke it as a slash command — e.g. `/audit-instruction-file [path-or-dir]`.
