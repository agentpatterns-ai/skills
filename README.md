# skills

Portable, project-agnostic agent skills. Each subdirectory is one skill: a `SKILL.md`
with load-bearing frontmatter (`name`, `description` with *invoke when / skip when*).

## Skills

- **[audit-instruction-file](audit-instruction-file/SKILL.md)** — audit any agent
  instruction file (`AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `copilot-instructions.md`, …)
  against context-engineering principles and recommend the smallest fix. An expandable
  check catalog that grows as new principles are learned.
- **[compress-prompt](compress-prompt/SKILL.md)** — compress any prompt or instruction
  file to maximize signal per token: prose → tables/bullets/rules, front-load critical
  constraints, structural compression. Semantics-preserving by default; the fix-side
  complement to `audit-instruction-file`'s `CE-10` detector.

## Using a skill

Skills are tool-agnostic Markdown. To make one available to Claude Code, expose it on the
skills path — e.g. symlink or copy the skill directory into a project's `.claude/skills/`,
or into `~/.claude/skills/` for all projects:

```sh
ln -s "$(pwd)/audit-instruction-file" ~/.claude/skills/audit-instruction-file
```

Then invoke with `/audit-instruction-file [path-or-dir]`.
