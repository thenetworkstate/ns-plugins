---
name: ns-design-system
description: Apply NS design system guidelines when creating or reviewing UI components. Use when asked to "follow design system", "style this component", "create a button", "design a form", or when building new UI features. Automatically triggered for frontend component work.
metadata:
  author: ns
  version: "1.0.0"
---

# NS Design System

## Live Reference

- **Kitchensink**: `/kitchensink` - Interactive component demos
- **Full docs**: `docs/frontend/design-system.md`
- **Figma**: [Global Components](https://www.figma.com/design/1NA1fWFw1kNy615I9rrhSD/ns.com--%7C--Learn?node-id=723-89072&m=dev)

## Brand Aesthetic

NS design is **clean, professional, action-oriented**:

- **Skeuomorphic depth**: Primary buttons have 3D shadows
- **Pill shapes**: Buttons/inputs use `rounded-full`
- **Generous whitespace**: `p-5` cards, `gap-6`+ sections
- **High contrast**: `iron-900` text on light backgrounds
- **Gold = premium**: Not for primary actions
- **Pillar identity**: Earn/Learn/Burn/Fun/Job each have distinct colors

**Don't**: Gold buttons. Excessive decoration. Borders everywhere.

**Do**: Clean interfaces. Skeuomorphic CTAs. Let content breathe.

## Colors

**Neutrals (Iron)**

- Disabled bg: `iron-25` #FCFBFC
- Hover: `iron-50` #F9F9F9
- Borders: `iron-200` #E4E4E7
- Placeholder: `iron-400` #A0A0AB
- Secondary text: `iron-600` #51525C
- Primary buttons: `iron-800` #26272B
- Headings: `iron-900` #1A1A1E

**Pillars**

- Earn: `#F7931A` (orange)
- Learn: `#1963FF` (blue)
- Burn: `#DC2626` (red)
- Fun: `#8A4FFF` (purple)
- Job: `#51525C` (gray)

**Semantic**

- Error: `#F04438`
- Success: `#099250`
- Info: `#155EEF`

**Gold** (use sparingly): `#B99463`, `#E5C083`, `#8E7143`

## Components

Base components in `apps/web/base/`. **Never modify—wrap instead**.

### NSButton

```tsx
<NSButton variant="primary" display="medium" type="button" onClick={fn}>
  Label
</NSButton>
```

Variants: `primary` (dark, skeuomorphic), `secondary` (white, border), `tertiary` (ghost)

Sizes: `small` (40px), `medium` (48px), `large` (56px)

### NSTextField

```tsx
<NSTextField display="medium" label="Email" type="email" required error={err} />
```

Sizes: `small` (32px), `medium` (48px), `large` (56px)

### NSCard

```tsx
<NSCard pillar="earn" earnType="bounty" href="/earn/123">
  <NSCard.Media src={img} aspectRatio={[4, 3]} />
  <NSCard.Title as="h3">Title</NSCard.Title>
</NSCard>
```

### NSCheckbox

```tsx
<NSCheckbox display="medium" label="Accept terms" checked={val} onChange={fn} />
```

## Quick Tokens

| Element | Token |
| ------- | ----- |
| Card padding | `p-5` |
| Section gaps | `gap-6` to `gap-8` |
| Form gaps | `gap-2` |
| Buttons | `rounded-full` |
| Cards | `rounded-2xl` |
| Primary shadow | `shadow-skeuomorphic` |

## Icons

- Primary: Lucide React
- Custom: `PillarIcon`, `CryptoCoinIcon`, `EarnTypeIcon` in `apps/web/base/icons/`

## Class Order

layout → positioning → sizing → spacing → background → border → typography → variants

```tsx
<div className="flex items-center w-full p-4 bg-white rounded-lg hover:bg-iron-50">
```

## Focus States

```css
focus-visible:ring-2 focus-visible:ring-iron-800 focus-visible:ring-offset-2
```
