# LLM_THIRD_PARTY_SKILLS.md — where other plugins' skills find things

Read before running a skill that is not the plugin `pipeline`'s and
that speaks of an issue tracker, triage labels, ADRs or a glossary
(Matt Pocock's engineering skills do). It maps their vocabulary onto
the layout of `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §3, which stays
authoritative; §5 there says what such a skill may not override.

## Issue tracker

There is none outside the repo: never call `gh issue …` or create
`.scratch/`.

| The skill says                 | Here                                                                                  |
|--------------------------------|---------------------------------------------------------------------------------------|
| an issue                       | a todo doc, `plan/todos/<category>/<slug>.md`, and its line in `./todo.md`              |
| the spec                       | the same doc's `## Current state` and `## Plan`                                        |
| tickets                        | the `[ ]` slices in `## Plan`; one too large gets its own todo doc, linked              |
| a comment                      | a dated line in the doc                                                               |
| closing an issue               | the doc moves to `plan/done/`, its line leaves `./todo.md`                             |
| a map with child tickets       | the effort's todo doc; a child is a `### NN — <question>` section with `Type:` and `Status:` lines |
| a branch, a worktree, a PR     | a jj workspace and a linear stack (§2 of the ways of working); no pull request         |

## Triage labels

| Role              | Marker                                              |
|-------------------|-----------------------------------------------------|
| `needs-triage`    | `!triage`                                           |
| `needs-info`      | `!decide <Q>`                                       |
| `ready-for-agent` | no marker; the line stands under "startable"        |
| `ready-for-human` | `!human <why>`                                      |
| `wontfix`         | `DROPPED <date>`; the doc moves to `plan/done/`     |

`!needs <X>` is not a triage role: the todo is blocked on X.

## Domain docs

- ADRs are the files of `plan/design_decisions/`: one per topic, named
  by the topic, not numbered. A decision on an existing topic is a
  dated entry (`DECIDED <date>`) in that file. There is no `docs/adr/`.
- The glossary is `GLOSSARY.md` at the project root, created when the
  first term is settled. Until then the terms are the code's and the
  repo map's; proceed without it silently.
- A skill's output that contradicts a recorded decision says so and
  names the file.

## Their pipeline skills

`to-spec`, `to-tickets`, `implement`, `implement-spec`, `triage`,
`wayfinder`, `pr` and their `code-review` are not this process: the
phases are `pipeline:gap-report`, `pipeline:spec`,
`pipeline:go-implement-it`, `pipeline:wrap-up` and `pipeline:decide`.
Run one of theirs only when the user types it.
