# Colour Contrast & Accessibility (WCAG 2.1 AA) — Full Reference

Full contrast requirements, colour/theming rules, and the WCAG 2.1 AA checklist
for the design-system-development skill. See `SKILL.md` for the short summary
and representative snippet; this file holds the complete tables and rules.

> **Correction note (this file was updated after a prior review found two
> factual errors in the original single-file version of this skill — the
> touch-target size and the cited contrast ratios below. Both are corrected
> here with the accurate WCAG figures.)**

---

## Colour System & Theming

### Contrast requirements (WCAG 2.1 AA)

| Pair | Min ratio | Notes |
|---|---|---|
| Normal text on background | 4.5:1 | Applies below the large-text threshold (see below) |
| Large text on background | 3:1 | Text ≥ 18pt (~24px), or ≥ 14pt bold (~18.66px / ~19px bold) |
| UI component / state indicator | 3:1 | Borders, icons, input outlines |
| Decorative / disabled | No requirement | Must not convey meaning alone |

**On the large-text threshold:** WCAG defines "large text" in **points**, not pixels — 18pt
regular weight or 14pt bold. Converted to CSS pixels (1pt = 1.333px) that is **≥ 24px regular**
or **≥ ~18.66px (commonly rounded to ~19px) when bold**. Writing the rule as a flat
"`font-size >= 18px`" is a common but inaccurate shorthand — at 18px regular weight, text is
still held to the stricter 4.5:1 ratio; only bold text at that size (or larger, in either
weight) qualifies for the relaxed 3:1 ratio.

```css
/* Verify with: https://webaim.org/resources/contrastchecker/ */

/* These pairs meet AA (recomputed from actual WCAG relative-luminance formula —
   see correction note above): */
/* --color-text-primary (#111827) on --color-surface (#fff)   → ~17.7:1 */
/* --color-brand (#2563eb) on --color-surface (#fff)          → ~5.15:1 */
/* --color-text-secondary (#4b5563) on --color-surface (#fff) → ~7.7:1 */
```

### Rules
- Never convey meaning by colour alone — always pair with an icon or text label
- Disabled states use opacity or muted colour — never the primary danger/success colour
- Never use `!important` to override theme values — cascade through token specificity instead.
  The single exception in this entire design system is the `prefers-reduced-motion` reset —
  see "Motion & reduced motion" below and `references/tokens.md` for the actual code block.

---

## Accessibility (WCAG 2.1 AA)

All components must meet WCAG 2.1 Level AA. Requirements are listed below.

**Non-negotiable requirements:**
- Keyboard navigable — all interactive elements reachable with Tab and activated with Enter/Space
- Visible focus indicator — `:focus-visible` outline, minimum 3px, contrasting colour
- Screen reader labels — every interactive element has an accessible name
- No keyboard traps — users can always exit any focused region with Escape or Tab
- Form errors — announced via `aria-live` or `aria-describedby`, not colour alone
- Images — informative images have descriptive `alt`; decorative images have `alt=""`
- Motion — respect `prefers-reduced-motion` (see "Motion & reduced motion" below)
- Touch targets — see "Touch target size" below

### Touch target size (corrected)

The original version of this skill stated "minimum 44×44px" as a WCAG 2.1 AA requirement.
That is incorrect: **44×44px is the WCAG 2.1 AAA-level criterion (SC 2.5.5 Target Size
Enhanced)** — it is a "nice to have," not required to pass AA. WCAG 2.1 AA itself has **no**
numeric touch-target requirement.

WCAG 2.2 (a later version of the standard) introduced an **AA-level** touch target
requirement: **SC 2.5.8 Target Size (Minimum) — 24×24 CSS pixels**, with exceptions for
inline text links, targets a user agent doesn't allow resizing, and targets in a sentence
or block of text.

**Use this framing going forward:**
- Minimum tappable area for interactive elements: **24×24px** (WCAG 2.2 AA, SC 2.5.8) — this
  is the actual current AA-level minimum
- Treat **44×44px** as a stretch target for primary/high-traffic controls (buttons, nav items)
  where layout allows — it is more comfortable for users with motor impairments and matches
  WCAG 2.1 AAA (SC 2.5.5), but is not required to claim AA conformance

### Motion & reduced motion

Respect `prefers-reduced-motion: reduce` for every animation and transition in the system.
The `@media (prefers-reduced-motion: reduce)` reset is the **only** place in this entire
design system where `!important` is permitted — see `references/tokens.md` for the full
code block and the motion duration rules that go with it. Every other stylesheet in this
skill must win specificity through normal cascade/token structure, never `!important`.
