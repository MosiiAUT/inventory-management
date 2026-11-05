---
name: saas-sidebar-redesign
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface - replaces a top navigation bar with a vertical left sidebar, introduces a design-token layer for consistent spacing, and applies a professional polish pass across views. Use when the user asks to "add a sidebar", "move the nav to the left", "replace the top nav", "SaaS redesign", "modernize the Vue UI", "make the app look more professional", or "unify spacing" in a Vue 3 project.
---

# SaaS Sidebar Redesign (Vue 3)

Turn a Vue 3 app with a horizontal top nav into a SaaS-style app shell:

```
+------------+-------------------------------------------+
|  Brand     |  Topbar: title / page controls / actions  |
|            +-------------------------------------------+
|  Nav       |                                           |
|  - item    |  Main content (scrolls)                   |
|  - item    |                                           |
|            |                                           |
|  Footer    |                                           |
|  (profile) |                                           |
+------------+-------------------------------------------+
```

The goal is a **visual and structural** redesign. Routes, data logic, API calls, i18n, emitted events, and modals must keep working exactly as before.

Bundled references (read them when you reach the step that needs them):
- `references/design-tokens.css`: drop-in `:root` token layer (spacing, radius, shadow, color, type, layout).
- `references/layout-patterns.md`: app-shell template, `AppSidebar.vue` skeleton, responsive rules, and the polish checklist.

---

## Step 1: Discover (read-only)

Before you change anything, build a short inventory. Read these:

1. **Project rules**: `CLAUDE.md` / `AGENTS.md`. Note design rules (palette, "no emojis", etc.), required subagents (e.g. "delegate `.vue` edits to X"), and test or run commands. **These override this skill's defaults.**
2. **Root layout**: usually `src/App.vue`, sometimes `src/layouts/*.vue` or a `DefaultLayout` component. Find the `<header>` / `<nav>` and every `router-link`.
3. **Router**: search for `createRouter` / `routes:`. List each path, name, and any `meta` (title, icon, hidden). The sidebar items must cover every route the old nav exposed, and no more.
4. **Header passengers**: components that currently live in the top bar (profile menu, language switcher, search, notifications, global filters). Each one must get a new home. **Never drop one.** For each passenger with a dropdown, record how it is positioned (e.g. `position: absolute; top: 100%`). A downward-opening dropdown placed at the bottom of a 100vh sidebar opens off-screen and is clipped by the sidebar's `overflow`.
   Also record every **sticky offset or z-index tied to the old header** (e.g. a filter bar with `top: 70px`). These must be updated to the new topbar height.
5. **Global styles**: where they live (`App.vue` unscoped `<style>`, `src/assets/*.css`, `main.css`). Note any existing CSS variables and the dominant colors, font sizes, paddings, and gaps (grep for repeated hex values and `rem`/`px` values).
6. **Tooling**: CSS approach (plain CSS, SCSS, Tailwind, UI library such as Vuetify/PrimeVue). If a UI library or Tailwind is present, express the design through *its* primitives and don't fight it with raw CSS. Check for an existing icon library. Note the Options API vs `<script setup>` style. Match the convention of the file you edit: extend an existing `setup()` in place rather than rewriting it. New components follow the style the project's other components use, and if project rules forbid mixing styles, follow them. Also check the browser support the project targets before you use `color-mix()` or `backdrop-filter`. If unsure, use solid surfaces.

Summarize the inventory for the user in a few bullets before you continue: routes, passengers and their new homes, where the styles live, and constraints.

## Step 2: Plan the shell

Default placement (adjust to the project):

| Element | New home |
|---|---|
| Brand / logo / app name | Sidebar top |
| Primary route links | Sidebar nav (group them if there are more than 6, with small uppercase group labels) |
| Profile / user menu | Topbar right (default). Use the sidebar footer only if its dropdown can open upward or sideways (`bottom: 100%` or `left: 100%`), or if you are allowed to change it to do so |
| Language / theme switchers | Same rule as the profile menu. Topbar right is the safe default |
| Global filters / page controls | Topbar or a sticky bar at the top of the content column |
| Page title | Topbar left (from route `meta.title`) or keep the per-view page header |

Layout mechanics:
- Use a CSS Grid shell: `grid-template-columns: var(--sidebar-w) minmax(0, 1fr)`. The `minmax(0, …)` stops wide tables from blowing out the grid.
- The sidebar is `position: sticky; top: 0; height: 100vh` with its own `overflow-y: auto`.
- The content column scrolls with the page. Any sticky topbar sits *inside* the content column.
- Remove the content `max-width` centering that existed for the top-nav layout, or keep a generous one (1440-1600px) inside the content column only.

## Step 3: Tokens first

Add the token layer from `references/design-tokens.css` to the global stylesheet **before** you touch components.

- **Map, don't replace**: if the project already has a palette (e.g. slate `#0f172a / #64748b / #e2e8f0`, accent `#2563eb`), set the token values to those colors. The redesign should feel like the same product, only more polished.
- Spacing uses a 4px scale (`--space-1` = 4px … `--space-12` = 48px). From here on, every new padding, margin, or gap uses a token.
- Keep existing status colors (success/info/warning/danger) and expose them as tokens. **Overwrite the token defaults with the project's actual values** (e.g. existing badge backgrounds), never the other way round. Leave existing badge styles alone unless they are inconsistent with each other.
- Only set `--font-sans` if the project has no font-family yet, or if you are mapping its existing one.
- Where the tokens go: the project's existing global stylesheet. If the global styles live in an unscoped `<style>` block in `App.vue`, put the token `:root` block at the top of that block. Delete CSS for the old header (`.top-nav`, `.nav-tabs`, etc.) once nothing references it (check with grep).

## Step 4: Build the components

Follow the skeletons in `references/layout-patterns.md`.

**`AppSidebar.vue`**
- Render items from a single `navItems` array (`{ to, label, icon }`). Never hand-copy links.
- Labels keep the project's i18n calls (`t('nav.x')`, `$t(...)`). Never hardcode text that was translated before.
- Active state: use `router-link`'s `active-class` / `exact-active-class`. Use exact matching for `/` so "Home" is not always active. Don't use manual `$route.path ===` comparisons unless the project needs them.
- Accessibility: `<nav aria-label="Main">`, `aria-current="page"` (router-link adds it on exact match), visible `:focus-visible` ring, and a collapse toggle button with `aria-expanded` and `aria-label`.
- Collapse state: keep it in a `ref`, persist it to `localStorage`, and wrap that access in try/catch.
- Icons: inline SVG (24x24 viewBox, `stroke="currentColor"`, `stroke-width="1.75"`, `aria-hidden="true"`). Add **no** icon dependency unless the project already uses one. Simple outline icons work well: home/grid, box, cart/clipboard, wallet, trending-up, file-text.
- The footer region holds the collapse toggle. Add a `#footer` slot only if a passenger actually moves there (see Step 2), and don't leave unused slots.
- Every link gets `:title="item.label"` at all widths. The rail can be forced by a media query, which the component's `collapsed` prop does not know about.
- Show and hide labels and brand text with **CSS only** (`display: none` under `.collapsed` and the media queries), never `v-if="!collapsed"`. Otherwise a persisted collapsed state hides the labels in the mobile drawer.
- Icons rendered via `v-html` are not reached by scoped styles. Size them with `:deep(svg) { width: 100%; height: 100% }`.
- Toggle and hamburger `aria-label`s go through i18n as well (add keys such as `nav.collapseSidebar`, `nav.expandSidebar`, `nav.openMenu` to every locale file).

**`AppTopbar.vue`** (optional; skip it if the views already have good page headers): page title, global filters, right-side actions, and the mobile hamburger.

**Root layout (`App.vue` or similar)**: replace the old `<header>` with the grid shell. Keep every modal, global listener, provide/inject, and `setup` logic untouched.

## Step 5: Responsive behavior

| Width | Sidebar |
|---|---|
| >= 1024px | Full sidebar (`--sidebar-w`, ~248px), user can collapse to rail |
| 768-1023px | Icon rail (`--sidebar-w-collapsed`, ~72px), labels hidden, `title` attribute on links |
| < 768px | Off-canvas drawer: hidden by default, hamburger in topbar opens it, backdrop click / Esc / route change closes it, body scroll locked while open |

The collapsible sidebar and the icons-only rail are **required**, not optional:
- Collapse toggle (desktop only): a chevron button with `aria-expanded`. Persist its state. Hide or disable it at 768-1023px, where the rail is forced by the media query, so the button never looks broken.
- In rail mode every link keeps a `title` tooltip, and footer passengers (profile, language) switch to a compact or icon-only form. Their dropdowns must open *outside* the rail (e.g. `left: 100%` / `position: fixed`), never clipped by the sidebar's `overflow`.
- Animate the grid column width with `transition: grid-template-columns`. `prefers-reduced-motion` disables it through `--duration: 0ms`.

Drawer correctness (all of these were real bugs in a test run):
- Close the drawer **in JS** when the viewport crosses back above the mobile breakpoint (`matchMedia` change listener). CSS hides the drawer, but the JS state and the body scroll lock otherwise persist.
- The closed drawer must not be focusable: add `visibility: hidden` to the closed state (transition it so the slide still animates), or use `inert`. On open, move focus to the first link. Restore focus to the hamburger only when the user closed the drawer (Esc, backdrop or button), not after a route change or resize.
- State overrides (drawer vs collapsed vs rail) must win **by specificity**, not source order. Scoped attributes add equal weight to both sides.
- Icon-only links need `aria-label`, not just `title`, because `display: none` labels leave the accessibility tree.
- Check that existing toolbars (filter bars etc.) still fit the narrower content column: add `flex-wrap`, and stack them on mobile.

On mobile the grid becomes a single column. Pages and tables must not scroll horizontally. Wrap wide tables in an `overflow-x: auto` container.

## Step 6: Spacing and polish pass

Go through every view and shared component using the checklist in `references/layout-patterns.md`. Key rules:
- **One rhythm**: page padding `--space-8`, section gap `--space-6`, card padding `--space-5`/`--space-6`, grid gaps `--space-5`.
- **One page-header pattern**: title (`--text-2xl`, 600-700 weight, tight letter-spacing), muted description, actions aligned right.
- **Cards**: `--radius-lg`, 1px `--color-border`, `--shadow-sm`, and no heavy shadows.
- **Tables**: sticky header where useful, `--text-sm`, keep the project's existing cell density (only normalize inconsistent values; don't grow compact 8px/12px tables to 12px/16px), subtle row hover, right-aligned numbers with `font-variant-numeric: tabular-nums`.
- **Interactive states**: every button, link, and row needs hover, `:focus-visible`, active, and disabled styles. Transitions take 150ms, and `prefers-reduced-motion` is respected.
- Replace repeated magic numbers with tokens. **Do not change template logic, computed properties, or data flow** while polishing.

Prefer editing shared/global classes (e.g. `.card`, `.page-header`, `.stats-grid`) over touching every view.

## Step 7: Guardrails

- Every route reachable before is still reachable, and no new routes are added.
- All header passengers are still rendered and still emit or handle the same events.
- No new runtime dependencies (`package.json` unchanged unless the user agrees).
- i18n is preserved, and new user-facing strings go through the project's i18n system. This includes aria-labels and any nav label that was hardcoded before (add the key to **every** locale file).
- `v-for` keys stay stable and unique (never `index`).
- Add short comments for non-obvious logic. Put a one-line comment directly above the rule or statement for: exact-active matching on `/`, the drawer's scroll lock and focus restore, the route-change watcher, the localStorage try/catch, `minmax(0,1fr)`, specificity overrides between the collapsed and drawer states, and any updated sticky offset or z-index.
- Respect project rules found in Step 1, e.g. delegating `.vue` work to a designated subagent.
- Don't commit unless the user asks.

## Step 8: Verify

1. **Build**: run the project's build (`npm run build` or the equivalent) and fix any errors.
2. **Tests**: run the existing frontend tests if any exist. Selectors that targeted the old nav (e.g. `.nav-tabs a`) may need updating. Update the selectors and keep the assertions.
3. **Browser** (Playwright MCP if available; otherwise give the user a manual checklist):
   - 1440px: full sidebar, active item highlighted, every route loads.
   - 1024px / 900px: rail mode, tooltips on icons.
   - 375px: drawer opens and closes (hamburger, backdrop, Esc, route change), no horizontal scroll.
   - Keyboard: Tab through the sidebar, focus ring is visible, Enter navigates.
   - Moved passengers work (profile menu opens, language switch changes labels, filters still filter).
4. Report to the user what changed (files and the decisions you made, such as where passengers moved) and anything you could not verify.
