# Design tokens

Define these once in a global stylesheet and reference them everywhere. Values are a starting point: a slate neutral ramp with an indigo accent, which reads as professional and works for data-heavy dashboards. Change the values, never the names, if the user wants a different brand color.

## Contents
- Color
- Spacing
- Radius and shadow
- Typography
- Layout constants
- Usage rules

## Color

```css
:root {
  /* Neutrals (slate) */
  --color-bg: #f8fafc;            /* app background behind cards */
  --color-surface: #ffffff;       /* cards, tables, modals */
  --color-surface-muted: #f1f5f9; /* table headers, hover tint */
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-text-subtle: #94a3b8;

  /* Sidebar */
  --color-sidebar-bg: #0f172a;
  --color-sidebar-text: #cbd5e1;
  --color-sidebar-text-active: #ffffff;
  --color-sidebar-hover: rgba(255, 255, 255, 0.06);
  --color-sidebar-active: rgba(99, 102, 241, 0.18);

  /* Accent (indigo) */
  --color-accent: #4f46e5;
  --color-accent-hover: #4338ca;
  --color-accent-soft: #eef2ff;

  /* Status: use only to communicate status */
  --color-success: #16a34a;  --color-success-soft: #dcfce7;
  --color-warning: #d97706;  --color-warning-soft: #fef3c7;
  --color-danger:  #dc2626;  --color-danger-soft:  #fee2e2;
  --color-info:    #2563eb;  --color-info-soft:    #dbeafe;
}
```

A dark sidebar against a light content area is the classic SaaS look and gives navigation its own visual zone. If the user prefers an all-light sidebar, swap the sidebar tokens to the surface and border tokens; nothing else changes.

Text on `--color-text-muted` over white is about 4.8:1, which passes AA for body text. Do not go lighter than `--color-text-subtle` for anything meant to be read.

## Spacing (4px base)

```css
:root {
  --space-1: 4px;   --space-2: 8px;   --space-3: 12px;  --space-4: 16px;
  --space-5: 20px;  --space-6: 24px;  --space-8: 32px;  --space-10: 40px;
}
```

Consistency comes from restricting choices. Card padding is `--space-5` or `--space-6`, gaps between cards are `--space-4` or `--space-6`, page padding is `--space-8` on desktop and `--space-4` on mobile. If a value is not on the scale, move it to the nearest step instead of adding a new one.

## Radius and shadow

```css
:root {
  --radius-sm: 6px;   /* inputs, buttons, badges */
  --radius-md: 10px;  /* cards, modals */
  --radius-full: 9999px;

  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.05);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);   /* dropdowns, hovered cards */
  --shadow-lg: 0 16px 40px rgba(15, 23, 42, 0.18);  /* modals */
}
```

## Typography

```css
:root {
  --font-sans: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  --text-xs: 12px;  --text-sm: 13px;  --text-base: 14px;
  --text-lg: 16px;  --text-xl: 20px;  --text-2xl: 24px;
  --leading-tight: 1.25;  --leading-normal: 1.5;
}
```

Base body text is 14px: dense enough for data tables, still readable. Page titles use `--text-2xl` at weight 600; card titles `--text-lg` at 600; labels and table headers `--text-xs` uppercase with letter-spacing 0.04em in the muted color. Use `font-variant-numeric: tabular-nums` on any numeric column or stat so digits align.

## Layout constants

```css
:root {
  --sidebar-width: 248px;
  --sidebar-width-collapsed: 72px;
  --topbar-height: 64px;
  --page-max-width: 1440px;

  --z-sidebar: 40;
  --z-scrim: 45;
  --z-topbar: 30;
  --z-dropdown: 60;
  --z-modal: 100;
}
```

Keeping z-indexes in one list prevents the classic bug where a modal opens behind the sidebar.

## Usage rules

- No literal hex codes, pixel paddings, or shadows in component styles after the redesign. Use tokens.
- Hover and focus states: hover uses `--color-surface-muted` or `--color-accent-hover`; focus uses `outline: 2px solid var(--color-accent); outline-offset: 2px`.
- Transitions: `150ms ease` on color, background, border, and shadow. Nothing longer, nothing that moves layout.
- Respect `prefers-reduced-motion` for the drawer slide.
