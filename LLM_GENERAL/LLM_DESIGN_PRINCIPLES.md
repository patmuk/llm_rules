# LLM_DESIGN_PRINCIPLES.md — design principles

How we decide *what to build* — the principles that drive type design,
domain modelling, and the development process: making faulty state
unrepresentable (§1), Domain-Driven Design (§2) and Test-Driven
Development (§3). Read it in full when designing types, modelling a new
domain concept, or deciding how to structure tests. Distinct from
`LLM_GENERAL/LLM_CODING_STYLE.md`, which governs the local shape of
code (combinators vs loops, match vs let-else, comments). The
layer-specific patterns that realise these principles are in the
project's `LLM/LLM_AREA_PATTERNS.md`, the architectural rules in the
project's `LLM/LLM_REPO_MAP.md`, and what a language or a generator
adds or restricts in the stack modules the project selects
(`stacks/<file>`).

Type names in the examples are illustrative. The examples are written
in Rust; the principles hold in every language the project uses.

---

## 1. Make faulty state unrepresentable

If the type system cannot express an invalid value, no validation,
defensive check, or test is required to enforce the invariant. Reach
for the strongest of these tools the situation supports:

### 1.1 Algebraic data types over flag combinations

Distinct lifecycle states get distinct enum variants with dedicated
payloads. Never `Option<bool>`, status strings, or "valid only when
some other field is `None`".

```rust
enum Order {
    Draft(DraftOrder),
    Placed(PlacedOrder),
    Shipped(ShippedOrder),
    Cancelled(CancelledOrder),
}

enum Attachment {
    Unknown,
    None,
    Some(Vec<u8>),
}
```

A consumer that handles `Placed` cannot accidentally see `Cancelled`
fields — they don't exist on that arm. See the project's
`LLM/LLM_AREA_PATTERNS.md` for the layer-specific rule.

Use a nested enum, `Variant(SomeEnum)`, when one outer variant has
internal sub-cases that should themselves be
unrepresentable-when-mismatched.

A variant's name says what its payload means (`InStock` /
`OutOfStock`), not the process state (`Found` / `NotFound`). A variant
with one collection payload carries it directly; a wrapper struct
exists only where a tool demands it.

Where a persistence or bridge generator accepts only some variant
shapes, the stack module and the project's `LLM/LLM_REPO_MAP.md` list
them; a type the generator rejects is reshaped to a supported form, not
weakened to flags.

### 1.2 Validating constructors (`try_new`)

Newtypes wrap raw primitives so the *only* path to an instance enforces
the invariant. The unchecked form does not exist in the public API.

```rust
let name = Name::try_new(input)?;                // Result<Self, ValidationError>
let quantity = Quantity::try_new(input)?;
let day = CalendarDay::try_new(2026, 11, 3)?;
```

Once an instance exists, downstream code can trust the invariant — no
re-validation, no `assert!`, no defensive branches. A crate-private
`new_unchecked` for literals in a library's own tests is acceptable; it
never leaves the crate.

Private fields plus accessors are the strongest form — use it wherever
nothing forces the fields open. Some tools (a persistence generator, a
bridge that mirrors types) read fields directly and force them public;
which types that applies to is stack-driven and written in the stack
module and the project's `LLM/LLM_AREA_PATTERNS.md`. Validation still
happens in the `try_new` constructor there, and callers use it rather
than the struct literal: where fields need public access, avoid
accessing them directly nevertheless.

### 1.3 Transforming functions over destructive mutation

State transitions return a new value rather than mutating fields on
the old one. The old shape becomes inaccessible by construction.

```rust
fn cancel(order: PlacedOrder) -> CancelledOrder { … }
fn replacing_line(&self, line: LineNumber, text: &str) -> Result<Document, ValidationError> { … }
```

Compare with `order.is_cancelled = true` — that leaves all the
placed-only fields visible and reachable, and every consumer must
remember to re-check the flag. Likewise `document.lines[i] = text`
leaves every consumer to remember the index and whatever else must
change with it. The transformation removes the choice.

---

## 2. Domain-Driven Design (DDD)

We model the domain explicitly and keep it isolated from infrastructure.

- **Single source of truth** — the domain core (named in the project's
  `LLM/LLM_REPO_MAP.md`) owns all domain state, types, and rules.
  Shells (a UI, a CLI) only fire commands and queries; they never
  decide business outcomes.
- **Ubiquitous language** — type and method names are domain terms
  (`Invoice`, `LineItem`, `cancel_order`), not framework or storage
  jargon (`…Manager`, `…Entity`, `…Row`).
- **Aggregates own their invariants** — domain models mutate only via
  methods on the aggregate, never by reaching into fields from outside.
- **Pure libraries under the aggregate** — rules that need no state of
  their own (parsing, merging, arithmetic) live in pure libraries; the
  models call them.

For the architectural rules that enforce this (dependency layers, where
async lives, generated APIs) see the project's `LLM/LLM_REPO_MAP.md`.
For method-signature rules see the project's
`LLM/LLM_AREA_PATTERNS.md`.

---

## 3. Test-Driven Development (TDD)

Tests come first; they specify the behaviour before the code exists to
satisfy them.

- A new domain rule starts with a failing test in the relevant unit
  test module (or the project's integration-test suite for cross-model
  scenarios).
- The scenario / acceptance criterion lives as a comment above the
  test, not as a standalone plan file.
- Write code until the test passes; refactor under the green test.
- **One test, then one implementation.** A cycle is one failing test
  and the code that makes it pass, then the next test — vertical, never
  all tests first.
- **Tests sit at the seams agreed with the user** and go through public
  interfaces: the specification names the seams, a test calls what a
  caller would call and checks what a caller can observe.
- **The tests are the reference**: a scenario, once written, holds the
  behaviour. Never drop a scenario because it is inconvenient — a
  scenario that must change is a `!decide`
  (`LLM_GENERAL/LLM_WAYS_OF_WORKING.md`).

Anti-patterns, by name:

- **Coupled to the implementation** — the test reaches into private
  state or asserts how the code does it, so it breaks on a refactor
  that changes no behaviour.
- **Tautological** — the expected value is computed the way the code
  computes it, so the test passes whatever the code does. Expected
  values are written out as literals or derived independently.
- **All tests first** — every test written before any implementation;
  the tests then specify a guess, and nothing is learned from one cycle
  for the next.

For the test layout (unit vs cross-model vs full-runtime), the test
framework and helper locations see the project's
`LLM/LLM_AREA_PATTERNS.md` and the stack modules.
