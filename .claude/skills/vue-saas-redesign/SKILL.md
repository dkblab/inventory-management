---
name: vue-saas-redesign
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a vertical navigation sidebar on the left (replacing any top nav bar), a consistent spacing and color system, and a polished professional look. Use this skill whenever the user asks to modernize, restyle, refresh, or redesign a Vue app, move navigation to a sidebar or left rail, make the UI look like a SaaS product or admin dashboard, clean up inconsistent spacing or styling, or add a design system or design tokens to a Vue frontend, even if they never say "SaaS".
---

# Vue 3 SaaS Redesign

Turn a Vue 3 app with a top navigation bar and ad hoc styling into a SaaS-style interface: fixed left sidebar, a slim top bar for page-level controls, one shared design token set, and consistent cards, tables, badges, and spacing.

The goal is a visual and layout change only. Routes, props, emits, API calls, i18n keys, and chart logic must behave exactly as before. A redesign that quietly breaks a filter or a route is worse than no redesign, so keep each change at the styling and template-structure level.

## Why the work is ordered this way

Most Vue apps of this kind define shared classes (`.card`, `.badge`, `.stat-card`, tables) once in `App.vue` or a global stylesheet, and views reuse them. Restyling those globals and introducing tokens first gives the biggest consistency gain for the least risk. Doing per-view tweaks first leaves you with a half-redesigned app and many conflicting edits. So: tokens, then shell, then shared classes, then a sweep of component-scoped styles.

## Workflow

### 1. Audit before editing

Read, do not skim:
- The root shell (`App.vue`): header/nav markup, the router outlet, global `<style>` block, and any modals mounted there.
- `main.js` and the router file, to get the real route list and the order of nav items.
- The i18n setup (`useI18n`, locale files), so sidebar labels reuse existing keys. Hard-coded labels break translations.
- Global styles and where colors, radii, and spacing are hard-coded in view and component `<style>` blocks (`grep -rn "#[0-9a-fA-F]\{3,6\}" src` is a quick map).
- Anything positioned relative to the old top nav: `position: sticky; top: <nav height>`, `padding-top` on main content, `z-index` values for modals. These silently break when the nav moves.

Write down the nav items, the controls living in the old header (language switcher, profile menu, filters), and the sticky or fixed elements. You will place each of them deliberately.

### 2. Install icons

Use `@lucide/vue` (the maintained package; the older `lucide-vue-next` is deprecated on npm) and import icons individually (`import { LayoutDashboard } from '@lucide/vue'`) so unused icons are tree-shaken. Never use emojis as icons. If the project has an `.npmrc` pinning a registry, leave it alone, and do not commit lockfiles the project gitignores: public repos must not leak private registry URLs.

### 3. Add design tokens

Create one global token file (for example `src/styles/tokens.css`) imported once from `main.js`, and define every color, space, radius, shadow, and type size as a CSS variable. Read `references/design-tokens.md` for the full token set and the reasoning behind the scale. Everything after this step uses `var(--...)` instead of literals. That is what makes the result consistent and re-themeable.

### 4. Build the shell

Replace the top nav with this structure (full CSS and markup guidance in `references/layout-shell.md`):

```
.app-shell            grid: sidebar | content
  aside.sidebar       fixed, full height, brand at top, nav list, footer (profile / language)
  .app-content        scrolls independently
    header.topbar     page title area + global controls (filters, language, profile)
    main.page         padded, max-width container, <router-view />
```

Key decisions and why:
- **Sidebar width 248px, content fills the rest.** Wide enough for labels, narrow enough to leave the data room.
- **Active state is obvious**: tinted background plus accent-colored icon and text, not just an underline. Users scan a sidebar vertically, so the current page must be visible at a glance.
- **Responsive**: at 1024px and below, collapse to a 72px icon rail; at 768px and below, make the sidebar an off-canvas drawer opened by a hamburger button in the top bar, with a scrim. A fixed sidebar with no mobile behavior makes the app unusable on phones.
- **Move, don't delete, the old header controls.** The language switcher and profile menu go in the sidebar footer or top bar. Keep their components and emitted events untouched; only their placement changes.
- **Modals stay above everything**: give them a z-index higher than the sidebar and scrim.
- Use the router's `router-link` with its own active class (`router-link-exact-active`) rather than comparing `$route.path` by hand, unless the existing code has a reason for the manual check.

### 5. Restyle shared classes

Rewrite the global classes against tokens: page header, stat cards, cards and card headers, tables, badges and status chips, buttons, inputs and selects, empty and loading states. Keep class names unchanged so views need no template edits. Rules that give the polished look:
- One spacing scale (4px base). Card padding, gaps, and section margins all come from it. Mixed 14px/18px/22px values are what make UIs feel unpolished.
- Cards: white surface, 1px subtle border, small radius, very light shadow. No heavy shadows, no gradients on surfaces.
- Tables: sticky or clearly separated header row in small uppercase muted text, row hover tint, right-aligned numbers with tabular figures, hairline row dividers.
- Status colors only mean status (success, warning, danger, info). Do not reuse them decoratively.
- Focus rings on every interactive element. Do not remove outlines without replacing them.

### 6. Sweep component-scoped styles

Go through each view and component `<style>` block and swap hard-coded colors, radii, and spacing for tokens. Fix layout that assumed the old header (sticky offsets, top padding, full-viewport heights computed with the nav). Modals get the same radius, shadow, and padding as cards. Do not rewrite chart or data logic; if an SVG chart uses literal colors, map them to the status or accent tokens and leave the geometry alone.

If the project's instructions say to delegate `.vue` edits to a specialist agent (for example a `vue-expert` agent), follow that instruction for steps 4 to 6, and hand it this skill's reference files along with the audit notes.

### 7. Verify in a real browser

Do not declare success from the diff. Run the dev server and check:
- Every route renders inside the new shell with the correct nav item highlighted.
- Screenshots at roughly 1440px, 1024px, and 390px wide: no horizontal page scroll, no overlapped or clipped content, drawer opens and closes on mobile.
- The old header controls (language switch, profile menu, filters) still work and nothing in the browser console errors.
- Opening a modal from a page: it appears above the sidebar and scrim.
- `npm run build` succeeds.

If a Playwright MCP tool or browser tool is available, use it for the screenshots. Fix what you find, then re-check that route.

### 8. Report

Finish with a short summary: what changed (tokens, shell, shared classes, component sweep), files added, the dependency added, anything you deliberately left alone, and anything the user must decide. If the project documents its own design system (a palette or "no emojis" rule in a CLAUDE.md or README), say that the documented palette no longer matches and offer to update it. Do not edit those docs without asking.

## Constraints that protect the app

- No changes to routes, store/composable logic, API modules, or backend code.
- Keep every i18n key in use; add new keys to every locale file if you add any text (for example a "Menu" label for the hamburger button).
- Keep accessibility: the nav is a `<nav aria-label="Main">`, the drawer button has `aria-label` and `aria-expanded`, icon-only states keep a tooltip or `title`.
- Match the code style already in the project (Options vs Composition API, scoped styles). Do not migrate frameworks or add a CSS framework.
- Add a short comment where a style decision is not obvious, such as a z-index ordering or a breakpoint choice.
