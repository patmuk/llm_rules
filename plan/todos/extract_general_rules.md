# Extract the general LLM rules into this repo

Status: decided 2026-11-24 (grilling session), nothing built yet.

## Problem

Three app repos each carry a copy
of the same agent rules: `LLM_RULES.md` plus an `LLM/` folder. About
1,800 lines of process and style text exist three times and drift:
a lesson written into one repo does not reach the others. The always
loaded part is about 1,250 lines per session, and the technical
specifications the process asks for are too long to be read before
approval.

## Solution

- This repo, `llm_rules`, holds the general rules once. Every project
  reads them live from the sibling checkout and keeps only what is its
  own.
- Procedures (the phases of the task protocol) become skills in the
  owner's plugin repository; constraints and facts stay in rule files.
- The process takes over what Matt Pocock's engineering skills do
  better; his thinking skills are used unmodified, his pipeline skills
  are not used.

## Out of scope

- Versioning of the shared crate the apps build on: its crate versions
  solve it, as a separate todo.
- Forking Matt Pocock's skills.
- A second harness. The layout is harness-neutral (`AGENTS.md`), but
  only Claude Code is wired.
- Parallel implementer agents.

## Decisions

### D1 — Layout

```
llm_rules/
  AGENTS.md        for an agent working IN this repo: bump the version,
                   add a changelog line; imports LLM_RULES.md
  LLM_RULES.md     the entry the projects import; carries the version line
  CHANGELOG.md
  LLM_GENERAL/     LLM_WAYS_OF_WORKING.md, LLM_CODING_STYLE.md,
                   LLM_DESIGN_PRINCIPLES.md, LLM_JJ_PLAYBOOKS.md,
                   UI_DESIGN_PRINCIPLES.md, LLM_USER_TESTING.md,
                   the adapter for third-party skills (D9)
  stacks/          rust.md, flutter.md, swift.md, nix_just.md
  plan/

<project>/
  AGENTS.md        the pointer block (D2), then the project's imports
  LLM/             LLM_PROJECT.md (always loaded: verify table, commit
                   scopes, workspace gotchas, which stacks apply),
                   LLM_REPO_MAP.md, LLM_AREA_PATTERNS.md,
                   LLM_PROJECT_USER_TESTING.md (on demand)
```

- No file name exists in both repos, and the folder names differ
  (`LLM_GENERAL/` here, `LLM/` in a project), so a path or a bare file
  name is never ambiguous. No symlink.
- A project has no `CLAUDE.md`: Claude Code reads `AGENTS.md` natively
  when no `CLAUDE.md` exists (needs v2.1.277 or later). `LLM_RULES.md`
  in a project is renamed to `AGENTS.md`.
- Three layers: general rules, stack modules a project selects, the
  project overlay.
- General files import nothing further (Claude Code follows imports
  four hops deep; the chain `AGENTS.md` → `LLM_RULES.md` →
  `LLM_GENERAL/…` uses two).
- The general text never names a project. It refers to fixed slots of
  the overlay ("the project's verify table, `LLM/LLM_PROJECT.md`
  §Verify"). This repo carries the overlay template with the required
  headings.

### D2 — Reference and version

A project's `AGENTS.md` starts with (the relative depth differs per
project):

```
The general rules are defined in ../../llm_rules/LLM_RULES.md (imported
below); they are mandatory and override default behavior. Every path
they name under `LLM_GENERAL/` or `stacks/` is relative to that
directory; `LLM/` is this project's.

The latest version read was v1.0.0. If the imported file carries a later
version, read CHANGELOG.md for the entries in between, adapt this
project where an entry asks for it, and update the number here in a
commit of its own. If it carries an older one, report to the user and
stop.

If the directory is missing, tell the user and read the online backup
(the GitHub repository of the same name), which may be older than the
local version.
```

followed by the import line outside any code block.

- Semantic versioning: patch = wording; minor = a rule added or
  changed; major = the contract with the projects changed (a slot, a
  required heading, a file name).
- Every change here bumps the version, adds a `CHANGELOG.md` entry and
  names the version in the commit message, so
  `jj log -r 'description(substring:"v1.20.3")'` finds the rules an old
  project commit was written under. No lock file.
- The projects read the local checkout live; nothing is pushed first.
- The first version a project is wired to is v1.0.0.

### D3 — Base text

The three projects are named here by their role: the **oldest** (the
rules were first written there), the **base** (cloned from the oldest
and evolved furthest since) and the **newest** (a fresh, uncurated copy
of the base). The base project's text is the base; the oldest
project's newer rules are merged in; the newest project's copy is no
source. Where two wordings conflict, the base project's wins. Rules found in
one repo only, to carry over:

- from the base project: finish with `jj commit -m`; copy edited files before
  `update-stale`; tests are the reference and no scenario is dropped;
  the string-offset rules; "where the code follows a recorded decision,
  the doc is outdated"; a doc is finished when only the device test is
  missing;
- from the oldest project: clippy and the analyzer stay clean, an `#[allow]` carries
  a `reason`; status-surface colour pairs; the monitor checks that both
  capture files grow; scripts run through bash, not zsh.
- Clippy runs with `--all-targets` (decided 2026-11-24): tests,
  examples, benches and build scripts are linted too, not only the
  library and the binaries.
- Stack-driven differences go to the overlay or a stack module, e.g.
  "almost everything `pub`" (FRB mirrors) against "private fields and
  accessors".

### D4 — What is always loaded

Hard rules, structure and pointers only. Detail moves to files read on
demand: most of a repo map's event flow, dated incident narratives, the
procedures (now skills). `retro` (Matt Pocock's skill) is run
periodically to prune rules that change nothing.

### D5 — Process changes in `LLM_WAYS_OF_WORKING.md`

The file keeps constraints only: when the gates apply and which skill
each one starts, the jj basics, what may be rewritten, the workspace
rule, the plan and todo layout with its markers, the commit rules, the
skill-precedence rule. Changes against today's text:

- **Gate 1**: the gap report halts only when it holds a `!decide` or a
  conflict between docs and code; those are asked as one grilling round,
  each question with a recommendation. Otherwise the report is printed
  and the work continues into the spec.
- **Gate 2**: no approval of the full specification. Decisions are
  grilled in rounds, the full spec goes into the plan doc, and the
  agent halts on a one-screen brief: problem, solution, out of scope,
  test seams, the slice list, and anything it decided against its own
  recommendation.
- **Spec sections**: problem, solution, user stories (short: one per
  behaviour, each matched by a test scenario), implementation
  decisions, testing decisions with the agreed seams, out of scope. The
  gap report (verified against the code) and the self-verification
  stay.
- **Build order**: vertical slices through all layers, each verified
  before the next.
- **Commits**: they follow the slices, grouped by feature. The
  commit-sequence table and "split by theme, not by buildability" are
  dropped. "One topic per commit" and "a stack shows no undone work"
  stay; red tests in an intermediate commit stay allowed.
- **TDD**: one test, one implementation; tests at agreed seams through
  public interfaces; the anti-patterns named (coupled to the
  implementation, tautological, all tests first).
- **Close-out**: before the report a fresh subagent reviews the stack
  on two separate axes — against the plan doc, against the rule docs —
  read-only through jj; its findings are fixed first.
- **Skill precedence**: a skill's instructions never override the
  rules: jj not git, the gates, the plan-doc layout, the commit rules.
  Where a skill says commit, branch, pull request or issue, the agent
  does the repo's equivalent or stops.
- **Markers** in todo docs and `todo.md`, beside `!needs` and
  `!decide`: `!triage` (an agent created it, the user has not looked),
  `!human <why>` (needs the user's hands), `DROPPED <date>` (will not be
  done; the doc moves to `plan/done/`).

### D6 — The pipeline as skills

A plugin `pipeline` in the owner's plugin repository, beside
`code-review-local`, as plain `SKILL.md` folders. Each skill can be
typed by the user and started by the agent when the ways of working
name it.

| Matt Pocock's (installed, not used) | Ours             | What ours does                                                        |
|--------------------------------------|------------------|-----------------------------------------------------------------------|
| —                                    | `gap-report`     | analysis verified against the code; halts only on `!decide` / conflict |
| `to-spec`                            | `spec`           | grilled decisions, the sections of D5, the one-screen brief            |
| `to-tickets`                         | (inside `spec`)  | the slice list in the plan doc, no ticket files                        |
| `implement`, `implement-spec`        | `implement-it`   | vertical slices, jj workspace, verify per slice, calls `tdd`           |
| `code-review`                        | `wrap-up`        | two-axis review through jj, plan doc and `todo.md` updated, the report |
| `wayfinder`                          | `decide`         | a map of decision questions in the todo doc, one per session           |
| `triage`                             | —                | the markers of D5 only                                                 |
| `pr`                                 | —                | no pull-request flow                                                   |
| —                                    | `device-round`   | a device test round with a monitor subagent                            |

- The table goes into the plugin's README.
- `implement-it` is the phrase the owner already says; `wrap-up`
  because `review` is taken by `code-review-local:review`. Typing
  `/impl…` also offers Matt Pocock's `implement`: pick the `pipeline:`
  one.
- The skills call Matt Pocock's thinking skills by name: `grilling`,
  `domain-modeling`, `tdd`, `diagnosing-bugs`, `prototype`, `research`,
  `codebase-design`. His plugin stays unmodified so it updates freely.
- The jj playbooks stay a reference file, read on demand.

### D7 — Facts verified in the Claude Code documentation (2026-11-24)

- An import resolves relative to the file that contains it; an import
  that leaves the project asks for approval once per project.
- Reading the general files on demand needs standing access to this
  directory: `permissions.additionalDirectories` in the user settings,
  with an absolute path. The exact JSON shape is to be checked when
  wiring the first project.
- A plugin from a marketplace added by local path loads in place: an
  edit is live at the next session or after `/reload-plugins`.

### D8 — Migration order

The newest project first (least to lose), then the base project, then
the oldest. A project is never named in this repo; a new project wires
itself (D2) and needs no entry here.

### D9 — Adapter for third-party skills

One short general file tells Matt Pocock's skills where the tracker is
(`todo.md` and `plan/todos/`, per the ways of working), which marker
stands for which triage role, and where the domain docs are (ADRs in
`plan/design_decisions/`, a `GLOSSARY.md` at the project root, created
when first needed).

## Plan

- [ ] 1. General text: build `LLM_RULES.md`, `LLM_GENERAL/` and
      `stacks/` from the base project's files, merge D3, apply D4 and D5,
      write the overlay template, `AGENTS.md`, `CHANGELOG.md`; v1.0.0.
- [ ] 2. Plugin `pipeline`: the six skills of D6 and the README table;
      check how the plugin repository is installed (local path).
- [ ] 3. Wire the newest project: `AGENTS.md` with the block of D2, the
      overlay, remove `CLAUDE.md` and the copied general files; approve
      the external import, add the directory to the user settings; run
      one small task through the pipeline.
- [ ] 4. Wire the base project the same way; a `retro` on step 3 first.
- [ ] 5. Wire the oldest project the same way.
- [ ] 6. Remove the `LLM_RULES.md` line from the user's global
      instructions.

Each step is a session of its own and starts with a gap report against
the repo it touches.

## Open

Nothing.
