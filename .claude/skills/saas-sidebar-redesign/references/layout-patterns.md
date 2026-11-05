# Layout patterns

Reference implementations for the SaaS sidebar shell. Adapt them to the project: match its
API style (Options API vs `<script setup>`), i18n helper, and existing component names.

## 1. App shell (root layout)

```vue
<template>
  <!-- Grid shell: sidebar column + content column -->
  <div class="app-shell" :class="{ 'is-collapsed': collapsed, 'is-drawer-open': drawerOpen }">
    <AppSidebar
      :collapsed="collapsed"
      :drawer-open="drawerOpen"
      @toggle-collapse="toggleCollapse"
    />

    <!-- Backdrop only matters on mobile when the drawer is open -->
    <div v-if="drawerOpen" class="sidebar-backdrop" @click="drawerOpen = false" />

    <div class="app-content">
      <header class="app-topbar">
        <button
          class="topbar-menu-btn"
          type="button"
          :aria-expanded="drawerOpen"
          aria-label="Open navigation"
          @click="drawerOpen = true"
        >
          <!-- hamburger svg -->
        </button>
        <!-- Right side: former header passengers (profile, language). Topbar is the safe home
             because their dropdowns usually open downward (see SKILL.md Step 2). -->
        <div class="topbar-actions">
          <LanguageSwitcher />
          <ProfileMenu @show-profile-details="..." />
        </div>
      </header>
      <!-- Existing global filter bar (if any) stays here; update its sticky `top`
           to var(--topbar-h) and keep its z-index below the topbar's -->
      <FilterBar />
      <main class="app-main">
        <router-view />
      </main>
    </div>

    <!-- Modals stay at the root, unchanged -->
  </div>
</template>
```

```css
.app-shell {
  display: grid;
  /* minmax(0, 1fr) lets wide children (tables, charts) shrink instead of overflowing the grid */
  grid-template-columns: var(--sidebar-w) minmax(0, 1fr);
  min-height: 100vh;
  background: var(--color-bg);
  transition: grid-template-columns var(--duration) var(--ease);
}
.app-shell.is-collapsed { grid-template-columns: var(--sidebar-w-collapsed) minmax(0, 1fr); }

.app-content { display: flex; flex-direction: column; min-width: 0; }

.app-topbar {
  position: sticky; top: 0; z-index: 20;
  display: flex; align-items: center; gap: var(--space-4);
  min-height: var(--topbar-h);
  padding: var(--space-3) var(--page-padding);
  /* Solid surface by default; color-mix()/backdrop-filter blur only if the project's browser targets allow */
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}
.topbar-actions { margin-left: auto; display: flex; align-items: center; gap: var(--space-3); }
.topbar-menu-btn { display: none; }

.app-main {
  width: 100%;
  max-width: var(--content-max-w);
  margin: 0 auto;
  padding: var(--space-6) var(--page-padding) var(--space-12);
}

/* Tablet: force the icon rail */
@media (max-width: 1023px) {
  .app-shell, .app-shell.is-collapsed { grid-template-columns: var(--sidebar-w-collapsed) minmax(0, 1fr); }
}

/* Mobile: single column, sidebar becomes an off-canvas drawer */
@media (max-width: 767px) {
  .app-shell, .app-shell.is-collapsed { grid-template-columns: minmax(0, 1fr); }
  .topbar-menu-btn { display: inline-flex; }
  .sidebar-backdrop {
    position: fixed; inset: 0; z-index: 40;
    background: rgba(15, 23, 42, 0.4);
  }
}
```

Shell script logic (root component):
- `collapsed`: a `ref` initialized from `localStorage` (inside try/catch) and written back on toggle.
- `drawerOpen`: a `ref(false)`. Watch `route.path` and set it to `false` so navigation closes the drawer.
- Also close the drawer (and release the scroll lock) when the viewport grows past 767px: use `matchMedia('(min-width: 768px)')` with a change listener. Otherwise the body stays locked after a resize or rotation.
- On `onBeforeUnmount`, remove all listeners. Restore `document.body.style.overflow` only if the drawer is open.
- Focus: on open, focus the first nav link. On close, return focus to the hamburger only for user-initiated closes (Esc, backdrop or button), and pass a `returnFocus` flag.
- A `keydown` listener for Esc closes the drawer. Add it in `onMounted` and remove it in `onBeforeUnmount`.
- While `drawerOpen`, set `document.body.style.overflow = 'hidden'`, and restore it on close.

## 2. AppSidebar.vue skeleton

```vue
<template>
  <aside class="sidebar" :class="{ collapsed, open: drawerOpen }">
    <div class="sidebar-brand">
      <span class="brand-mark" aria-hidden="true">A</span>
      <!-- Hidden via CSS, not v-if: the mobile drawer must show it even when collapsed is persisted -->
      <span class="brand-text">
        <strong>{{ t('nav.companyName') }}</strong>
        <small>{{ t('nav.subtitle') }}</small>
      </span>
    </div>

    <nav class="sidebar-nav" aria-label="Main">
      <router-link
        v-for="item in navItems"
        :key="item.to"
        :to="item.to"
        class="sidebar-link"
        :title="item.label"
        :aria-label="item.label"
        active-class=""
        exact-active-class="is-active"
      >
        <!-- Use exact-active for '/', active-class for nested sections if routes nest -->
        <span class="sidebar-icon" aria-hidden="true" v-html="item.icon" />
        <span class="sidebar-label">{{ item.label }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <button
        type="button"
        class="sidebar-collapse"
        :aria-expanded="!collapsed"
        :aria-label="collapsed ? t('nav.expandSidebar') : t('nav.collapseSidebar')"
        @click="$emit('toggle-collapse')"
      >
        <!-- chevron svg -->
      </button>
    </div>
  </aside>
</template>
```

Notes:
- `navItems` is a `computed` so i18n labels react to language changes.
- Icons: prefer small inline SVG components or template-literal strings that you define in the file (static, trusted content). Never pass user data to `v-html`.
- With nested routes (e.g. `/orders/:id`), use `active-class="is-active"` on non-root items, so the parent stays highlighted, and `exact-active-class` only for `/`.

```css
.sidebar {
  position: sticky; top: 0; height: 100vh;
  display: flex; flex-direction: column; gap: var(--space-2);
  padding: var(--space-4) var(--space-3);
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  overflow-y: auto; overflow-x: hidden;
  z-index: 50;
}
.sidebar-brand { display: flex; align-items: center; gap: var(--space-3); padding: var(--space-2) var(--space-2) var(--space-5); }
.brand-mark {
  display: grid; place-items: center; flex-shrink: 0;
  width: 32px; height: 32px; border-radius: var(--radius-md);
  background: var(--color-text); color: #fff; font-weight: 700;
}
.sidebar-nav { display: flex; flex-direction: column; gap: var(--space-1); flex: 1; }
.sidebar-link {
  display: flex; align-items: center; gap: var(--space-3);
  height: 40px; padding: 0 var(--space-3);
  border-radius: var(--radius-md);
  color: var(--color-text-muted); font-size: var(--text-sm); font-weight: 500;
  text-decoration: none; white-space: nowrap;
  transition: background var(--duration) var(--ease), color var(--duration) var(--ease);
}
.sidebar-link:hover { background: var(--color-surface-muted); color: var(--color-text); }
.sidebar-link.is-active { background: var(--color-primary-soft); color: var(--color-primary); }
.sidebar-icon { display: inline-flex; width: 20px; height: 20px; flex-shrink: 0; }
/* v-html content is not reached by scoped styles */
.sidebar-icon :deep(svg) { width: 100%; height: 100%; }
.sidebar-footer { display: flex; flex-direction: column; gap: var(--space-2); padding-top: var(--space-4); border-top: 1px solid var(--color-border); }

.sidebar.collapsed .sidebar-label,
.sidebar.collapsed .brand-text { display: none; }
.sidebar.collapsed .sidebar-link { justify-content: center; padding: 0; }

@media (max-width: 1023px) {
  .sidebar .sidebar-label, .sidebar .brand-text { display: none; }
  .sidebar .sidebar-link { justify-content: center; padding: 0; }
  /* Rail is forced here, so the manual toggle would do nothing */
  .sidebar .sidebar-collapse { display: none; }
}

@media (max-width: 767px) {
  .sidebar {
    position: fixed; inset: 0 auto 0 0; width: min(280px, 85vw);
    transform: translateX(-100%);
    /* Closed drawer must not be tabbable; delay visibility so the slide-out still animates */
    visibility: hidden;
    transition: transform var(--duration) var(--ease), visibility 0s linear var(--duration);
    box-shadow: var(--shadow-lg);
  }
  .sidebar.open { transform: translateX(0); visibility: visible; transition: transform var(--duration) var(--ease); }
  /* Drawer always shows labels. Triple class (0,3,0+) beats `.sidebar.collapsed …` (0,2,0+)
     by specificity, not source order, so a persisted desktop collapse cannot hide drawer labels. */
  .sidebar.sidebar.sidebar .sidebar-label, .sidebar.sidebar.sidebar .brand-text { display: inline; }
  .sidebar.sidebar.sidebar .sidebar-link { justify-content: flex-start; padding: 0 var(--space-3); }
}
```

## 3. Icon set (outline, 24x24, stroke=currentColor, stroke-width 1.75)

| Use | Path(s) |
|---|---|
| Dashboard | `<rect x="3" y="3" width="7" height="9" rx="1"/><rect x="14" y="3" width="7" height="5" rx="1"/><rect x="14" y="12" width="7" height="9" rx="1"/><rect x="3" y="16" width="7" height="5" rx="1"/>` |
| Inventory / box | `<path d="M21 8l-9-5-9 5 9 5 9-5z"/><path d="M3 8v8l9 5 9-5V8"/><path d="M12 13v8"/>` |
| Orders / clipboard | `<rect x="6" y="4" width="12" height="17" rx="2"/><path d="M9 4h6v3H9z"/><path d="M9 12h6M9 16h4"/>` |
| Finance / wallet | `<rect x="3" y="6" width="18" height="14" rx="2"/><path d="M3 10h18"/><path d="M16 15h2"/>` |
| Forecast / trend | `<path d="M3 17l6-6 4 4 8-8"/><path d="M15 7h6v6"/>` |
| Reports / file | `<path d="M14 3H6a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V9z"/><path d="M14 3v6h6"/><path d="M8 13h8M8 17h5"/>` |
| Menu | `<path d="M4 6h16M4 12h16M4 18h16"/>` |
| Chevron left | `<path d="M15 18l-6-6 6-6"/>` |

Wrap each one in `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">…</svg>`.

## 4. Polish checklist

**Spacing**
- [ ] Page padding, section gaps, card padding, and grid gaps all use tokens (see the SKILL.md Step 6 rhythm).
- [ ] No stray `margin-top` hacks between sections. Use the parent `gap` instead.
- [ ] Form controls and buttons share a height (36px default, 32px compact).

**Typography**
- [ ] One h1/page-title style per page. Section titles use `--text-lg` at weight 600.
- [ ] Muted text uses `--color-text-muted`. No more than 3 text colors appear per view.
- [ ] Numbers in tables and KPIs use `font-variant-numeric: tabular-nums`.

**Surfaces**
- [ ] The app background is `--color-bg`, and cards are `--color-surface` with a border and `--shadow-sm`.
- [ ] Radius is consistent: `--radius-lg` for cards and modals, `--radius-md` for buttons, inputs, and nav items, `--radius-full` for badges.
- [ ] Status badges use soft background + strong text pairs (`--color-success-soft` / `--color-success`).

**States**
- [ ] Hover, `:focus-visible`, active, and disabled states exist on every interactive element.
- [ ] Loading states are skeletons or a spinner, and the layout does not jump.
- [ ] Empty states show a short message plus an optional action, never a blank table.

**Tables**
- [ ] Header row: `--text-xs`, uppercase or 600 weight, muted color, `--color-surface-muted` background.
- [ ] Rows keep the project's existing cell density (normalize only inconsistent values) and have a hover background.
- [ ] Wide tables sit in an `overflow-x: auto` wrapper.

**Accessibility**
- [ ] Text contrast is at least 4.5:1 (muted slate `#64748b` on white passes; `#94a3b8` is for decorative text only).
- [ ] Icon-only buttons have an `aria-label`. Collapsed sidebar links have a `title`.
- [ ] The drawer can be closed with Esc, and focus returns to the hamburger button.
