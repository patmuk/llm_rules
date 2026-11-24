# flutter.md — Dart / Flutter idioms

The Dart constructs that realise the functional-first rules R1–R10 of
`LLM_GENERAL/LLM_CODING_STYLE.md` §1 (§1.D) and the note on doc
comments behind a Rust bridge. Read it in full whenever you generate
Dart code, after `LLM_GENERAL/LLM_CODING_STYLE.md`. For Flutter widget
conventions see the project's `LLM/LLM_AREA_PATTERNS.md`. Type names in
the examples are illustrative.

---

## §1.D — Dart constructs

Keyed to the rules.

### Functional methods (R1, R8)

`.where().map().toList()`, `.fold()`, `.any()`, `.every()`,
`.firstWhere()`. Pass method references where the type works:
`_queue.where(pendingIds.contains).toList()`. Don't write
`(x) => set.contains(x)` when `set.contains` works.

### Collection-for and collection-if (R1)

Inside `[]`, `{}`, `{key: value}` rather than building imperatively:

```dart
final pendingIds = {
  for (final order in orders)
    if (order.isPending) order.id,
};

final ordersById = {for (final order in allOrders) order.id: order};

Column(children: [
  for (final t in tags) _TagTile(tag: t),
  _NewTagTile(),
]);
```

### Switch expressions (R1)

Dart 3, for value mapping:

```dart
final pad = switch (size) {
  ButtonSize.sm => const EdgeInsets.symmetric(horizontal: 14, vertical: 8),
  ButtonSize.md => const EdgeInsets.symmetric(horizontal: 22, vertical: 12),
  ButtonSize.lg => const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
};
```

Records destructure cleanly: `final (icon, color) = switch (state) {
… };`.

### Imperative `for` reserved for sequential `await` (R2)

With `if (!mounted) return;` between iterations. Don't functionalise —
`Future.wait` loses ordering, `forEach` can't `await`.

### Cascade `..` for builder-style configuration

Mutable objects (Paint setup, controller setup). Use over a temp +
chained assignments.

### `maybeWhen` for variant matching with a default

```dart
final isShipped = order.maybeWhen(
  shipped: (_) => true,
  orElse: () => false,
);
```

Use full `.when(...)` only when each variant has a meaningfully
different result.

### Records for ad-hoc grouped returns

Inside one function. Don't promote to a named class until used in 2+
places.

---

## Doc comments

`///` doc comments on the public Rust types and commands behind the
bridge are **expected** (`LLM_GENERAL/LLM_CODING_STYLE.md` §3) — they
propagate to the FRB-generated Dart bindings as IDE tooltips.
