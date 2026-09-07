# Full Worked Example — Button, End to End

This file walks one real component — `Button` — through every stage this skill
describes: token usage, the full API contract, Figma naming, and a completed
Component Documentation Template. Use it as the pattern to follow when building
out a new component from scratch.

---

## 1. Token usage

`Button` consumes only **semantic** tokens (never global tokens directly) and defines a
small set of **component tokens** scoped to itself, per the token hierarchy in
`references/tokens.md`:

```css
/* components/button.css */

.btn {
  /* Component tokens — scoped to Button, derived from semantic tokens */
  --btn-padding:          var(--space-interactive);        /* space-2 space-4 */
  --btn-radius:           var(--radius-interactive);       /* radius-md */
  --btn-font-size:        var(--font-size-base);
  --btn-font-weight:      var(--font-weight-medium);

  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);
  padding: var(--btn-padding);
  border-radius: var(--btn-radius);
  font-size: var(--btn-font-size);
  font-weight: var(--btn-font-weight);
  min-height: 24px; /* WCAG 2.2 AA touch target floor — see references/accessibility.md */
  min-width: 24px;
}

.btn--primary {
  background: var(--color-brand);
  color: var(--color-text-on-brand);
}

.btn--primary:hover {
  background: var(--color-brand-hover);
}

.btn--primary:focus-visible {
  outline: none;
  box-shadow: var(--focus-ring);
}

.btn--danger {
  background: var(--color-danger);
  color: var(--color-text-on-brand);
}

.btn--danger:focus-visible {
  outline: none;
  box-shadow: var(--focus-ring-danger);
}

.btn:disabled {
  background: var(--color-neutral-200);
  color: var(--color-text-disabled);
  cursor: not-allowed;
}

.btn {
  /* All interactive state changes use the shared transition + reduced-motion
     handling defined once in references/tokens.md — do not redeclare it here. */
  transition:
    background-color var(--duration-fast) var(--ease-default),
    box-shadow        var(--duration-fast) var(--ease-default);
}
```

Sizes map to the `sm` / `md` / `lg` scale via spacing tokens, never hard-coded pixels:

```css
.btn--sm { padding: var(--space-1) var(--space-3); font-size: var(--font-size-sm); }
.btn--md { padding: var(--space-2) var(--space-4); font-size: var(--font-size-base); }
.btn--lg { padding: var(--space-3) var(--space-6); font-size: var(--font-size-lg); }
```

---

## 2. Full API contract

```markdown
## Button

### Props
| Prop      | Type                                            | Default   | Description                   |
|-----------|--------------------------------------------------|-----------|--------------------------------|
| variant   | 'primary' \| 'secondary' \| 'ghost' \| 'danger'   | 'primary' | Visual style                   |
| size      | 'sm' \| 'md' \| 'lg'                              | 'md'      | Height and padding scale       |
| disabled  | boolean                                           | false     | Prevents interaction           |
| loading   | boolean                                           | false     | Shows spinner; disables click  |
| type      | 'button' \| 'submit' \| 'reset'                   | 'button'  | HTML button type               |
| icon      | string (icon name)                                | —         | Leading icon                   |
| iconTrail | string (icon name)                                | —         | Trailing icon                  |

### Events
| Event | Payload    | Description       |
|-------|------------|--------------------|
| click | MouseEvent | Standard click     |

### Slots
| Slot    | Description        |
|---------|---------------------|
| default | Button label text   |

### Accessibility
- Always renders a `<button>` element — never `<div>` or `<span>`
- `disabled` attribute prevents focus and announces state to screen readers
- `loading` adds `aria-busy="true"` and `aria-label="Loading…"` to the spinner
- Minimum touch target: 24×24px (WCAG 2.2 AA, SC 2.5.8); 44×44px preferred where layout
  allows (WCAG 2.1 AAA, SC 2.5.5) — see `references/accessibility.md`
```

---

## 3. Figma naming

Following the `{ComponentName}/{Variant}/{State}` pattern from `references/figma.md`:

```
Button/Primary/Default
Button/Primary/Hover
Button/Primary/Focus
Button/Primary/Disabled
Button/Primary/Loading
Button/Secondary/Default
Button/Secondary/Hover
Button/Danger/Default
Button/Danger/Hover
```

Layer names inside each component instance: `Button label` (text layer), `Icon leading`,
`Icon trailing`, `Spinner` (hidden layer prefixed `_Spinner` when not in the Loading state).
Every semantic colour used (`--color-brand`, `--color-brand-hover`, `--color-text-on-brand`,
`--color-danger`) is bound to a matching Figma Variable in the Color collection, and the
component is wired up via Code Connect so Inspect shows the real `variant` / `size` /
`disabled` / `loading` props instead of raw Figma layer properties.

---

## 4. Completed Component Documentation Template

```markdown
# Button

> A clickable control that triggers an action or submits a form.

## When to use
- Primary calls to action (submit a form, confirm a dialog, start a flow)
- Secondary or tertiary actions alongside a primary button (use `secondary` / `ghost`)
- Destructive actions that need explicit confirmation (use `danger`)

## When NOT to use
- Navigation to another page or view — use a link (`<a>`) styled as a button instead,
  so browser navigation semantics (open in new tab, right-click menu) keep working
- Toggling a persistent on/off state — use a Switch or Checkbox component instead

## Variants
| Variant   | Use for                                      |
|-----------|-----------------------------------------------|
| primary   | The single main action on a screen or section |
| secondary | Supporting actions alongside a primary button  |
| ghost     | Low-emphasis actions (e.g. "Cancel")           |
| danger    | Destructive or irreversible actions            |

## Sizes
| Size | Height | Use for                                  |
|------|--------|--------------------------------------------|
| sm   | 32px   | Dense UI (tables, toolbars)               |
| md   | 40px   | Default — most forms and dialogs          |
| lg   | 48px   | Hero CTAs, marketing surfaces             |

## States
Default, Hover, Focus, Active, Disabled, Loading, Error (danger variant only)

## API
See "Full API contract" above (Props / Events / Slots tables).

## Accessibility
- Role: native `button` (no `role` attribute needed — the element itself is a `<button>`)
- Keyboard: reachable via Tab; activates on Enter and Space
- Screen reader: announces the label text plus `disabled` or `aria-busy` state when set
- Focus management: Button never moves focus itself; if it triggers a dialog, the dialog
  is responsible for moving focus into itself on open and restoring it to the Button on close

## Do / Don't
| Do | Don't |
|----|-------|
| Use one primary button per view or form | Stack multiple primary buttons side by side |
| Disable + `loading` while an async action is in flight | Let the button be clicked twice during submission |
| Pair `danger` with a confirmation step for irreversible actions | Wire `danger` directly to a destructive action with no confirmation |

## Related components
- IconButton — icon-only variant for toolbars
- ButtonGroup — for a segmented set of related actions
- LinkButton — button-styled anchor for navigation
```
