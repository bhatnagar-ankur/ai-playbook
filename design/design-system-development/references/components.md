# Component API Patterns & Documentation — Full Reference

Full API contract template and component documentation template for the
design-system-development skill. See `SKILL.md` for the short summary and a
representative snippet; this file holds the complete templates. See
`references/examples.md` for both templates filled out for a real component
(Button) end to end.

---

## Component API Patterns

Every component has a documented API with: variants, sizes, states, slots, and events.

### API contract template

```markdown
## Button

### Props
| Prop      | Type                                        | Default     | Description                       |
|-----------|---------------------------------------------|-------------|-----------------------------------|
| variant   | 'primary' \| 'secondary' \| 'ghost' \| 'danger' | 'primary' | Visual style                      |
| size      | 'sm' \| 'md' \| 'lg'                        | 'md'        | Height and padding scale          |
| disabled  | boolean                                     | false       | Prevents interaction              |
| loading   | boolean                                     | false       | Shows spinner; disables click     |
| type      | 'button' \| 'submit' \| 'reset'             | 'button'    | HTML button type                  |
| icon      | string (icon name)                          | —           | Leading icon                      |
| iconTrail | string (icon name)                          | —           | Trailing icon                     |

### Events
| Event   | Payload | Description           |
|---------|---------|------------------------|
| click   | MouseEvent | Standard click     |

### Slots
| Slot    | Description           |
|---------|------------------------|
| default | Button label text     |

### Accessibility
- Always renders a `<button>` element — never use `<div>` or `<span>` for buttons
- `disabled` attribute prevents focus and announces state to screen readers
- `loading` adds `aria-busy="true"` and `aria-label="Loading…"` to the spinner
- Minimum touch target: 24×24px (WCAG 2.2 AA, SC 2.5.8 Target Size Minimum); 44×44px is
  the more comfortable WCAG 2.1 AAA target (SC 2.5.5) — prefer it when layout allows, but
  24×24px is the actual AA-level requirement. See `references/accessibility.md`.
```

### Component naming rules

- Component name: `PascalCase` in code, `kebab-case` for HTML element (`<app-button>`, `<ui-button>`)
- Variant prop: named by visual intent, not implementation (`primary` not `blue`)
- Size prop: `sm` / `md` / `lg` — never pixel values as prop values
- Boolean props: positive framing (`disabled` not `enabled`, `loading` not `notReady`)
- Events: past-tense verbs (`change`, `submit`, `dismiss`) — never `on` prefix in the event name itself

---

## Component Documentation Template

Every component in the design system must include:

```markdown
# {ComponentName}

> One-sentence description of the component's purpose.

## When to use
- [Use case 1]
- [Use case 2]

## When NOT to use
- [Anti-pattern 1 — link to alternative]

## Variants
[Table or descriptions of each variant]

## Sizes
[Table or descriptions: sm / md / lg]

## States
Default, Hover, Focus, Active, Disabled, Loading, Error

## API
[Props / slots / events table]

## Accessibility
- Role: [e.g. `button`, `dialog`, `listitem`]
- Keyboard: [Tab, Enter, Space, Escape behaviours]
- Screen reader: [what is announced and when]
- Focus management: [where focus goes on open/close/submit]

## Do / Don't
| Do | Don't |
|----|-------|
| Use for primary actions | Stack multiple primary buttons |

## Related components
- [Link to related component]
```
