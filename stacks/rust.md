# rust.md — Rust idioms and forbidden constructs

The Rust constructs that realise the functional-first rules R1–R10 of
`LLM_GENERAL/LLM_CODING_STYLE.md` §1 (§1.R), the constructs forbidden
in production Rust (§2.R, the Rust list of that file's §2) and the
form of Rust doc comments. Read it in full whenever you generate Rust
code, after `LLM_GENERAL/LLM_CODING_STYLE.md`. Type names in the
examples are illustrative.

---

## §1.R — Rust constructs

Listed in order of frequency, keyed to the rules.

### Iterator chains (R1, R3)

- `filter_map` — filter + transform when the two are *one operation*:
  variant extraction (`match o { Order::Shipped(n) => Some(*n), _ => None }`),
  lookup-and-build (the lookup *is* the filter), `Option`-returning
  helpers chained together. Body shape: `if predicate { Some(value) }
  else { None }` or a `match`. **Don't** force `filter_map` when the
  predicate and transform are conceptually independent stages (e.g. a
  date cutoff guard followed by a separate domain operation) —
  `.filter(…).map(…)` reads better because the staging is the point.
  Iterator adapters fuse, so there's no allocation cost to splitting;
  the choice is purely about communicating intent.
- `partition` — destructure into two typed bindings:
  `let (downloaded, pending): (Vec<_>, Vec<_>) =
  files.into_iter().partition(|f| f.is_downloaded);`.
- `find_map` — single-element search-and-transform.
- `try_fold` — fallible reductions returning `Result`, e.g. merging
  per-item outcomes where building one can fail.
- `fold` — infallible reductions. The merge shape:
  `|mut acc, mut item| { acc.merge(&mut item); acc }`. Default to the
  two-stage `.map(build).fold(merge)` (R4) — separates "what to produce
  per item" from "how to combine." Collapse into a single fold *only*
  when the alternative is
  `.filter_map(|x| if p { Some(build) } else { None }).fold(merge)` and
  the `Some/None` ceremony obscures the work. Don't collapse a clean
  `.map(build).fold(merge)` just for fewer stages — the named binding
  inside fold becomes a smell. Also don't collapse if you'll later want
  to short-circuit with `try_fold` / `find`.
- `any` / `all` — boolean reductions.
- `flat_map`, `chain`, `zip`, `enumerate` when shape demands.

### `Option` / `Result` combinators (R10)

- `.unwrap_or_else(|| …)` — always lazy when the default is a call.
- `.map_or(default, |x| …)` — absent ⇒ default, present ⇒ map.
- `.is_some_and(|x| …)` / `.is_ok_and(|x| …)` over
  `match … { Some(x) if … => true, _ => false }`.
- `.and_then(…)`, `.map_err(…)` for chaining.
- `?` for short-circuit propagation — never
  `match … { Err(e) => return Err(e), Ok(v) => v }`.

### Control-flow shapes

- **`let-else`** (R7) for early-return unwrapping — preferred over a
  `match` whose negative branch just returns / errors / continues.
- **`matches!`** (R5) for state-only checks —
  `matches!(self.phase, SyncPhase::Ready)`.
- **`if let Some(x) = …`** (R6) for one-sided conditionals where the
  absent branch is a no-op.
- **`match` vs `let-else`** (R9) — value-construction test. Keep
  `match` when arms are visually parallel; lift to `let-else` when the
  happy path dwarfs guards or contains nested control flow / a nested
  `match`.

### Method references (R8)

`.filter_map(Result::ok)` — don't wrap a named function in `|x| f(x)`.

### Strings and offsets

- Text is UTF-8 in Rust; a consumer on the other side of a bridge may
  count differently (a Swift / Apple text view counts UTF-16 units). An
  offset or length handed to such a consumer is in ITS unit, computed
  over the prefix (`encode_utf16().count()` for UTF-16) — never
  `str::len()`, never a byte index handed across the bridge.
- Slicing text at a marker goes through `str::find` + `split_at` on
  char boundaries, never arithmetic on byte positions.

---

## §2.R — Forbidden in production Rust

- `.unwrap()` / `.expect()` — use `?` or structured error.
- `panic!`.
- Indexing and slicing (`v[i]`, `&v[a..b]`, `&s[a..b]`) — use `.get()`,
  `.first()`, `split_*`, iterators, `str::get`.
- `unreachable!()` in non-trivially-exhaustive matches. OK when the
  compiler proves the arm cannot run.

Tests may use `.unwrap()` / `.expect("…")` freely.

Where a crate enforces this, clippy does it: `deny` on `unwrap_used`,
`expect_used`, `panic`, `indexing_slicing` outside `cfg(test)`, in the
crate's `lib.rs`. Which crates do is named in the project's
`LLM/LLM_PROJECT.md`. Why it matters: a panic inside a step poisons the
state the step holds, and what was not persisted is lost.

---

## Doc comments

`///` doc comments on public types and commands are **expected**
(`LLM_GENERAL/LLM_CODING_STYLE.md` §3) — a binding generator carries
them into the generated code of the UI language, where the IDE shows
them. Keep them short: one or two sentences plus an optional
`# Errors` / `# Note on …` block.

## Verify and format

- `cargo clippy --workspace --all-targets -- -D warnings` exits 0:
  tests, examples, benches and build scripts are linted too, not only
  the library and the binaries.
- A finding is fixed the way the lint asks, whoever wrote the code. An
  `#[allow]` / `#[expect]` carries a `reason = "…"` and stands only
  where the lint is wrong for the code, or on generated code — on the
  generated module's `mod` line, never in the generated file. A finding
  in what a macro generates is fixed in the macro.
- Format with `cargo fmt -p <crate>` for the crates you touched. Never
  `cargo fmt --all`: it follows path dependencies and reformats the
  sibling checkouts too.
- Which crates and suites: the project's `LLM/LLM_PROJECT.md` §Verify.

## Build output in a jj workspace

- Each workspace has its own `target/`, so its first build is cold.
- Never point `CARGO_TARGET_DIR` of one workspace at another's
  `target/`: workspace crates get the same artifact hash whatever their
  path, and cargo then links the OTHER tree's build as "fresh" — a
  symptom is an item "not a member" that the source has.
  `cargo clean -p <crates>` repairs it.

## Dependencies

- A sibling checkout named by path is not pinned: `Cargo.lock` records
  its crate version only. Its crate versions are bumped when its
  behaviour changes, so a commit here says which one it was built
  against.
