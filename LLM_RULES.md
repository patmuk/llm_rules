# LLM_RULES.md — general rules

Version: v1.0.0

These rules apply to **all LLM-assisted work** in every project that
imports this file. They are mandatory and override default LLM
behavior. Changes between versions: `CHANGELOG.md` beside this file.

Paths: `LLM_GENERAL/…` and `stacks/…` are relative to this file's
directory (the project's `AGENTS.md` says where that is). `LLM/…`,
`plan/…` and `./todo.md` are the project's.

---

## Mandatory always

Apply on every interaction, regardless of task.

### Source of truth
- The project's `LLM/LLM_REPO_MAP.md` is authoritative for architecture,
  layers, and dependencies. If a request would break a rule there,
  **refuse and explain why**.
- If information is missing or unclear, ASK before assuming.
- If docs and code disagree, surface the conflict; do not silently pick
  one. Where the code follows a recorded decision, the doc is outdated:
  update it in the same task (`LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §3).
- Do NOT infer architecture; do NOT hallucinate missing code.

### Context discipline
- Search the repository yourself (grep, read files) before assuming or
  asking; verify claims against the actual code.
- Ask only when genuine ambiguity remains after searching — e.g.
  conflicting requirements or an unstated product decision.

### Change discipline
- Make minimal, targeted edits; briefly explain intent when modifying code.
- Do not change existing code without functional changes. The diffs
  should not show unnecessarily changed files. Thus:
  - Do NOT reformat unrelated code. Use the tool formatting on touched
    files only (the commands: the project's `LLM/LLM_PROJECT.md`
    §Verify).
  - Do NOT reorder import statements. This is only ok if the formatter
    does this.
  - Do NOT rename symbols unless explicitly requested.
- Never drop an existing output field, column, or return value on a
  targeted edit; diff the output shape before and after.

### Quality bar
- Prefer correctness over cleverness, explicitness over abstraction.
- Avoid "framework-style" patterns unless already present.
- When uncertain, ask a precise question; do not guess.

### Failure mode
If a task cannot be completed safely with the available context:
- say so explicitly
- list exactly what is missing
- stop

Silence, guessing, or partial hallucinations are unacceptable.

### Ways of working — `LLM_GENERAL/LLM_WAYS_OF_WORKING.md`
Imported below in full: when the gated task protocol applies and which
skill runs each phase, the jj rules and per-task workspaces, the plan /
todo layout, the verify rules, and what a skill may not override.
Active before the first tool call.

@LLM_GENERAL/LLM_WAYS_OF_WORKING.md

---

## Mandatory when reasoning about a codebase

### Design principles — `LLM_GENERAL/LLM_DESIGN_PRINCIPLES.md`
Read in full when designing types, modelling new domain concepts, or
deciding how to structure tests. Covers making faulty state
unrepresentable (ADTs, validating constructors, transforming
functions), DDD, and TDD.

### UI design principles — `LLM_GENERAL/UI_DESIGN_PRINCIPLES.md`
Read in full for any UI work that picks or introduces a size, spacing,
radius, icon dimension, type role, or touch-target value. New theme
tokens MUST snap to the tables there; reuse an existing token before
adding a new one ("fewer sizes").

### The project's own docs
`LLM/LLM_REPO_MAP.md` and `LLM/LLM_PROJECT.md` are imported by the
project's `AGENTS.md`. Read the section of `LLM/LLM_AREA_PATTERNS.md`
that matches the area the task touches, per the dispatch table at the
top of the repo map.

---

## Mandatory when external APIs or frameworks are involved

Apply whenever the task touches an external framework, library, or API.

- Start with the project's `LLM/LLM_REFERENCES.md`; inside a
  subdirectory, also read its closest `LLM_REFERENCES.md`.

---

## Mandatory when writing code

Cumulative with the sections above.

### Cross-cutting idioms — `LLM_GENERAL/LLM_CODING_STYLE.md`
Read in full when writing code: functional idioms, the comment policy.

### Stack rules — `stacks/<stack>.md`
Read the file of every stack the task touches. The project's
`LLM/LLM_PROJECT.md` §Stacks lists which apply (`rust`, `swift`,
`flutter`, `apple`, `nix_just`).

### Version control — `LLM_GENERAL/LLM_JJ_PLAYBOOKS.md`
The basics are in the ways of working (imported above). Read the
playbooks on demand for the longer procedures: distributing conflict
fixes, folding commits, re-slicing a stack.

---

## On demand

- `LLM_GENERAL/LLM_USER_TESTING.md` — a device test round with the user.
- `LLM_GENERAL/LLM_THIRD_PARTY_SKILLS.md` — before running a skill that
  is not the plugin `pipeline`'s: where it finds the tracker, the
  triage markers and the domain docs here.
- `LLM_GENERAL/LLM_PROJECT_TEMPLATE.md` — when wiring a project, or when
  a changelog entry changes what a project must provide.
