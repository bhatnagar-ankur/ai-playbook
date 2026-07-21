---
author: Ankur Bhatnagar
---

# Figma — Full Reference

Figma variables, component variants, auto layout conventions, Code Connect, and handoff checklist.

---

## Table of Contents
1. [File Organisation](#file-organisation)
2. [Naming Conventions](#naming-conventions)
3. [Figma Variables (Tokens)](#figma-variables-tokens)
4. [Component Variants](#component-variants)
5. [Auto Layout Conventions](#auto-layout-conventions)
6. [Interactive Components](#interactive-components)
7. [Component Documentation in Figma](#component-documentation-in-figma)
8. [Code Connect](#code-connect)
9. [Handoff Checklist](#handoff-checklist)
10. [Design Review Checklist](#design-review-checklist)

---

## File Organisation

### Recommended file structure

```
Design System Library
├── 00 - Cover             (file thumbnail, version, changelog)
├── 01 - Foundations
│   ├── Colour             (token swatches with names)
│   ├── Typography         (type scale specimens)
│   ├── Spacing            (scale reference)
│   ├── Icons              (icon set, usage guide)
│   └── Motion             (animation curves, timing reference)
├── 02 - Components
│   ├── Actions            (Button, Link, IconButton)
│   ├── Forms              (Input, Textarea, Select, Checkbox, Radio, Toggle)
│   ├── Navigation         (Tabs, Breadcrumb, Pagination, Sidebar)
│   ├── Feedback           (Toast, Alert, Badge, Skeleton, Spinner)
│   ├── Overlays           (Modal, Drawer, Tooltip, Popover, Dropdown)
│   ├── Data Display       (Table, Card, Avatar, Divider, Tag)
│   └── Layout             (Grid, Stack, Cluster, Container)
├── 03 - Patterns
│   ├── Form patterns      (login, registration, checkout)
│   ├── Data tables        (sortable, pagination, bulk actions)
│   └── Empty states
└── 04 - Templates
    ├── Dashboard
    ├── List / Index page
    ├── Detail page
    └── Settings page
```

### Page structure within a component page

```
[Component Name]
├── Cover frame          (name, description, status badge)
├── Anatomy              (labelled exploded view)
├── Variants             (all variant × state combinations)
├── Sizing               (sm / md / lg side by side)
├── Do / Don't           (side-by-side usage examples with labels)
└── Specs                (spacing, colour, and type callouts)
```

---

## Naming Conventions

### Layer naming

Use **sentence case**. Be descriptive — layers are read by developers in code connect and inspect panels.

```
Good layer names:
  Button / Container
  Label text
  Left icon
  Error message
  Chevron icon

Bad layer names:
  Group 247
  Rectangle 3
  Frame 14
  icon_v2_FINAL
```

### Component naming: `{ComponentName}/{Variant}/{State}`

```
Button/Primary/Default
Button/Primary/Hover
Button/Primary/Disabled
Button/Secondary/Default
Button/Ghost/Focus

Input/Default/Empty
Input/Default/Filled
Input/Error/Empty
Input/Disabled/Filled

Badge/Success/Default
Badge/Warning/Default

Modal/Small/Default
Modal/Medium/Default

Toast/Success
Toast/Error
Toast/Warning
```

### Variable / token naming in Figma

Mirror the CSS token naming exactly so there is a direct mapping between design and code.

```
Collections and groups:

Primitives (global tokens)
  color/blue/50
  color/blue/100
  ...
  color/blue/900
  space/0
  space/1
  ...
  font-size/xs
  font-size/sm
  ...

Semantic
  color/brand/primary
  color/brand/primary-hover
  color/text/primary
  color/text/secondary
  color/surface/default
  color/border/default
  color/feedback/error
  ...

Component
  button/primary/bg
  button/primary/bg-hover
  button/primary/text
  input/border
  input/border-focus
  ...
```

---

## Figma Variables (Tokens)

Figma Variables (2023+) replace Styles for token management. Set up three collections matching the three CSS token tiers.

### Collection 1: Primitives

Mode: `Light` only (raw values, no dark mode concept at this tier)

```
Group: color/blue
  50:   #eff6ff
  100:  #dbeafe
  200:  #bfdbfe
  ...
  900:  #1e3a8a

Group: color/neutral
  0:    #ffffff
  50:   #f9fafb
  ...
  950:  #030712

Group: space
  0:    0
  1:    4
  2:    8
  3:    12
  4:    16
  6:    24
  8:    32
  10:   40
  12:   48
  16:   64
  [etc.]

Group: font-size
  xs:   12
  sm:   14
  base: 16
  lg:   18
  xl:   20
  2xl:  24
  3xl:  30
  4xl:  36
```

### Collection 2: Semantic

Modes: `Light` and `Dark`

```
Group: color/brand
  primary:        → Primitives/color/blue/600  (Light)
                  → Primitives/color/blue/400  (Dark)
  primary-hover:  → Primitives/color/blue/700  (Light)
                  → Primitives/color/blue/300  (Dark)

Group: color/text
  primary:        → Primitives/color/neutral/900  (Light)
                  → Primitives/color/neutral/50   (Dark)
  secondary:      → Primitives/color/neutral/600  (Light)
                  → Primitives/color/neutral/400  (Dark)

Group: color/surface
  default:        → Primitives/color/neutral/0    (Light)
                  → Primitives/color/neutral/900  (Dark)
  raised:         → Primitives/color/neutral/0    (Light)
                  → Primitives/color/neutral/800  (Dark)

[All other semantic tokens follow the same pattern]
```

### Collection 3: Component

Modes: `Default` (inherits from Semantic)

```
Group: button
  primary/bg:         → Semantic/color/brand/primary
  primary/bg-hover:   → Semantic/color/brand/primary-hover
  primary/text:       → Semantic/color/text/inverse

Group: input
  border:             → Semantic/color/border/default
  border-hover:       → Semantic/color/border/strong
  border-focus:       → Semantic/color/border/focus
  border-error:       → Semantic/color/feedback/error-border
```

### Applying variables to components

- Set fill colour → use variable → select from Semantic or Component collection.
- For text, set text colour, font size, line height, and letter spacing all via variables.
- Use the `Padding` and `Gap` fields in auto layout — bind them to Primitives/space variables.
- For corner radius, bind to the appropriate `radius` variable.
- Never type raw hex values into fill fields — always use a variable.

---

## Component Variants

### Setting up variant properties

Use Figma's **Component Properties** panel (not the legacy Variants panel) for all new components.

Property types:
- `Variant` — for mutually exclusive options: `variant = primary | secondary | ghost`
- `Boolean` — for true/false toggles: `loading = true | false`, `disabled = true | false`
- `Text` — for editable string content: `label`, `placeholder`
- `Instance Swap` — for swappable nested components: `Left icon`, `Right icon`

```
Button component properties:
  Variant:        primary | secondary | ghost | destructive | link
  Size:           sm | md | lg
  State:          default | hover | active | focus | disabled | loading
  Loading:        Boolean (false)
  Full width:     Boolean (false)
  Left icon:      Instance swap (None)
  Right icon:     Instance swap (None)
  Label:          Text ("Button")
```

### State management

Model states as a `State` variant property:

```
State values: Default | Hover | Active | Focus | Disabled | Loading | Error | ReadOnly
```

- `Default` — no interaction
- `Hover` — pointer over
- `Active` — pointer down / key pressed
- `Focus` — keyboard focus (should show focus ring)
- `Disabled` — `aria-disabled="true"`; still visible but non-interactive
- `Loading` — busy; action pending
- `Error` — validation failure (for form fields)
- `ReadOnly` — visible, not editable

### Variant naming examples

```
Button
  Variant=Primary,   Size=MD, State=Default
  Variant=Primary,   Size=MD, State=Hover
  Variant=Primary,   Size=MD, State=Focus
  Variant=Primary,   Size=MD, State=Disabled
  Variant=Primary,   Size=MD, State=Loading
  Variant=Secondary, Size=MD, State=Default
  ...

Input
  Variant=Default, Size=MD, State=Empty
  Variant=Default, Size=MD, State=Filled
  Variant=Default, Size=MD, State=Focus
  Variant=Error,   Size=MD, State=Empty
  Variant=Error,   Size=MD, State=Filled
  Variant=Default, Size=MD, State=Disabled
  Variant=Default, Size=MD, State=ReadOnly
```

---

## Auto Layout Conventions

Auto layout is mandatory for all components and frames that ship in the library. Manual absolute positioning is allowed only for decorative overlapping elements.

### Direction

| Setting | When to use |
|---|---|
| `Horizontal` | Row of elements (button group, nav bar, form row) |
| `Vertical` | Column of elements (form field, card body, list) |
| `Wrap` | Tag group, chip list — items wrap when they overflow |

### Sizing

| Dimension | Setting | When |
|---|---|---|
| Width | `Fixed` | Component has a defined width (icon button 44px) |
| Width | `Hug` | Component width fits its content (button, badge) |
| Width | `Fill` | Component stretches to fill parent (full-width input) |
| Height | `Fixed` | Component has a defined height (button 40px, input 40px) |
| Height | `Hug` | Component height fits content (card, modal body) |
| Height | `Fill` | Rare — use in flex stretch scenarios |

### Spacing (Gap and Padding)

Always bind Gap and Padding to space variables:

```
Button (horizontal, hug × fixed):
  Padding: Top=0, Bottom=0, Left=space/4, Right=space/4
  Gap:     space/2

Form Field (vertical, fill × hug):
  Padding: 0 (children handle their own padding)
  Gap:     space/1-5

Card (vertical, fixed × hug):
  Padding: space/6
  Gap:     space/4
```

### Alignment

- Horizontal layout: `Align center` for same-height siblings; use `Baseline` for text + icon combinations.
- Vertical layout: `Align left` for stacked form elements; `Align center` for centred card content.
- Do not use absolute positioning for items that move with content — they break auto layout reflow.

### Min / Max width

Use min/max constraints to build responsive components:

```
Card:
  Width:     Fill (grow with grid)
  Min width: 240px
  Max width: 480px

Modal (Medium):
  Width:     Fill
  Max width: 560px
  Min width: 320px
```

---

## Interactive Components

Use Figma's **Prototype** panel to wire state transitions for presentations and user testing. Link variants to show hover, focus, and active states.

```
Button / Primary / Default  ──[While hovering]──►  Button / Primary / Hover
Button / Primary / Hover    ──[Mouse down]─────►   Button / Primary / Active
Button / Primary / Default  ──[Key: Tab]───────►   Button / Primary / Focus
```

Prototype connections:
- `While hovering` → swap to Hover variant
- `Mouse down` → swap to Active variant
- `Key: Tab` → swap to Focus variant
- `On click` → navigate to next frame / close overlay

---

## Component Documentation in Figma

Every component page must include an **Annotations** frame covering:

```
[Component name] — Annotations

Status:      Stable | Beta | Deprecated
Version:     1.2.0
Last updated:2024-03-15

---

When to use
  Short paragraph — 2-3 sentences max.

When NOT to use
  Bulleted list of anti-patterns.

Variants
  Table: Variant | Use case

Sizes
  Table: Size | Height | Typical use

States
  Table: State | Description

Accessibility
  - ARIA role
  - Keyboard interaction
  - Screen reader announcement
  - Required ARIA attributes

Do / Don't
  Side-by-side frame:
    [GREEN frame] Do: <description>
    [RED frame]   Don't: <description>

Related components
  Links to related Figma components
```

Use the `Figma annotation` plugin (or native sticky notes) in a dedicated `📋 Specs` section on each page for spacing, colour, and type callouts visible in Dev Mode.

---

## Code Connect

Code Connect links a Figma component to its production code so that developers see real component code in Figma's Dev Mode.

### Setup

```bash
# Install
npm install -D @figma/code-connect

# Authenticate (one-time)
npx figma connect auth

# Publish all connections
npx figma connect publish
```

### `figma.connect()` file

Create a `*.figma.tsx` file alongside every component:

```tsx
// src/components/Button/Button.figma.tsx

import figma from "@figma/code-connect";
import { Button } from "./Button";

figma.connect(
  Button,
  "https://www.figma.com/design/<FILE_KEY>?node-id=<NODE_ID>",
  {
    props: {
      // Map Figma variant property → React prop
      variant: figma.enum("Variant", {
        Primary:     "primary",
        Secondary:   "secondary",
        Ghost:       "ghost",
        Destructive: "destructive",
      }),
      size: figma.enum("Size", {
        SM: "sm",
        MD: "md",
        LG: "lg",
      }),
      disabled: figma.boolean("Disabled"),
      loading:  figma.boolean("Loading"),
      children: figma.string("Label"),
    },

    example: ({ variant, size, disabled, loading, children }) => (
      <Button
        variant={variant}
        size={size}
        disabled={disabled}
        loading={loading}
      >
        {children}
      </Button>
    ),
  }
);
```

```tsx
// src/components/FormField/FormField.figma.tsx

import figma from "@figma/code-connect";
import { FormField } from "./FormField";

figma.connect(
  FormField,
  "https://www.figma.com/design/<FILE_KEY>?node-id=<NODE_ID>",
  {
    props: {
      label:       figma.string("Label"),
      hint:        figma.string("Hint"),
      error:       figma.string("Error message"),
      disabled:    figma.boolean("Disabled"),
      required:    figma.boolean("Required"),
      placeholder: figma.string("Placeholder"),
    },

    example: ({ label, hint, error, disabled, required, placeholder }) => (
      <FormField
        id="field-id"
        label={label}
        hint={hint}
        error={error}
        disabled={disabled}
        required={required}
        placeholder={placeholder}
      />
    ),
  }
);
```

### Code Connect publishing

Run before every design system release:

```bash
# Dry run — preview what will be published
npx figma connect publish --dry-run

# Publish
npx figma connect publish

# Remove stale connections
npx figma connect unpublish --node-url "https://www.figma.com/..."
```

Include in CI: run `npx figma connect publish --dry-run` on PRs to validate connections are valid before merge.

---

## Handoff Checklist

Complete this checklist before marking a design ready for engineering.

### Design completeness
- [ ] All required variants and sizes designed (see component API)
- [ ] All states modelled: Default, Hover, Active, Focus, Disabled, Loading, Error (where applicable)
- [ ] Dark mode variants present (if using dual-mode library)
- [ ] Empty states designed (no data, loading, error)
- [ ] Responsive breakpoints covered (mobile 375px, tablet 768px, desktop 1440px minimum)
- [ ] All layers named in sentence case — no "Group 247" or "Rectangle 3"
- [ ] No detached components or overridden base styles
- [ ] All colours, typography, and spacing reference library variables (no raw hex values)

### Accessibility
- [ ] Contrast checked for all text on all backgrounds (using Figma plugin: Colour Contrast or A11y Focus Orderer)
- [ ] Focus states visible and meet contrast requirements (3:1 against adjacent colour)
- [ ] Touch target sizes >= 44 × 44 px for all interactive elements
- [ ] Annotations include ARIA role, keyboard interactions, and screen reader notes
- [ ] Error states use icon + text (not colour alone)
- [ ] Images have alt text annotations

### Content and copy
- [ ] Copy is final (not placeholder Lorem Ipsum)
- [ ] Character limits respected (test with longest likely string)
- [ ] Truncation behaviour specified for long text (ellipsis, wrap, clamp)
- [ ] Number and date formats match localisation requirements

### Specification annotations
- [ ] Spacing between all elements specified (via Dev Mode variables)
- [ ] Behaviour for variable-length content documented (wrap / scroll / truncate)
- [ ] Animation / transition specified (duration, easing, trigger)
- [ ] Interaction edge cases documented (error recovery, empty input submission)
- [ ] Component name and variant match the code component name exactly

### Delivery
- [ ] Frame is at a logical zoom level (100% or 50%) so devs see accurate sizes
- [ ] Component is published to the shared library (not just in a draft)
- [ ] Link to Storybook or equivalent live component demo included in annotations
- [ ] Code Connect `*.figma.tsx` file written (or Jira ticket created)
- [ ] Design review completed with at least one developer

---

## Design Review Checklist

Run before every design critique or handoff session.

**Visual consistency**
- [ ] Component uses only design system components — no custom one-offs
- [ ] Spacing follows the 4 px grid — no odd values (7px, 13px, 19px)
- [ ] Typography uses only the defined type scale — no ad-hoc font sizes
- [ ] Colours reference semantic tokens — not global primitive tokens directly
- [ ] Border radius follows the token scale

**Interaction design**
- [ ] Every interactive element has a hover, focus, and active state
- [ ] Destructive actions have a confirmation step
- [ ] Loading and error states are designed (not just "happy path")
- [ ] Success feedback is designed
- [ ] Form validation behaviour is documented (on-blur? On submit? Inline?)

**Layout and responsiveness**
- [ ] Content reflows gracefully at mobile width (375px)
- [ ] Long text does not break the layout (test with a long username, long address)
- [ ] Images have fixed aspect ratio containers or explicit fallback behaviour
- [ ] Grid columns are consistent with the layout grid

**Accessibility (quick check)**
- [ ] All interactive elements have a label (no unlabelled icon buttons)
- [ ] Focus order is logical (left to right, top to bottom in LTR layouts)
- [ ] Error messages are adjacent to the field they describe
- [ ] Decorative images have empty alt annotations
