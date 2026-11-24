# LLM_PROJECT_TEMPLATE.md — what a project provides

Read when wiring a project to the general rules, or when a changelog
entry changes this contract (a major version). The general rules never
name a project; they name the files and headings below. A project
provides all of them — a heading with nothing to say holds "none".

## Files

| File                                  | Loaded      | Holds                                                         |
|---------------------------------------|-------------|---------------------------------------------------------------|
| `AGENTS.md`                           | always      | the pointer block below, then the imports of the next two     |
| `LLM/LLM_PROJECT.md`                  | always      | the headings below                                            |
| `LLM/LLM_REPO_MAP.md`                 | always      | hard rules, structure, dependency rules, a dispatch table by topic; detail goes to the area patterns |
| `LLM/LLM_AREA_PATTERNS.md`            | on demand   | per-area patterns and gotchas, one section per area           |
| `LLM/LLM_REFERENCES.md`               | on demand   | external docs for the frameworks in use                       |
| `LLM/LLM_PROJECT_USER_TESTING.md`     | on demand   | devices, capture commands and platform steps of a test round  |
| `./todo.md`, `plan/`                  | —           | as `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §3 lays out           |

No file of a project carries the name of a file in `LLM_GENERAL/` or
`stacks/`. A project has no `CLAUDE.md`: Claude Code reads `AGENTS.md`
when none exists.

## `AGENTS.md`

````markdown
# AGENTS.md — <project>

The general rules are defined in <relative path>/llm_rules/LLM_RULES.md
(imported below); they are mandatory and override default behavior.
Every path they name under `LLM_GENERAL/` or `stacks/` is relative to
that directory; `LLM/` is this project's.

The latest version read was v1.0.0. If the imported file carries a later
version, read the CHANGELOG.md beside it for the entries in between,
adapt this project where an entry asks for it, and update the number
here in a commit of its own. If it carries an older one, report to the
user and stop.

If the directory is missing, tell the user and read the online backup
(the repository `llm_rules` of the project owner's GitHub account),
which may be older than the local version.

@<relative path>/llm_rules/LLM_RULES.md

## This project

<naming, one paragraph>

@LLM/LLM_PROJECT.md

@LLM/LLM_REPO_MAP.md
````

Once per machine and project: approve the external import when the
harness asks, and give the harness standing read access to the
`llm_rules` directory (Claude Code: `permissions.additionalDirectories`
in the user settings, an absolute path).

## `LLM/LLM_PROJECT.md` — required headings

```markdown
# LLM_PROJECT.md — <project>

## Stacks
Which files of `stacks/` apply: e.g. `rust`, `swift`, `apple`, `nix_just`.

## Gate triggers
What, when touched, makes the gated protocol apply (ways of working
§1.1): domain types, a persistence shape, a file or wire format, a
bridge surface, a generator, …

## Edge cases
The kinds of hidden edge case a gap report looks for here: platform
constraints, data-safety and consent rules, limits of a generator,
async invariants.

## Spec checks
What the self-verification checks beyond the general list (e.g. every
persisted type fits a supported storage shape).

## Commit scopes
The feature names of the first scope segment, and example messages.

## Generated files
Files never edited and never hand-merged, each with the command that
regenerates it.

## Fresh workspace
What to run or copy in a new jj workspace before the first build or
format.

## Verify
The table: touched area → command; last row: the final commit. Then the
format commands for touched files, and the commands never to run.

## UI theme
The theme file and how a new token is added.

## Gotchas
Anything else that is this project's alone and needed in every session.
Per-area gotchas go to `LLM/LLM_AREA_PATTERNS.md`.
```
