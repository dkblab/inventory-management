# Layout shell: sidebar, top bar, content

Reference markup and CSS for step 4. Adapt names to the app's existing conventions, and keep the existing `router-link` targets and i18n keys.

## Contents
- Markup
- Sidebar CSS
- Top bar and content CSS
- Responsive behavior
- Drawer state
- Porting checklist

## Markup

```vue
<template>
  <div class="app-shell" :class="{ 'drawer-open': drawerOpen }">
    <aside class="sidebar" id="primary-sidebar">
      <div class="sidebar-brand">
        <span class="brand-mark" aria-hidden="true"></span>
        <div class="brand-text">
          <span class="brand-name">{{ t('nav.companyName') }}</span>
          <span class="brand-sub">{{ t('nav.subtitle') }}</span>
        </div>
      </div>

      <nav class="sidebar-nav" aria-label="Main">
        <router-link v-for="item in navItems" :key="item.to" :to="item.to" class="nav-item">
          <component :is="item.icon" class="nav-icon" :size="18" aria-hidden="true" />
          <span class="nav-label">{{ t(item.labelKey) }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu @show-profile-details="..." @show-tasks="..." />
      </div>
    </aside>

    <div class="scrim" @click="drawerOpen = false"></div>

    <div class="app-content">
      <header class="topbar">
        <button class="menu-button" type="button" aria-label="Open menu"
                :aria-expanded="drawerOpen" aria-controls="primary-sidebar"
                @click="drawerOpen = !drawerOpen">
          <Menu :size="20" aria-hidden="true" />
        </button>
        <!-- Existing global controls (for example FilterBar) go here -->
      </header>
      <main class="page">
        <router-view />
      </main>
    </div>
  </div>
</template>
```

Drive `navItems` from one array (`{ to, labelKey, icon }`) built from the existing nav links, so the sidebar and the router stay in sync. If a nav label was hard-coded in the old header, add an i18n key for it in every locale file.

Use `router-link`'s default `router-link-exact-active` class for the active state, but for the home route (`/`) make sure exact matching is used so it is not highlighted on every page.

## Sidebar CSS

```css
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  background: var(--color-bg);
  font-family: var(--font-sans);
  color: var(--color-text);
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  padding: var(--space-4);
  gap: var(--space-6);
  background: var(--color-sidebar-bg);
  color: var(--color-sidebar-text);
  z-index: var(--z-sidebar);
}

.sidebar-nav { display: flex; flex-direction: column; gap: var(--space-1); flex: 1; overflow-y: auto; }

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--color-sidebar-text);
  font-size: var(--text-base);
  font-weight: 500;
  text-decoration: none;
  transition: background 150ms ease, color 150ms ease;
}
.nav-item:hover { background: var(--color-sidebar-hover); color: var(--color-sidebar-text-active); }
.nav-item.router-link-exact-active {
  background: var(--color-sidebar-active);
  color: var(--color-sidebar-text-active);
}
.nav-item.router-link-exact-active .nav-icon { color: #a5b4fc; }

.sidebar-footer { display: flex; flex-direction: column; gap: var(--space-3); padding-top: var(--space-4); border-top: 1px solid rgba(255, 255, 255, 0.08); }
```

Components moved into the dark sidebar footer (language switcher, profile menu) often have light-theme colors baked in. Give them a restyle pass so text and borders stay legible on the dark background, or place them in the top bar instead if that is simpler.

## Top bar and content CSS

```css
.app-content { min-width: 0; display: flex; flex-direction: column; }

.topbar {
  position: sticky;
  top: 0;
  z-index: var(--z-topbar);
  display: flex;
  align-items: center;
  gap: var(--space-4);
  min-height: var(--topbar-height);
  padding: 0 var(--space-8);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}
.menu-button { display: none; }

.page {
  width: 100%;
  max-width: var(--page-max-width);
  margin: 0 auto;
  padding: var(--space-8);
}
```

`minmax(0, 1fr)` and `min-width: 0` on the content column matter: without them wide tables and charts push the grid wider than the viewport and the page scrolls horizontally.

If the old layout had a sticky filter bar under the nav, its `top` offset is now `var(--topbar-height)` (or the bar moves into the top bar itself). Search for `position: sticky` and `top:` and update each.

## Responsive behavior

```css
/* Collapse to an icon rail */
@media (max-width: 1024px) {
  .app-shell { grid-template-columns: var(--sidebar-width-collapsed) minmax(0, 1fr); }
  .brand-text, .nav-label { display: none; }
  .nav-item { justify-content: center; }
  .topbar { padding: 0 var(--space-6); }
  .page { padding: var(--space-6); }
}

/* Off-canvas drawer */
@media (max-width: 768px) {
  .app-shell { grid-template-columns: minmax(0, 1fr); }
  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: var(--sidebar-width);
    transform: translateX(-100%);
    transition: transform 200ms ease;
  }
  .brand-text, .nav-label { display: initial; }   /* labels return inside the drawer */
  .nav-item { justify-content: flex-start; }
  .drawer-open .sidebar { transform: translateX(0); }

  .scrim {
    position: fixed; inset: 0;
    background: rgba(15, 23, 42, 0.45);
    opacity: 0; pointer-events: none;
    transition: opacity 200ms ease;
    z-index: var(--z-scrim);
  }
  .drawer-open .scrim { opacity: 1; pointer-events: auto; }

  .menu-button { display: inline-flex; }
  .topbar { padding: 0 var(--space-4); }
  .page { padding: var(--space-4); }
}

@media (prefers-reduced-motion: reduce) {
  .sidebar, .scrim { transition: none; }
}
```

## Drawer state

Close the drawer on navigation, otherwise it stays over the page the user just chose: watch the route (`watch(() => route.path, () => { drawerOpen.value = false })` or the Options API equivalent) and close it. Also close it on `Escape`.

## Porting checklist

- Every old nav link has a sidebar item with the same target and label key.
- Language switcher, profile menu, and any task or notification triggers still reachable and their events still wired.
- Filter bar or other global controls placed in the top bar or at the top of the page, with sticky offsets updated.
- Modals mounted at the shell level still sit above sidebar and scrim (`--z-modal`).
- No leftover CSS for the removed `.top-nav`, `.nav-container`, `.nav-tabs`.
- Body or `.app` no longer sets a `padding-top` or `margin-top` that compensated for the old header.
