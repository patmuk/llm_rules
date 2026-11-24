# LLM_JJ_PLAYBOOKS.md — jj procedures, read on demand

Longer jj procedures that `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §2 only
names: distributing conflict fixes into the commits that own them,
folding refactor commits into the features below, the user's `reviewed`
/ review-commit flow and its footguns, and re-slicing a long stack from
its final tree. Read the matching section before starting one of these
procedures, or when a rebase, squash or absorb left conflicts in a
stack. Every step here respects §2.2 of the ways of working (what you
may rewrite): rewrite own-session commits freely; everything else is
the user's.

The build commands in the examples are placeholders for the project's
own: the verify table is the project's `LLM/LLM_PROJECT.md` §Verify.

---

## 1. Distribute conflict fixes into the owning commits

Situation: a small change rebased UNDER a refactor stack conflicts in
several of the stack's commits. Wanted: each conflict resolved INSIDE
the commit that owns the hunk — no tip "merge" commit, the base change
stays where it is.

Bottom-up, one conflicted commit at a time:

1. `jj new <commit>` → edit the markers to THAT commit's intermediate
   state (not the final tip state).
2. `jj squash --use-destination-message` → descendants auto-rebase.
3. `jj log -r "conflicts()"` → repeat at the next conflicted commit.
4. Abandon the now-empty scratch / tip-resolution commits (own only).

Rules of thumb:

- Generated files, listed in the project's `LLM/LLM_PROJECT.md`
  §Generated files, are never hand-merged: regenerate at the commit
  whose job is regeneration (the command is named there), then `jj st`.
- Marker anatomy: `%%%%%%%` block = diff base→side (context + `+` lines
  = that side); `+++++++` block = the other side's snapshot verbatim;
  an empty block = that side deleted the region. Hand-editing `%%%%%%%`
  diffs is fiddly — replace the whole `<<<<<<< conflict N of M … >>>>>>>`
  block by script.
- Squash-cancellation casualties: when two `%%%%%%%` diffs are supposed
  to cancel (work squashed down via a helper commit), "keep the
  destination snapshot" silently drops the NON-cancelling parts — seen:
  a lost `}` impl closer, a lost struct-initialiser entry, lost import
  items, an insertion landing after an impl's `}`. After every
  resolve+squash round: (1) per-commit brace balance
  (`s.count('{') - s.count('}')`) over the stack, (2) the compile check
  and the tests at the stack head (the project's verify table, every
  touched area), (3) fix each casualty INTO its owning commit. Two
  amends touching the same line range in different commits re-conflict
  the stack — resolve again at the first toucher; expected.
- Amending a change that a merge already includes re-merges silently:
  conflicts can appear in the merge and every descendant with no error
  output. `jj log -r "conflicts()"` after every snapshot of such a
  change; treat any hit as stop-the-line. Diff a subagent's
  claimed-untouched files (`jj diff --stat`) before trusting its report
  — a broad formatter run is the recurring cause.
- Replacing a commit under a long stack: build the replacement commits
  on the old parent, `jj rebase -s <old>+ -d <new>`, then resolve roots
  one at a time. Record the pre-rebase ids first
  (`jj log --at-op <op> -r '<range>' -T 'change_id ++ commit_id'`):
  the resolution target of a file is the OLD commit's version with the
  replacement's delta applied, and `jj diff --from <old id> --to <change>
  --name-only` per commit proves every delta stays inside the replaced
  design's files. Compile-check every commit afterwards — a descendant
  can break without a textual conflict (a new test file calling an API a
  later commit retyped; the fix goes into that later commit).
- Never `jj new` an old (hidden) commit id to inspect it: it revives the
  commit and every hidden ancestor, and their change ids turn divergent.
  Read old versions with `jj file show -r <old id>` / `jj diff --from`;
  abandon revived ids by commit id if it happened.
- `jj workspace update-stale` with unsnapshotted edits resets the files
  to the last snapshot — the edits are gone. `jj st` in a workspace
  before anything elsewhere rewrites its `@`; back up edits before
  recovering a stale one.
- In old trees a build or a generation step can rewrite tracked files
  (lockfiles, resolved-package files, IDE project files, a package
  manifest): `jj restore --from @-` them before squashing a resolution.
  Keep the lock flag on a gate's build calls (`cargo … --locked`) so a
  lockfile change is caught instead of made.
- `nix develop` evaluates the default directory's CURRENT `flake.nix`;
  with the default `@` parked on an old tree whose flake no longer
  evaluates, use a fixed revision instead:
  `nix develop "git+file://$PWD?rev=<commit id>" --command bash -c 'cd
  ../<repo>-<task-slug> && …'`.

## 2. Fold refactor commits back into the feature commits below

Goal: review sees only surviving code; the end tree stays byte-identical.

1. User makes a `backup` bookmark / duplicate of the tip first. Verify
   `jj diff --from backup --to tip` is empty before AND after every round.
2. Per refactor commit, bottom-up: note its commit id (pre-absorb tree),
   then `jj absorb --from <refactor> --into '<featureRoot>::<featureTip>'`.
3. Absorb conflicts land at the FIRST feature commit that blamed the
   lines; resolving there cascades a NEW conflict to the NEXT commit that
   touched the file (its old diff re-applies against the resolution).
4. At the first toucher: hand-resolve, keeping only that commit's own
   surviving contribution (drop code the refactor deletes; check for
   already-absorbed replacement fns to avoid duplicates).
5. At the LAST feature toucher of a churned file: don't hand-merge —
   `jj new <toucher>; jj restore --from <pre-absorb-commit-id> <file>;
   jj squash`. Sets the file wholesale to post-refactor content; every
   non-toucher above cancels clean and the refactor's file diff empties.
6. Resolution edits always via `jj new <commit>` → edit → `jj squash`;
   never `jj resolve` (interactive).
7. End with empty refactor husks; abandon them (own) or hand the list to
   the user.

Down-move of a definition between commits: insert the byte-identical
fragment at the same anchor in the ancestor → the descendant's identical
addition auto-collapses on rebase. Up-move: remove + squash, then re-add
at the owner.

## 3. The user's review flow — `reviewed` / review commit

How AI-generated stacks get reviewed. The agent does not drive this; it
must not disturb it.

Two lines off `base` (the commit before the AI work):

- **AI branch** `base → … → last-ai` — the batched commits, kept; user
  corrections go on top. This is the deliverable.
- **Review staging** (scratch): `base → reviewed → review-commit`, where
  `review-commit` = all AI changes as ONE commit
  (`jj new -m "review commit" -r base; jj restore --from last-ai`) and
  `reviewed` = parent holding approved / corrected chunks, so they drop
  out of the review surface. `reviewed` may sit conflicted (throwaway).

Invariant: `review-commit` tree = `last-ai + corrections`, so
`jj diff --from last-ai --to review-commit` is exactly the corrections.

Filing a correction into `reviewed`:

- Regime 1 (leave `reviewed` conflicted): `jj new -r review-commit -m fix`
  → edit → `jj squash --from fix --into reviewed -u`. Corrected lines show
  as conflict blocks in `jj diff -r review-commit` (handled); still
  unreviewed hunks show as clean diffs (focus there). Self-resolves when
  `review-commit` is finally drained into `reviewed`.
- Regime 2 (keep `reviewed` clean): once a FILE is fully reviewed,
  `jj edit -r review-commit` → corrections in place →
  `jj squash --from review-commit --into reviewed <file> -u` (a MOVE).

Footguns (all verified):

- `jj squash` MOVES from `--from`: `--from review-commit` removes the
  change from it; `--from <child>` leaves review-commit untouched (a
  COPY) — rebasing that copy onto review-commit just empties the copy.
- `jj resolve --tool :ours/:theirs` picks WHOLE files — on a mixed file
  it silently drags unreviewed lines into `reviewed`. Never on mixed
  files.
- A clean `review-commit` on a conflicted `reviewed` is an illusion:
  resolving `reviewed` mid-flow re-conflicts review-commit. Let it drain
  at the end.
- Sub-hunk separation of reviewed vs unreviewed IS the review work; no
  jj op does it. Only interactive `jj split -i` / `jj diffedit` give
  sub-file granularity (user-run).

The `reviewed` bookmark on `main` marks the last user-reviewed commit;
everything above it is implemented but unreviewed (§2.2 of the ways of
working: read-only for the agent below it).

---

## 4. Re-slicing a long stack from its final tree

When a stack above `reviewed` has grown long — fixes far from the
features they fix, a subsystem a later commit removed again, plan docs
rewritten thirty times — rewriting it in place (rebase / squash /
split) is the wrong tool: every squash rebases everything above it, and
a file touched by forty commits conflicts on nearly every step. Rebuild
the stack forward from the final tree instead. Done right: zero
conflicts, final tree byte-identical.

The method (session scripts — `walk.py` (attribution and the per-commit
byte check), `generate.py` (the rebirth rule, the identity proof),
`topic_map.py`, `gate.py`, `commit.py` — kept in the scratchpad;
rebuild them from this description if needed):

1. **Freeze the stack.** One operator; no implementing session on top of
   the old tip until the new tip exists.
2. **Cache** every touched file at every commit and at `reviewed`
   (`jj file show -r`), plus `jj diff -r <c> --git --context 0` per
   commit. Use jj's own hunks for alignment — `difflib` pairs an
   original `}` with a later block's `}` and the intermediate trees stop
   parsing.
3. **Attribute lines** with a forward walk from `reviewed`: one record
   per line ever seen — `text`, `owner` (inserting commit),
   `deleted_by`, the hunk it came with, its replacements (`succ`) and
   what it replaced (`pred`). Insertions splice into a union order that
   stays consistent with every version. Sanity after every commit: the
   live lines equal that commit's file, byte for byte.
4. **Topic map by hand**: commit → topic; per-path overrides for mixed
   commits (split a mixed commit per path first; hunk granularity is
   the rare exception); `plan/**` and the rule docs → one final docs
   commit each; the removed subsystem → `DROP`. Fold a fix into its
   feature only when no other feature's code sits between them — a fix
   folded backwards past intervening work references things that do not
   exist yet.
5. **The dropped subsystem** is where all the difficulty lives. Its
   removal commit is *transparent*: a replacement inherits the owner of
   the line it replaced, a coupling line it deleted is never shown, and
   what the dropped work left behind (an enum variant, a contract type,
   parameter plumbing later features use) becomes one honest `…-kept`
   commit at the dropped topic's chronological slot. A dropped line is
   kept only if its lineage reaches the final tree — verbatim, retyped
   (a similar successor), or as part of a block a later feature rewrote
   rather than removed. A unit (blank-line-delimited run of one hunk)
   that names the dropped code and was made whole again by a later
   feature defers its verbatim survivors to that feature. The rest is a
   handful of explicit overrides: a brace whose role changed when the
   arms between it and `} =>` vanished (keyed by final line number), a
   renamed helper that needs a short alias shim in one intermediate
   commit, one test caller. Expect five to ten of those.
6. **Order topics chronologically** by first commit. Do not move a
   mechanical retype to the front and let later features inherit its
   form: the intermediate trees mix vocabularies and stop compiling. A
   rename stays a commit at its own position.
7. **Generate and gate**: for topic k a line is present iff
   `rank(owner) ≤ k < rank(deleted_by)`; write the tree into a fresh
   workspace at `reviewed`, run the compile check with resolved versions
   pinned (`cargo check --workspace --locked`; hard gate: libraries and
   binaries), then with the test targets (`--all-targets`; soft, logged
   — the original history had red test targets between a "types" commit
   and its implementation too), then `jj commit -m` with the primary
   commit's original description plus "folds …". If a topic fails
   alone, try the union with the next one before touching the map.
8. **Proof**: `jj diff --from <old tip> --to <new tip>` is empty. Then
   the full suites on the new tip. The old stack stays as a sibling
   until the user abandons it; every rewrite attempt leaves orphans
   (`jj log -r "conflicts()"`, descriptionless commits off `reviewed`)
   — abandon them at the end, never mid-loop.

### 4.1 Script traps

- **`jj` resolves file paths against the shell cwd, not against `-R`.**
  A script that shells out from the scratchpad gets `None` from every
  `jj file show`, seeds every file empty, and the byte-for-byte sanity
  check passes *vacuously* — absent equals absent. Run every jj call
  with `cwd=<workspace>`. A green sanity gate on an empty cache is the
  worst failure mode here; assert the base file is non-empty.
- **Seed each path from `reviewed` on first touch.** The first hunk's
  old-line numbers address the file as it already exists at the base,
  not an empty list.
- **Parse the diff header properly.** `diff --git a/P b/P` splits by
  halving only when the two paths are equal; jj emits real renames
  (`a/X b/Y` + `rename from` / `rename to`). Take the paths from
  `rename from`/`rename to`, else from `--- a/` / `+++ b/`, and treat a
  rename as delete(old) + add(new) owned by the renaming topic.
- **Apply a file's hunks in reverse old-line order**, against one
  live-index snapshot taken before the commit. Insert positions then
  stay valid as the record list grows.
- **Symlinks read as empty.** `new file mode 120000` — `jj file show`
  returns nothing, so attribute the link target whole, recreate with
  `os.symlink`, and skip the byte check for that path. Same treatment
  as binaries (asset catalogs, images).
- **Track file life events** (`add` / `delete` / rename) separately:
  a file can be created empty, and "no live lines" is then
  indistinguishable from "absent".
- **Key line-level overrides by record INDEX, never by text.** Moving a
  field to another topic must carry its doc comment and the blank line
  that closes it; matching on text moves *every* blank line in the file.
  Take whole blank-line-delimited runs of the owning rank, and pass the
  index into the topic map.
- **Exclude generated, untracked output from the per-topic gate.** A
  target that compiles against output only a generation step writes
  (an ignored generated module, a built native library) fails at every
  intermediate tree without that step; gate the rest of the workspace
  per topic, and run the generation step and that target's build once
  at the tip.
- **Writing trees from the final state drops the executable bit** of
  files that exist only mid-stack (they are deleted at the tip and
  recreated as 644). Take the modes from `jj file list -r reviewed -T
  'if(executable, path ++ "\n", "")'`; check with
  `jj diff -r <c> --git | grep '^old mode'` over the new stack.
- **Colocated repo, editor with git integration:** `jj commit` in a
  loop sometimes fails with `Could not acquire lock for index file`
  AFTER the commit landed, and the next jj command imports the stale git
  HEAD as a divergent, undescribed head. A retry must first check
  whether the commit landed (`@` empty, `@-` described); afterwards
  `jj log -r "divergent() | (reviewed:: & empty())"`, abandon the strays
  by commit id.

**A previous re-slice leaves evidence.** Markers like `TODO remove
after review [topic]` naming the topic each block belongs to, plus a
later commit that strips them all, are already actioned — the code had
been moved — so: drop every marker introduced above `reviewed`, keep
the ones inherited from `reviewed`, and the stripping commit keeps a
small honest job. Their text is also the best map for the folds a
hand-written topic map should make.

### 4.2 Dropping replaced designs

- **One rebirth rule for every hunk.** Give each hunk a slot S (its
  topic, or an override). A line it deletes dies at max(S, the line's
  birth); a line it inserts is born at max(S, the earliest birth of the
  lines it replaces — pair them inside the hunk with difflib), a pure
  insertion at max(S, its neighbours' births). For an order-preserving
  map this is exactly `rank(owner) ≤ k < rank(deleted_by)`; for merges
  and reorders it puts each rewrite where its context first exists.
  Prove identity first: every ORIGINAL tree regenerates byte for byte.
- **A design disappears cleanly only inside one topic.** Merge the
  introducer and its replacement (or the replacement into the
  introducer's slot). Making lines vanish across a topic boundary breaks
  partially rewritten files — method bodies without their header, a
  view body with half a modifier chain — and only the compiler or the
  analyzer shows it. Whole files and whole lineages (a line and every
  line that replaced it) vanish safely; a file whose nesting several
  commits edit needs a path slot that moves all of its edits to one
  topic.
- **Count stale vocabulary before hoisting a refactor.** If commits
  between the old design and the refactor rewrote lines that use the old
  type before the refactor touched them, hoisting leaves those lines
  stale; fixing them means writing code that never existed. Many such
  lines: keep the refactor as its own commit. A rename in between
  blocks every hoist across it.
- **A moved line may need its pre-upgrade text.** An import or API name
  that a toolchain / dependency upgrade rewrote gets a text override
  bounded by the upgrade commit. Imports and API renames only, never
  logic.
- **Gate by tree hash.** Hash each topic's code tree; gate only the
  topics whose tree matches no original commit (build files included,
  nix files excluded). Gate in plain copies of `reviewed` in the
  scratchpad, each with its own build directory (`CARGO_TARGET_DIR`),
  and commit in a fresh jj workspace with no build output, so nothing
  untracked is snapshotted.
- **A second toolchain gates the same way, in a copy of its own.** An
  analyzer that runs after code generation gets its own plain copy of
  `reviewed` and runs beside the first gate; relative path dependencies
  need an override file in the copy, because the copy sits elsewhere.
- **Pin resolved versions on the gate** (`cargo … --locked`, the
  equivalent flag of every other build tool). It catches a lockfile
  change an earlier split put one commit too early.

### 4.3 Moving the toolchain down; deltas at the tip

- **New work as one delta commit on the old tip.** Make that tree green
  first; it is the target head. Merge it into a new topic low in the
  order and let the rebirth rule place each hunk where its context is
  born. Override only what the rule gets wrong: a new type or re-export
  whose neighbours are older than its first user (slot = the user), a
  new line appended at a file end (its neighbour is ancient).
- **Never force an inserted line's birth earlier.** The union order puts
  it next to the CURRENT version of its neighbour; while an older
  version is live the line lands on the wrong side (an attribute after
  its field). Render the older versions with the line prefixed instead
  (ranged text override `[from, until)` on each ancestor, found through
  the aligned delete groups of the inserting hunks).
- **A toolchain moved below features leaves pre-upgrade lines stale.**
  The upgrade commit only rewrote lines alive at its old slot; older
  versions of those lines break until the feature that rewrote them.
  Ranged text overrides, API renames only.
- **Package removals wait for their users.** An update that removes a
  dependency gets a hunk slot at the commit deleting the last user.
- **A dependency line can be reborn late** (after a later manifest hunk
  replaced its neighbours): set its birth back to the toolchain commit
  when its consumer exists from there.
- **A toolchain has no baseline before its dependencies resolve** (a
  local-path dependency that older trees do not find): pick its
  checkpoints after the commit that makes them resolve, and compare
  error causes, not counts.

### 4.4 Folding device-test fixes into their features

- **Suggest fold targets from line ownership first.** Per new commit,
  count the inserting commits of the lines its hunks delete (or of a pure
  insertion's neighbours) — what `jj absorb` would pick. A good first map;
  it still needs the compile gate.
- **A fix folds only as far down as everything it names.** A fold into a
  commit before the type, field or command it uses fails the hard gate
  (a generated command variant, a view field). Fold into the closest
  commit after the last thing it names, or group the fixes as one topic
  at the first of them.
- **Tests carry the same constraint.** A test written against a later
  signature or mock turns the soft gate red; keep that fix at its own
  slot.
- **Pure insertions wait for their neighbours.** A new top-level fn or
  field beside later-born code is born late, while its callers are born
  at the slot. A per-commit `PURE_INSERT_AT_SLOT` births them at the
  slot; safe for items whose order inside the file does not matter.
- **The original history can be red too.** A test call ahead of its
  signature by twenty commits: a line-birth override (`BIRTH_OVERRIDE`,
  path + inserting commit + exact text → topic) makes the intermediate
  test targets green.
- **Docs commits hide code.** A "docs(plan)" commit can carry a
  toolchain bump in the nix files or a stray build artifact. List
  non-doc paths of docs commits before mapping them to the docs topics.

### 4.5 A stack that does not compile; dropping a design with dependents

- **Measure the original before gating.** When most intermediate
  commits of the original stack do not compile, a plain "does it
  compile" gate fails everywhere. Gate by error SETS: compile-check each
  new topic and the original commit it is keyed by, normalise (`path
  code message`, no line numbers) and look only at the errors the new
  tree adds — old trees carry toolchain noise, only the difference
  counts. Where a generator fails on both sides, the comparison says
  nothing.
- **Check the identity trees against jj for every path** before blaming
  the generator: `tree_at(k)` for paths a commit did not touch is not
  covered by the identity proof.
- **A design's dependents decide what can drop.** Rank commits by the
  share of their inserted lines that die later; then grep the symbols
  the candidates define in the lines of every OTHER commit before the
  replacement. A commit whose type or argument later code still uses
  has to stay; its follow-ups can drop.
- **Pure insertions beside dropped lines must not wait for them;
  insertions inside dropped code must.** Ignore a dropped commit's lines
  as neighbours only when the other neighbour is kept code
  (`IGNORE_NEIGHBOUR_COMMITS`); an insertion between two dropped lines
  follows them (a test argument added to a dropped test). Constants
  inserted between two dropped constants need `FORCE_BIRTH` (by hunk).
- **An argument list that a dropped commit extended** keeps the old last
  line, with its `))`, until the replacement: the new arguments of the
  commits in between need a ranged `TEXT_OVERRIDE` on that line (drop
  the `))`) plus `FORCE_BIRTH` with an explicit death for their own
  closing line.
- **A fold needs everything it names.** The compiler catches folds whose
  lines name fields or bindings of later commits. Move the fold up or
  keep the commit.
