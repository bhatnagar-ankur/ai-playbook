# Figma Handoff Conventions — Full Reference

Full Figma naming, layer, and Code Connect/variable conventions for the
design-system-development skill. See `SKILL.md` for the short summary and a
representative snippet; this file holds the complete rules and examples. See
`references/examples.md` for a worked Figma naming example applied to Button.

---

## Component naming in Figma

```
Pattern: {ComponentName}/{Variant}/{State}

Examples:
Button/Primary/Default
Button/Primary/Hover
Button/Primary/Disabled
Button/Secondary/Default
Form Field/Default/Empty
Form Field/Default/Error
Form Field/Default/Filled
Modal/Default
Badge/Success
Badge/Danger
```

## Layer naming rules
- Use English, sentence case: `Button label` not `ButtonLabel` or `button label`
- Auto Layout frames: name after the component (`Card body`, `Input wrapper`)
- Hidden layers start with `_` (`_Mask`, `_Shadow layer`)
- Never leave Figma default names (`Frame 42`, `Rectangle 1`)

## Variables & Code Connect
- Map every semantic token to a Figma Variable in the matching collection (Color, Spacing, Typography)
- Use Code Connect to bind Figma components to their code counterparts so Inspect shows real props
- Export tokens as JSON via the Tokens Studio plugin or Figma Variables REST API for CI sync
