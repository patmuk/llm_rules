# Changelog

One entry per version of `LLM_RULES.md`. The digits: patch = wording;
minor = a rule added, changed or removed; major = the contract with the
projects changed, and the entry says what a project must do.

## v1.1.0 — 2026-11-24

- **Device rounds**: the roles name model families; each takes the
  latest version of its family, never a pinned one. Nothing for a
  project to do.

## v1.0.0 — 2026-11-24

The first version, extracted from the rule docs three projects carried
as copies. Against those copies:

- **Layout**: the general rules live here (`LLM_RULES.md`,
  `LLM_GENERAL/`, `stacks/`); a project keeps `AGENTS.md` and its
  `LLM/` files as `LLM_GENERAL/LLM_PROJECT_TEMPLATE.md` lays out.
  A project to wire: follow that template; remove its copies of the
  general files, its `LLM_RULES.md` and its `CLAUDE.md`.
- **Gate 1**: the gap report halts only on a `!decide` or a conflict
  between docs and code.
- **Gate 2**: the user approves a one-screen brief, not the spec.
- **Spec**: problem, solution, short user stories, implementation
  decisions, testing decisions with agreed seams, slices, out of scope.
- **Build order**: vertical slices, each verified before the next.
- **Commits** follow the slices; the commit-sequence table and "split
  by theme, not by buildability" are gone.
- **Wrap-up**: a two-axis review by fresh subagents before the report.
- **TDD**: one test, one implementation; agreed seams; named
  anti-patterns.
- **Markers**: `!triage`, `!human <why>`, `DROPPED <date>`.
- **Skills**: the phases are the plugin `pipeline`'s skills; a skill
  never overrides the rules.
- **Verify**: linters stay clean whoever wrote the code; clippy runs
  with `--all-targets`.
