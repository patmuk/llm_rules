# LLM_WAYS_OF_WORKING.md — process rules

How LLM agents organise, version, and verify work: when the gated task
protocol applies, the jj / workspace rules, where plans live, what
proves a change, and what a skill may not override. This file holds the
constraints; the procedures are the skills of the plugin `pipeline`
(§1.2). It is imported by `LLM_RULES.md` into every session, so its
rules are active before the first tool call.

What is the project's own is named by its slot in the project's
`LLM/LLM_PROJECT.md` (the headings: `LLM_GENERAL/LLM_PROJECT_TEMPLATE.md`).

---

## 1. Task execution protocol (gated)

### 1.1 When the gates apply

Run the phases of §1.2 when ANY of these holds:

- the work will need more than one commit;
- it touches what the project lists under §Gate triggers (domain types,
  a persistence shape, a file or wire format, a bridge surface, a
  generator, …);
- it spans layers;
- the user calls it a feature, a refactor, or a non-trivial bug fix.

Skip Phases 1–2 and start at Phase 3 when:

- the user supplies the spec (a plan doc, a decided todo entry, or an
  explicit step list);
- the change is a single-commit fix, a doc / plan edit, formatting, or
  a test-only change;
- the user says "skip gates" (or equivalent) in the current message.

A phase gate is a stop: **end the turn** and wait. Never continue past
a gate in the same reply.

### 1.2 The phases and their skills

Each phase is a skill of the plugin `pipeline`; call it when its phase
starts. Without the plugin, follow the summary here.

| Phase                | Skill                       | Gate                                                              |
|----------------------|-----------------------------|-------------------------------------------------------------------|
| 1 — Gap report       | `pipeline:gap-report`       | halts only on a `!decide` or a conflict between docs and code     |
| 2 — Specification    | `pipeline:spec`             | halts on the one-screen brief                                     |
| 3 — Implementation   | `pipeline:go-implement-it`  | none; a fix that needs a design change returns to Phase 2         |
| 4 — Wrap-up          | `pipeline:wrap-up`          | none; ends with the report                                        |
| a large `!decide`    | `pipeline:decide`           | one decision question per session                                 |
| a device test round  | `pipeline:device-round`     | `LLM_GENERAL/LLM_USER_TESTING.md`                                 |

**Phase 1 — Gap report.** Locate the work item (`./todo.md` → the
linked doc, or the `plan/…` doc the user names; no doc yet → create
one, §3). Read the code area and the matching area patterns; verify
every claim against the code, never against the docs alone. Write into
the plan doc under `## Current state`, short: what the code does today
against what it must do; the hidden edge cases (the project's §Edge
cases names the kinds to look for); conflicts between docs and code;
open product decisions as `!decide` lines. Then:

- a `!decide` or a conflict → ask them as ONE round of numbered
  questions, each with a recommended answer worded so that "yes"
  accepts it, and halt;
- neither → print the report and continue into Phase 2.

**Phase 2 — Specification.** Open decisions are settled with the user
in rounds of questions first (as above); the spec only records what was
decided. Write it into the same plan doc under `## Plan`, with these
sections:

- **Problem** and **Solution**, from the user's perspective;
- **User stories** — short: one per behaviour, each matched by a test
  scenario;
- **Implementation decisions** — changes per layer, types first (ADTs
  and validating constructors before functions);
- **Testing decisions** — the seams the tests sit at, agreed with the
  user; which scenarios (a scenario goes as a comment above its test);
- **Slices** — the vertical slices to build, in order, each an action
  item `[ ]`;
- **Out of scope**.

Then the Self-Verification Summary — ALWAYS. Check the spec against
each item and print one line per item, as `checked <rule>: ok` or
`checked <rule>: adjusted — <what>`: the repo map's hard rules,
dependency rules and async rules; what the project lists under §Spec
checks; `LLM_GENERAL/LLM_DESIGN_PRINCIPLES.md` §1–3;
`LLM_GENERAL/LLM_CODING_STYLE.md` and the stack files for any code
sketched; `LLM_GENERAL/UI_DESIGN_PRINCIPLES.md` for any new UI size or
spacing.

Halt on a **one-screen brief**, not on the spec: problem, solution, out
of scope, the test seams, the slice list, and anything decided against
your own recommendation. Ask exactly: "Do you approve this brief?"

Plan mode of a harness may carry Phases 1–2; leaving it with the user's
approval counts as the Phase 2 gate. The plan doc, not the plan-mode
file, is the persisted spec.

**Phase 3 — Implementation.** In a task workspace (§2.3) unless working
in the user's checked-out commit (§2.4). Build the slices in order;
each slice cuts through every layer it needs and is verified (§4)
before the next starts. Within a slice: one test, then the code that
makes it pass, and again (`LLM_GENERAL/LLM_DESIGN_PRINCIPLES.md` §3).
After each slice:

- a red test the spec predicted → note it, continue;
- any other red test or compile error → fix it now. No permission
  needed for compile or test fixes; state the error and the fix;
- a fix that needs a design change → stop, report, return to Phase 2
  for that part.

**Phase 4 — Wrap-up.** Before the report:

1. two fresh subagents review the stack, read-only through jj, on two
   separate axes that are never merged: against the plan doc (missing,
   wrong, or unasked-for behaviour) and against the rule docs;
2. findings are fixed, each into the commit that owns it (§2.2);
3. everything compiles, all touched suites green, touched files
   formatted, `jj log -r "conflicts()"` empty;
4. the plan doc is updated (status, or moved to `plan/done/`) with its
   line in `./todo.md`; the workspace is cleaned up (§2.3);
5. report: commits (change ids + what each delivers), verify commands
   run with their results, and every deviation from the spec.

---

## 2. Version control — jj

### 2.1 Basics

- **jj, never git** — read-only included: `jj log` / `jj diff -r` /
  `jj file show` / `jj file annotate`, never `git log|show|grep|blame`.
  Plain `mv` / `rm` for files — jj snapshots them. Never
  `git mv|rm|commit`.
- Non-interactive only: always `-m "…"`; never `-i` /
  `--interactive`. `jj squash` between two described commits needs
  `--use-destination-message`, else it opens an editor and hangs.
- Prefer change ids (letters) over commit ids (hex).
- Finish a commit with `jj commit -m "…"` (describe + new empty
  change). `jj describe` alone leaves `@` on the finished commit, so
  the next edits silently amend it.
- Generated files (the project's §Generated files) are never
  hand-merged in a conflict — regenerate at the commit whose job is
  regeneration, then `jj st`.
- Formatting-only accidental edits (a broad formatter run) are undone
  with `jj restore --from <base> -- 'root:<path>'` in the owning
  workspace. If a file changes because of running a formatter, commit
  this in its own commit.
- Don't edit the tree under a running review workflow — a review reads
  a snapshot; mutating mid-run makes its agents disagree on the target.
- Content grep across commits: loop `jj diff -r $rev --git | grep`.
  Find an introducer with `jj file annotate <path>`.
- After every rebase, squash, or amend of a change with descendants:
  `jj log -r "conflicts()"` must be empty. Resolve conflicts bottom-up,
  each hunk into the commit that owns it.
- One VCS operator at a time per jj workspace: no subagent or workflow
  runs jj while the user may be rewriting the same stack. Two agents
  working on the same files create a workspace each.

### 2.2 What you may rewrite

**Own-session commits** — commits you created in this session — may be
rewritten freely and without asking: `describe`, `squash`, `split`,
`rebase`, `abandon`, `restore`, `absorb` (when every target is your
own).

Everything else is read-only; suggest the command and let the user run
it:

- commits the user made (in any session);
- anything at or below the `reviewed` bookmark (`::reviewed`);
- immutable commits (`main`, pushed history).

Further limits:

- `jj undo` / `jj op restore`: only when `jj op log` shows the operation
  to revert is yours and is the latest — the user may be working
  concurrently in the default workspace.
- `jj git push` (any form): always confirm first.
- A fix for something an own-session commit introduced goes INTO that
  commit. A fix for anything else is a new commit on top of the stack
  that cites the introducer's change id. Keep the stack linear.
- Commit messages: `type(feature/scope): summary`. The type stays
  truthful (`feat` / `fix` / `refactor` / `chore` / `test` / `docs` /
  `style` / `perf`); the FIRST scope segment names the feature the
  commit serves (the project's §Commit scopes lists them), so a
  reviewer reads one feature at a time in a linear stack
  (`jj log -r 'reviewed..main & description(substring:"(<feature>")'`
  — the `substring:` prefix is needed because the pattern starts with a
  paren). The second segment is the area inside it; drop it when it
  would repeat the feature. `docs(llm)` / `docs(plan)` stay as they
  are. Work in progress: `wip(<task-slug>): …`.

### 2.3 Workspaces — the default one belongs to the user

The `default` workspace (the repo directory) is the user's review
space; they move its `@` with `jj edit` at any time. Agent work happens
in a per-task workspace unless §2.4 applies.

- First: `jj workspace list`. Reuse a workspace only for the same task.
- Create, from inside the repo directory (the path resolves against
  the shell cwd, not against `-R`):
  `jj workspace add ../<repo>-<task-slug> --name <task-slug> -r <base>`
  — a sibling of the repo, so every relative path out of the repo holds
  in it too; `<base>` is the change to build on (usually the user's `@`
  or the stack tip). This creates a fresh child change.
- Immediately describe it:
  `jj -R ../<repo>-<task-slug> describe -m "wip(<task-slug>): …"`.
  Never point a second working copy at a change yourself (`jj edit`);
  when the user does so to review, snapshot with `jj st` in your
  workspace and run `jj workspace update-stale` on whichever side
  reports stale.
- All reads / edits / builds / test runs use the workspace path.
- Every jj command snapshots the working copy of the workspace it runs
  in before doing its job (`--ignore-working-copy` is the only
  opt-out); `jj st` is the cheapest one. Run it in the workspace after
  every edit round — editors and build tools never snapshot, so without
  it the user's view of the same change stays stale.
- "Working copy is stale" → `jj workspace update-stale` (safe; run it
  yourself) — but FIRST copy any file edited since the last snapshot
  somewhere outside the workspace: the update resets the files to the
  commit, and edits the stale workspace never snapshotted are gone.
- A fresh workspace holds no build output and no ignored file: the
  project's §Fresh workspace says what to run or copy first. The
  environment and build-cache rules of a workspace are in
  `stacks/nix_just.md`, `stacks/rust.md` and `stacks/apple.md`.
- `<ws>/.jj/repo` holds a RELATIVE path to the main repo; after
  `mv`-ing a workspace directory rewrite it, then `jj st` works again.
- Names: subagents get task-derived descriptions; a worktree-isolated
  agent's `agent-<hex>` workspace is renamed to a task slug
  (`jj workspace rename`) and its `@` described before it reports.
- **Cleanup when the work is done** (wrap-up reported): leave the
  finished commits in place, then — from the default repo directory,
  never from inside the workspace being removed —
  `jj workspace forget <task-slug>` and `rm -rf ../<repo>-<task-slug>`,
  then the build caches that live outside it (`stacks/apple.md`).
  Confirm with `jj log` that the commits survived. Follow-ups on the
  same task get a new workspace.

### 2.4 Working in the user's checked-out commit

When the user asks for a change "in the checked-out commit", "here", or
"in `@`", work directly in the default workspace on their `@`:

- no workspace, no `jj new`, no `jj edit` — the edits amend their `@`;
- this mode is meant for quick fixes. If the request turns out large
  (more than one commit, or any §1.1 trigger), ask for confirmation
  before editing.

### 2.5 Commit rules

- Commits follow the slices and are grouped by feature: what a slice
  delivers is one commit, or a few where it holds separate topics.
- **A commit contains only changes that need each other.** One topic
  per commit: a feature and an import reorder it does not need are two
  commits; a rename that the feature does not depend on is its own
  commit, before it.
- **A stack shows no undone work.** Code that a later commit in the
  same stack changes again or removes never appears in the earlier
  one: fold the fix into the commit that introduced the thing, or drop
  the intermediate step.
- Intermediate commits must compile; their tests MAY be red where the
  spec said so.
- The final commit compiles with all touched suites green.
- Rebase / split / squash own-session commits freely to reach that
  shape (§2.2). The user squashes reviewed stacks into feature commits.

---

## 3. Plans, todos, design decisions

- `./todo.md` is the meta index: one line per open todo, grouped
  blocked / to be tested / startable. The linked doc is the source of
  truth — update BOTH when a todo lands, and when you create one. Only
  the major items go into the index.
- Todo docs live in `plan/todos/{todos_for_mvp, todos_for_prod,
  todos_tech_debt, future_extension_ideas}/`. Each doc leads with its
  open items and its markers:
  - `!needs <X>` — waits on X; its line stands under "blocked";
  - `!decide <Q>` — waits on a product or protocol decision; the agenda
    goes inline;
  - `!triage` — an agent created the todo, the user has not looked at
    it yet;
  - `!human <why>` — needs the user's own hands (a device, a store, an
    account);
  - `DECIDED <date>` — the record of a decision;
  - `DROPPED <date>` — will not be done; the doc moves to `plan/done/`.
- Large `!decide` items: discuss to a decision first, no code
  (`pipeline:decide`). Record it in the doc (`DECIDED <date>`) before
  Phase 2.
- Finished docs move to `plan/done/`. A doc is finished when only the
  device test is missing: built, no `!decide` and no unbuilt item left;
  it states what the device round checks, and `./todo.md` keeps its
  line under "to be tested".
- Design rationale lives in `plan/design_decisions/`, one file per
  topic.
- The gap report and the spec live in the work item's doc
  (`## Current state`, `## Plan`) — not in chat, not in a harness's
  plan file.
- Test scenarios go as comments above the test, never as standalone
  files. The tests are the reference of the behaviour: no scenario is
  dropped or weakened to make a change pass; changing one is a
  `!decide`.
- A doc found outdated is updated in the task that finds it: a doc that
  claims to describe the present (a rule doc, a README, a code comment,
  an open todo) is corrected against the code, in a `docs(…)` commit
  apart from the code. A history doc (`plan/done/`, a dated log, a
  decision's record) is not rewritten: it gets a dated pointer line to
  what holds now.
- Durable knowledge goes into a repo, not into the agent's auto-memory.
  When a session learns something, edit the owning doc:
  - a process rule, an idiom, a design rule, a tool gotcha that holds
    for every project → the general rules (the repository of this file:
    its `AGENTS.md` says how to change it);
  - architecture → `LLM/LLM_REPO_MAP.md`; per-area patterns and gotchas
    → `LLM/LLM_AREA_PATTERNS.md`; project commands, scopes and
    workspace steps → `LLM/LLM_PROJECT.md`;
  - decisions and product invariants → `plan/design_decisions/`; status
    → the todo doc + `todo.md`; device and tooling how-tos → `README.md`.
  Auto-memory is only for what no repo must or can hold (personal
  tooling, cross-project preferences).

---

## 4. Verify — build, test, format

- The commands are the project's: `LLM/LLM_PROJECT.md` §Verify, a table
  of touched area → command. Run them for every touched area after each
  slice, from the workspace root unless the table says otherwise; the
  table's last row is what the final commit must pass.
- Format only what you touched, with the commands named there. Never a
  repo-wide formatter or lint-fix run: it reformats unrelated files.
- Linters and analyzers stay clean, whoever wrote the code: a finding
  is fixed the way the lint asks. A suppression carries its reason and
  stands only where the lint is wrong for the code, or on generated
  code — at the place that includes the generated code, never in the
  generated file.
- UI tests and previews that run against a mocked core skip the
  domain's validation. UI that depends on domain rules also needs an
  integration scenario against the real core — green UI tests alone do
  not prove it.
- A red test is believed only once its run is known to be sound (the
  app started, the fixture loaded); a test that fails as if the app
  were wrong may be failing on a stale install.

---

## 5. Skills

- A skill's instructions never override these rules: jj not git, the
  gates, the plan-doc layout, the commit rules. Where a skill says
  commit, branch, worktree, pull request or issue, do this file's
  equivalent (§2, §3) or stop and say so.
- The process is the plugin `pipeline` (§1.2). Its skills call these
  thinking skills where they fit, and so may you: `grilling` (decisions
  in rounds), `domain-modeling` (terms, the glossary, a decision's
  record), `tdd`, `diagnosing-bugs` (a red feedback loop before any
  hypothesis), `prototype`, `research`, `codebase-design`.
- Before another plugin's skill reads or writes "the issue tracker",
  "triage labels", "ADRs" or a glossary:
  `LLM_GENERAL/LLM_THIRD_PARTY_SKILLS.md`.
- `retro` is run from time to time on a finished task: its findings go
  to the owning doc (§3), and a rule that changes no behaviour is
  removed.
