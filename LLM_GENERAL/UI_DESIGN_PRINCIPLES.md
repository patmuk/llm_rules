# UI_DESIGN_PRINCIPLES.md — Mobile UI Spacing & Sizing

Authoritative source for the spacing, sizing, type, icon and corner
radius scales, the touch minimums and the screen safe areas of a
project's mobile UI. Read it in full for any UI work that picks or
introduces a size, spacing, radius, icon dimension, type role, colour
or touch-target value. When introducing or changing a design value,
pick from the tables below first; only add a new token after confirming
nothing existing fits.

Units: pt (iOS) / dp (Android) / logical px — interchangeable for our
purposes.

---

## 1. The 8pt grid

Every spacing and size is a multiple of **8**. **4** is the only allowed
half-step.

| Step | Use |
|------|-----|
| 4    | details (half-step; icon↔label, title↔subtitle) |
| 8    | related items glued together |
| 12   | friends — items in a list, related form fields |
| 16   | default screen padding, gap between unrelated lines |
| 24   | acquaintances — groups inside a card, card↔card |
| 32   | between sections, focal element padding |
| 48   | hero block isolation |
| 64   | major divider between unrelated regions |

## 2. Spacing friendship (gap tells the story)

| Gap        | Relationship   | Examples                              |
|------------|----------------|---------------------------------------|
| 4–8        | family         | icon + label, title + subtitle        |
| 12–16      | friends        | list items, related form fields       |
| 24–32      | acquaintances  | card sections, card ↔ card            |
| 48+        | strangers      | hero ↔ body, unrelated regions        |

## 3. Touch targets

Finger pad ≈ 9 mm. Smaller = rage-taps.

| Size  | Meaning             |
|-------|---------------------|
| 44pt  | iOS minimum         |
| 48dp  | Android minimum     |
| 56pt  | comfortable / FAB   |

## 4. Screen safe areas

Reference values only — the UI framework applies the real insets
through its safe area; never hard-code them in layout.

### iOS (iPhone)
- Edge padding: 16–20 pt
- Status bar: 44–59 pt
- Home indicator: 34 pt bottom
- Tab bar: 49 + 34 pt
- Nav bar: 44 pt

### Android
- Edge padding: 16 dp
- Status bar: 24 dp
- Gesture area: ~16 dp bottom
- App bar: 56 dp
- Bottom nav: 80 dp

## 5. Type scale

Breaks the 8pt grid on purpose. Ratio ≈ 1.2×. Line-height ≈ size × 1.4
(body) or × 1.2 (headings).

| Size | Role               |
|------|--------------------|
| 34   | H1 — page title    |
| 24   | H2 — section head  |
| 20   | H3 — card title    |
| 17   | Body lg — emph.    |
| 15   | Body — default     |
| 13   | Small — caption    |
| 11   | XS — label / meta  |

## 6. Icon sizes

Store each icon on a square frame; leave ~2 pt breathing room inside.

| Size | Use                       |
|------|---------------------------|
| 16   | inline with text          |
| 20   | small buttons             |
| 24   | default everywhere        |
| 32   | feature / empty states    |
| 48+  | hero illustrations        |

## 7. Corner radius

Pick one family; mixing looks accidental.

| Radius | Use                          |
|--------|------------------------------|
| 4      | inputs, small chips (subtle) |
| 8      | buttons, small cards (default) |
| 12–16  | cards, modals (friendly)     |
| 24+    | pill buttons (soft)          |
| 50 %   | avatars, FAB (circular)      |

## 8. Six rules to live by

1. **Multiples of 8. Always.** 4 is a half-step.
2. **Fewer sizes.** 4–6 type sizes, 4–6 spacings.
3. **Related = closer.** If unsure, too loose.
4. **One hierarchy per screen.** Only one element should scream.
5. **Respect the thumb.** Primary actions in the bottom third.
6. **When in doubt, add space.** Amateur UI is cramped.

## 9. Size reference (ascending)

Spacing & text: 4, 8, 11, 12, 13, 15, 16, 17, 20, 24, 32, 34.
Touch & components: 44, 48, 49, 56, 64, 80.
Device widths: 320 (iPhone SE 1st gen), 360 (Galaxy S25),
375 (iPhone 12–15), 402 (iPhone 17 Pro), 430 (iPhone Pro Max).

---

## Project mapping

The canonical expression of these values is the project's theme file,
named in `LLM/LLM_PROJECT.md` §UI theme, with its token groups
(spacing, radii, sizes, text, colours, …). Views MUST read tokens by
name from it; never inline magic numbers, raw colour literals, or the
framework's system text roles. See the project's
`LLM/LLM_AREA_PATTERNS.md` (UI conventions) for enforcement.

Every design value — colour, font, length, position, opacity, shadow,
angle, timing — is a named token in the theme file. Views contain no
design numbers; what remains is structure (line limits, `0`, halves for
centring) and math. A new token snaps to the tables above; reuse an
existing token before adding one (rule 8.2 — "fewer sizes").

Values taken over from a design: lengths are snapped to the 8 pt grid
(4 pt half-step) and font sizes to the type scale §5, ties rounding up.
Stroke weights, opacities, shadows, angles and textures keep the
design's values.

Status surfaces come in pairs, a background and what stands on it, the
same roles in light and dark – reuse one before adding a colour. The
project's pairs are listed with its theme (`LLM/LLM_PROJECT.md`
§UI theme).
