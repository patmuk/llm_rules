# swift.md — Swift idioms and forbidden constructs

The Swift constructs that realise the functional-first rules R1–R10 of
`LLM_GENERAL/LLM_CODING_STYLE.md` §1 (§1.S), the constructs forbidden
in production Swift (§2.S, the Swift list of that file's §2) and the
form of Swift doc comments. Read it in full whenever you generate Swift
code, after `LLM_GENERAL/LLM_CODING_STYLE.md`. For SwiftUI view
conventions see the project's `LLM/LLM_AREA_PATTERNS.md`. Type names in
the examples are illustrative.

---

## §1.S — Swift constructs

Listed in order of frequency, keyed to the rules.

### Sequence methods (R1, R3)

- `compactMap` — filter + transform when the two are *one operation*:
  case extraction (`guard case .shipped(let number) = order else { return nil }`),
  lookup-and-build (the lookup *is* the filter), optional-returning
  helpers chained together. **Don't** force `compactMap` when the
  predicate and transform are conceptually independent stages —
  `.filter(…).map(…)` reads better because the staging is the point.
  Eager chains build an array per stage: add `.lazy` when the sequence
  is large or the chain ends in an early exit (`first(where:)`), not by
  default.
- Two-way split — the standard library has no non-mutating partition:
  for sets, `let known = members.filter { contacts[$0] != nil }` then
  `members.subtracting(known)`; `Dictionary(grouping:by:)` when
  splitting by a key.
- `first(where:)` — single-element search; `lazy.compactMap(f).first`
  for search-and-transform.
- `reduce(into:)` — reductions. `reduce` rethrows, so a throwing closure
  is the fallible reduction. Default to the two-stage
  `.map(build).reduce(merge)` (R4). Collapse into a single reduce *only*
  when the alternative is `.compactMap { p ? build($0) : nil }.reduce(merge)`
  and the optional ceremony obscures the work.
- `contains(where:)` / `allSatisfy` — boolean reductions.
- `flatMap`, `zip`, `enumerated()` when shape demands. `enumerated()`
  yields offsets, not indices — over a slice use `zip(xs.indices, xs)`.

### `Optional` / `Result` (R10)

- `??` for defaults — the right-hand side is an autoclosure, so a call
  there is already lazy.
- `.map { … } ?? default` — absent ⇒ default, present ⇒ map.
- `.map(predicate) ?? false` for "present and matches".
- `?.`, `.flatMap { … }`, `Result.map` / `.mapError` for chaining.
- `try` for propagation — never `do { try f() } catch { throw error }`.
  `try?` only where absence *is* the meaning; it discards the error.

### Control-flow shapes

- **`guard let` / `guard case`** (R7) for early-return unwrapping —
  preferred over a `switch` whose negative branch just returns / throws
  / continues.
- **`==` on payload-free cases, `if case`** (R5) for state-only checks.
  When the same check appears in several places, give the enum a
  computed `var isShipped: Bool`.
- **`if let x`** (R6) for one-sided conditionals where the absent
  branch is a no-op.
- **`switch` vs `guard`** (R9) — value-construction test. Keep `switch`
  when cases are visually parallel; lift to `guard` when the happy path
  dwarfs guards or contains nested control flow / a nested `switch`.
- **`if` / `switch` expressions** for value mapping:

  ```swift
  let padding = switch size {
  case .sm: theme.spacing.sm
  case .md: theme.spacing.md
  case .lg: theme.spacing.lg
  }
  ```

### Function references (R8)

`.compactMap(Int.init)`, `.map(\.id)` — don't wrap a named function in
`{ f($0) }`.

```swift
strings.compactMap(Int.init)
notes.map(\.id)
```

### Imperative `for` reserved for sequential `await` (R2)

With `try Task.checkCancellation()` between iterations when the work
may be cancelled. Don't functionalise — a `TaskGroup` loses ordering.

### Tuples for ad-hoc grouped returns

Labelled tuples inside one function. Don't promote to a named struct
until used in 2+ places.

### Bridge records

The records a bridge generator emits (UniFFI records such as an
`OrderCard` or an `InvoiceView`) are plain Swift structs: use them as
they come, no wrapping `ViewModel` classes mirroring them a second
time. A derived display value
(`card.title ?? String(localized: "Untitled")`) is a computed property
in an `extension` next to the view that needs it.

---

## §2.S — Forbidden in production Swift

- Force unwrap `!` and implicitly unwrapped optionals (`T!`) — use
  `guard let`, `??`, or a thrown error.
- `try!` and `as!` — use `try` / `as?`.
- `fatalError`, `precondition`, `preconditionFailure` as error handling.
- `fatalError("unreachable")` / a `default:` that traps to silence
  exhaustiveness — restructure the type so the compiler proves the
  `switch` exhaustive.
- Unchecked subscripts and trapping accessors (`xs[i]`, `xs[a..<b]`,
  `removeFirst()` / `removeLast()` on a possibly empty collection) —
  use `first`, `last`, `popFirst()`, `indices.contains(i)`, sequence
  methods.
- Trapping arithmetic on external input (`+`, `*` overflow traps) — use
  `addingReportingOverflow` & co. or a value the bridge validated.

Tests may unwrap freely; prefer `try #require(x)` (Swift Testing),
which records a failure instead of crashing the test run.

Where a linter is in use, enable the matching rules (e.g. SwiftLint
`force_unwrapping`, `force_try`, `force_cast`); whether one is, the
project's `LLM/LLM_PROJECT.md` says. Why it matters: a trap kills the
whole app process — there is no unwinding, and every unsaved change is
lost.

---

## Doc comments

`///` doc comments on public types and commands are **expected**
(`LLM_GENERAL/LLM_CODING_STYLE.md` §3) — Xcode Quick Help and DocC
render them, and the ones written in Rust reach the generated Swift
through UniFFI as Quick Help. Keep them short: one or two sentences
plus an optional `- Throws:` / `- Note:` callout.
