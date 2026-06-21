# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A curated collection of portable, project-agnostic **agent skills**, built from the
**agentpatterns** researched corpus. There is no build, no test runner, no dependencies — the
entire repo is Markdown. Each top-level directory is one skill, defined by a single `SKILL.md`
with load-bearing frontmatter.

This repo is one of three complementary projects: the **agentpatterns** corpus is the
researched source material, **learn** and **website** are sibling projects, and **skills** (this
repo) is the curated, executable distillation of that corpus into agent skills. When a skill's
claims derive from a pattern, they should trace back to that corpus (e.g. the agentpatterns.ai
references already cited in the skill bodies).

## Skill contract

Every skill is a directory containing `SKILL.md`. The frontmatter is the load-bearing part —
it controls when and how the skill is surfaced:

- `name` — must match the directory name.
- `description` — the dispatch signal. Always written as **"<what it does>. Invoke when … Skip
  when …"**; the *invoke when / skip when* clauses are how an agent decides to load the skill,
  so keep them sharp and mutually exclusive across skills.
- `user-invocable: true`, `usage: /<name> [args]` — exposes it as a slash command.
- `version` — semver; bump on any contract change (changed check IDs, output shape, or stance).

The skill body is itself an instruction file, so it must obey the very principles these skills
audit for: front-loaded constraints, high semantic density, no discoverable bloat.

## The skills are a detector/transform pair — keep them in sync

`audit-instruction-file` **detects** problems in instruction files via a numbered check catalog
(`CE-1` … `CE-10`). `compress-prompt` **applies** the fix for one of those checks (`CE-10`, low
semantic density). They reference each other by name and by check ID. When you change one side
of that relationship, update the other:

- Renaming/renumbering a `CE-N` check → update every cross-reference in both files and the README.
- `audit-instruction-file` is **detect + recommend**, applying only mechanical, semantics-preserving
  fixes on explicit confirmation; `compress-prompt` is the **transform** side. Preserve this
  detector→transform split — don't make the auditor silently rewrite governance files.
- Each `CE-N` check is intentionally self-contained (ID / Flags / Why / Fix, with a cited source)
  so it can be read in isolation. Preserve that when adding checks (see *Expanding this skill* at
  the bottom of `audit-instruction-file/SKILL.md`).

## When editing skills

- The README lists each skill with a one-line summary — keep it in sync when adding/removing a
  skill or materially changing a description.
- New normative claims in a skill body should cite a primary source inline, matching the existing
  style (these skills are evidence-based by design).
