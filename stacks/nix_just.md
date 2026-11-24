# stacks/nix_just.md — the nix dev shell and `just`

For projects whose environment is a nix flake (with direnv) and whose
commands are `just` recipes. Read before running a dev shell or a
recipe from a jj workspace.

## `just`

- The recipes are the project's commands; the verify table in
  `LLM/LLM_PROJECT.md` §Verify names the ones that prove a change.
- A recipe that formats or lints the whole repo is not for a task's
  touched files (`LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §4).

## `nix develop` — only from the default repo directory

Never from inside a jj workspace: a flake in a directory without `.git`
is a plain path input, and nix copies the WHOLE directory — every build
output in it — into `/nix/store` on every call (gigabytes a time; it
has filled a disk). From the git-backed default directory nix copies
tracked files only. Pattern, with the `cd` inside the command:

```sh
nix develop --command bash -c 'cd ../<repo>-<task-slug> && just <recipe>'
```

A stray copy shows up as `/nix/store/*-source` holding build output;
`nix store delete` removes it.

Exception: when the fix under test is in the workspace's `flake.nix`
and not yet in the default directory's, `nix develop "path:$PWD"` from
the workspace is allowed — while the workspace holds no build output: a
fresh workspace, the build directory set outside it, no build run in it
before. Nix then copies a few MB per call.

## direnv — never in a workspace

It is the same call: the tracked `.envrc` (`use flake .`) is in every
workspace, so `direnv allow`, `direnv exec .` or a direnv-hooked shell
inside a workspace evaluates the flake there, copies the directory and
PINS the copy as a gc root (`<ws>/.direnv/flake-inputs/`), so no
garbage collection removes it.

- Never allow or exec direnv in a workspace.
- A script that takes a repo as a parameter runs its dev shell from the
  default directory (the pattern above) when that repo has no `.git`.
- Repair: `rm -rf <ws>/.direnv`, then `nix store delete` the copy.
