# LLM_CODING_STYLE.md — cross-cutting code idioms

Code-level idioms that apply uniformly to every project and language:
the functional-first principles (§1), what production code must not
contain (§2) and the comment policy (§3). Read it in full before
writing code. The constructs that realise the rules in one language
live in the stack modules (`stacks/rust.md`, `stacks/swift.md`,
`stacks/flutter.md`); read the module of every language you generate.
For layer-specific patterns (domain method signatures, boundary async
rules, view / widget conventions, generator usage) see the project's
`LLM/LLM_AREA_PATTERNS.md`. When in doubt between two phrasings, match
the target file's neighborhood.

- **Always read**: §1 (functional-first principles), §2 (forbidden in
  production code), §3 (comments).
- **Mandatory read when generating Rust code**: §1 + `stacks/rust.md`
  (§1.R Rust constructs, keyed to the rules; §2.R the forbidden list).
- **Mandatory read when generating Swift code**: §1 + `stacks/swift.md`
  (§1.S Swift constructs, keyed to the rules; §2.S the forbidden list).
- **Mandatory read when generating Dart code**: §1 + `stacks/flutter.md`
  (§1.D Dart constructs, keyed to the rules).

Duplication between §1 and the per-language sections is intentional: §1
states the rule once with a small example; §1.R / §1.S / §1.D list
every language-specific construct that realises it, so you can scan one
language without cross-referencing.

The examples in §1 are written in Rust; the type names in them are
illustrative.

---

## 1. Functional-first — principles

Reach for combinators first; fall back to a loop only when it's
genuinely simpler.

### R1. Combinators when every element flows through the same shape of pipeline

Filter, map, reduce — pick the combinator that *names* the operation.

```rust
let tracking: Vec<TrackingNumber> = orders.iter().filter_map(|o| match o {
    Order::Shipped(number) => Some(*number),
    _ => None,
}).collect();
```

### R2. Loops win when the chain would dress up the truth

Use a loop when *one* of these is true — the combinator form only adds
ceremony:

- **Iteration variable is math, not a position.** `x` and `y` appear by
  name in formulae (a geometry computation, a transform, a hash). A
  `flat_map` form forces them into closure arguments and adds `move`
  boilerplate.
- **The collection grows during iteration.** Queue drains
  (`while let Some(event) = queue.pop_front()`) are a fixed-point
  computation over a mutable workset — no static sequence to chain over.
- **The stepping function itself can fail.**
  `while let Some(row) = rows.next()?` combines end-of-stream and error
  in one check; `Iterator::next` only models the first, so combinators
  would force materialising the whole stream.
- **Step size varies.** Parsers (`while !input.is_empty()`) consume a
  variable amount per iteration. Iterator adapters assume one-in-one-out.
- **The body's job is I/O.** Multiple `println!` / `await` calls with
  conditional branches (CLI dispatch, sequential effects with a
  cancellation or still-mounted check between them). A combinator
  forces building values then doing the I/O, fragmenting "what to do"
  across two layers.
- **The body mirrors a wire / external format.** Sequential writes of
  fixed fields (`put_u32` / `put_slice` calls, header lines, markers)
  *are* the protocol or file format. A `flat_map` chain hides the
  format inside closure machinery.
- **Per-element decisions need context.** `i > 0`, "what came before",
  produce-zero-or-more outputs (positional string building like
  `pascal_to_snake`, aligning two sequences in a merge). A chain has to
  thread state through tuple accumulators — exactly the
  named-binding-in-fold smell.

### R3. The decision rule — name the operation

`for_each` and `try_fold` are tempting "make it functional" rewrites but
only relabel a loop without naming what's happening. A combinator is a
real win only when there's a *name* for the operation: `filter_map`,
`partition`, `flat_map`, `find_map`, a clean `.map(build).fold(merge)`.
If the chain needs a stateful closure, a named accumulator, or `move`
captures just to compile, the loop is telling the truth and the chain is
dressing it up — keep the loop.

### R4. Two-stage `map(build).fold(merge)` over a single fold

Separate "what to produce per item" from "how to combine." This is the
canonical reduction shape across the codebase.

```rust
changed_paths.iter()
    .map(|path| self.read_effect(path))
    .fold(Outcome::default(), |mut acc, mut item| {
        acc.merge(&mut item);
        acc
    })
```

Collapse into a single `filter_map(...).fold(...)` *only* when the
alternative `Some/None` ceremony obscures the work, and not if you'll
later want to short-circuit with `try_fold` / `find`.

### R5. Dedicated construct for state-only / shape-only checks

When you only need yes/no on a variant — not its data — use the
language's dedicated check, not expanded match arms returning bools.

```rust
matches!(self.phase, SyncPhase::Ready)
```

### R6. One-sided conditional when the absent branch is a no-op

```rust
if let Some(draft) = self.drafts.get(&id) { … }
```

Escalate to early-return unwrapping (R7) when the absent branch
returns / errors / continues.

### R7. Early-return unwrapping over negative-arm match

```rust
let Some(document) = self.documents.get_mut(&path) else {
    return Err(DocumentsError::DocumentDoesNotExist { path });
};
```

Prefer over a `match` whose negative branch just returns `Err`.

### R8. Method references over identity lambdas

When the signature permits, pass the function itself.

```rust
xs.into_iter().filter_map(Result::ok)
```

### R9. `match` vs early-return — the value-construction test

The deciding question isn't "how many arms `return`?" — it's whether the
match still reads as a **balanced value-construction table**: each arm
produces the bound value (or exits) via a small, declarative expression,
all sitting at the same indent level.

**Keep `match`** when arms are visually parallel — happy-path arm is a
one-liner, even if other arms are error-returning guards. The match
shows "for each shape of the data, here is the outcome" at a glance:

```rust
let preview = match self.previews.get(&id) {
    None | Some(Preview::Unknown) => return Ok(self.read_preview_effect(id)),
    Some(Preview::None) => None,
    Some(Preview::Some(bytes)) => Some(bytes.clone()),
};
```

**Lift to early-return guards** when the happy-path arm dwarfs the
guards, contains nested control flow, or contains a *nested* `match` —
the value-table reading breaks down, and the imperative work belongs at
top level. Flatten the outer guard so the inner `match` stands at one
less indent.

For pure two-arm value-producing splits, `if let / else` vs `match` is a
taste call; pick one and stay consistent within a file.

### R10. `Option` / `Result` combinators over manual unwrap

- `.unwrap_or_else(|| …)` — always lazy when the default is a call.
- `.map_or(default, |x| …)` — absent ⇒ default, present ⇒ map.
- `.is_some_and(|x| …)` / `.is_ok_and(|x| …)` over
  `match … { Some(x) if … => true, _ => false }`.
- `.and_then(…)`, `.map_err(…)` for chaining.
- `?` for short-circuit propagation — never
  `match … { Err(e) => return Err(e), Ok(v) => v }`.

---

## 2. Forbidden in production code

Production code contains no construct that aborts instead of returning
an error. Each stack module lists its language's constructs and the
replacement for each (`stacks/rust.md` §2.R, `stacks/swift.md` §2.S);
the classes are the same everywhere:

- unwrapping that traps on absence or failure — propagate, or return a
  structured error;
- an explicit abort (`panic!`, `fatalError`) as error handling;
- unchecked indexing and slicing — use the checked accessors and
  iterators;
- an "unreachable" marker that silences the exhaustiveness check —
  allowed only where the compiler proves the arm cannot run; otherwise
  restructure the type.

Tests may unwrap freely.

Why it matters: an abort inside a step loses every change that was not
yet persisted, and where there is no unwinding it ends the whole
process. Which crates or targets enforce the list by lint is the
project's to say (`LLM/LLM_PROJECT.md`).

---

## 3. Comments — when and what

Default to **no comments**. Add one when:
- the domain rule is non-obvious;
- there's a hidden invariant that would surprise a reader;
- a workaround references an upstream issue;
- a file is kept for plausible reuse despite known shortcomings — a
  top-of-file comment states the limits and what future need would
  justify wiring it up. Prefer that over deleting the code.

Never:
- explain what well-named identifiers already say;
- reference the current task / PR / fix (those go in the commit
  message);
- leave commented-out code without a TODO explaining why.

Doc comments (`///`) on public types and commands are **expected** —
the IDE shows them, and a binding generator carries them into the
generated code of the other language. Keep them short: one or two
sentences plus an optional block for errors or a note, in the
language's form (see the stack module).
