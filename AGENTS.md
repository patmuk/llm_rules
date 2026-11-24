# AGENTS.md — llm_rules

This repository holds the general rules for LLM coding agents that
several projects import: `LLM_RULES.md` (the entry), `LLM_GENERAL/`
(rules for every project), `stacks/` (rules per toolchain). A project
reads this checkout live, by relative path, so an edit here applies to
every project at its next session.

The rules below are for work IN this repository. The general rules
apply here too:

@LLM_RULES.md

## Changing a rule

- Every change bumps the version line in `LLM_RULES.md`, adds an entry
  to `CHANGELOG.md`, and names the new version in the commit message
  (`docs(rules): v1.2.0 — <summary>`), so a project can find the rules
  an old commit of its was written under.
- Semantic versioning:
  - **patch** — wording, no change of meaning;
  - **minor** — a rule added, changed or removed;
  - **major** — the contract with the projects changed: a file name, a
    path, a required heading of `LLM_GENERAL/LLM_PROJECT_TEMPLATE.md`.
    The changelog entry says what a project must do.
- A changelog entry says what changed and, where a project has to
  follow, what to do there. One entry per version.

## What goes where

- A rule that holds for every project → `LLM_GENERAL/`.
- A rule of one language or toolchain → its file in `stacks/`.
- A rule of one project → that project's `LLM/`, never here.
- **Never name a project here**: no project, product, app, crate or
  type name of one, no path into one. An example uses invented names.
  This repository is public.
- A lesson keeps its rule and one clause of why; the incident's story
  stays out.
- What is loaded into every session (`LLM_RULES.md`,
  `LLM_GENERAL/LLM_WAYS_OF_WORKING.md`) holds constraints and pointers
  only. A procedure is a skill of the plugin `pipeline`; detail is a
  file read on demand.
- Files in `LLM_GENERAL/` and `stacks/` import nothing (`@file`): a
  harness follows imports only a few hops deep. They point by path.
- No file here takes the name of a file a project keeps in its `LLM/`
  (`LLM_PROJECT.md`, `LLM_REPO_MAP.md`, `LLM_AREA_PATTERNS.md`,
  `LLM_REFERENCES.md`, `LLM_PROJECT_USER_TESTING.md`).

## This repository's own layout

- `plan/` and `./todo.md` as in `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §3.
- No build and no tests: a change is verified by reading it against
  the files that cite it (`grep` the heading or the path you changed).
