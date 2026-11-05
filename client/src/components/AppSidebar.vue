<template>
  <aside id="app-sidebar" class="sidebar" :class="{ collapsed, open: drawerOpen }">
    <div class="sidebar-brand">
      <span class="brand-mark" aria-hidden="true">{{ brandInitial }}</span>
      <span class="brand-text">
        <strong>{{ t('nav.companyName') }}</strong>
        <small>{{ t('nav.subtitle') }}</small>
      </span>
    </div>

    <nav ref="navEl" class="sidebar-nav" :aria-label="t('nav.mainNavigation')">
      <router-link
        v-for="item in navItems"
        :key="item.to"
        :to="item.to"
        class="sidebar-link"
        :title="item.label"
        :aria-label="item.label"
        :active-class="item.to === '/' ? '' : 'is-active'"
        exact-active-class="is-active"
      >
        <!--
          Exact match for '/' so Overview is not highlighted on every page;
          other items use prefix matching so nested routes keep the parent active.
        -->
        <!-- Icon markup is a static, trusted string defined below (never user data) -->
        <span class="sidebar-icon" aria-hidden="true" v-html="item.icon" />
        <span class="sidebar-label">{{ item.label }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <slot name="footer" />
      <button
        type="button"
        class="sidebar-collapse"
        :aria-expanded="!collapsed"
        :aria-label="collapsed ? t('nav.expandSidebar') : t('nav.collapseSidebar')"
        @click="emit('toggle-collapse')"
      >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true" class="collapse-chevron">
          <path d="M15 18l-6-6 6-6" />
        </svg>
      </button>
    </div>
  </aside>
</template>

<script setup>
import { computed, ref, watch, nextTick } from 'vue'
import { useI18n } from '../composables/useI18n'

const props = defineProps({
  collapsed: { type: Boolean, default: false },
  drawerOpen: { type: Boolean, default: false }
})
const emit = defineEmits(['toggle-collapse'])

const { t } = useI18n()

// Move focus into the drawer when it opens so keyboard users land on the nav
const navEl = ref(null)
watch(() => props.drawerOpen, (open) => {
  if (!open) return
  nextTick(() => {
    const first = navEl.value && navEl.value.querySelector('a')
    if (first) first.focus()
  })
})

const brandInitial = computed(() => t('nav.companyName').charAt(0))

const svg = (inner) =>
  `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">${inner}</svg>`

const icons = {
  dashboard: svg('<rect x="3" y="3" width="7" height="9" rx="1"/><rect x="14" y="3" width="7" height="5" rx="1"/><rect x="14" y="12" width="7" height="9" rx="1"/><rect x="3" y="16" width="7" height="5" rx="1"/>'),
  inventory: svg('<path d="M21 8l-9-5-9 5 9 5 9-5z"/><path d="M3 8v8l9 5 9-5V8"/><path d="M12 13v8"/>'),
  orders: svg('<rect x="6" y="4" width="12" height="17" rx="2"/><path d="M9 4h6v3H9z"/><path d="M9 12h6M9 16h4"/>'),
  finance: svg('<rect x="3" y="6" width="18" height="14" rx="2"/><path d="M3 10h18"/><path d="M16 15h2"/>'),
  forecast: svg('<path d="M3 17l6-6 4 4 8-8"/><path d="M15 7h6v6"/>'),
  reports: svg('<path d="M14 3H6a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V9z"/><path d="M14 3v6h6"/><path d="M8 13h8M8 17h5"/>')
}

// Computed so labels react to language changes
const navItems = computed(() => [
  { to: '/', label: t('nav.overview'), icon: icons.dashboard },
  { to: '/inventory', label: t('nav.inventory'), icon: icons.inventory },
  { to: '/orders', label: t('nav.orders'), icon: icons.orders },
  { to: '/spending', label: t('nav.finance'), icon: icons.finance },
  { to: '/demand', label: t('nav.demandForecast'), icon: icons.forecast },
  { to: '/reports', label: t('nav.reports'), icon: icons.reports }
])
</script>

<style scoped>
.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding: var(--space-4) var(--space-3);
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  overflow-y: auto;
  overflow-x: hidden;
  z-index: 50;
}

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-2) var(--space-5);
}

.brand-mark {
  display: grid;
  place-items: center;
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  border-radius: var(--radius-md);
  background: var(--color-text);
  color: #fff;
  font-weight: 700;
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
  line-height: var(--leading-tight);
}

.brand-text strong {
  font-size: var(--text-base);
  color: var(--color-text);
  letter-spacing: var(--tracking-tight);
}

.brand-text small {
  font-size: var(--text-xs);
  color: var(--color-text-muted);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  flex: 1;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  height: 40px;
  padding: 0 var(--space-3);
  border-radius: var(--radius-md);
  color: var(--color-text-muted);
  font-size: var(--text-sm);
  font-weight: 500;
  text-decoration: none;
  white-space: nowrap;
  transition: background var(--duration) var(--ease), color var(--duration) var(--ease);
}

.sidebar-link:hover {
  background: var(--color-surface-muted);
  color: var(--color-text);
}

.sidebar-link.is-active {
  background: var(--color-primary-soft);
  color: var(--color-primary);
}

.sidebar-icon {
  display: inline-flex;
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

/* Icons are injected via v-html, so scoped styles need :deep to reach the svg */
.sidebar-icon :deep(svg) {
  width: 100%;
  height: 100%;
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-4);
  border-top: 1px solid var(--color-border);
}

.sidebar-collapse {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 36px;
  background: none;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  color: var(--color-text-muted);
  cursor: pointer;
  transition: background var(--duration) var(--ease), color var(--duration) var(--ease);
}

.sidebar-collapse:hover {
  background: var(--color-surface-muted);
  color: var(--color-text);
}

.collapse-chevron {
  width: 18px;
  height: 18px;
  transition: transform var(--duration) var(--ease);
}

.sidebar.collapsed .collapse-chevron {
  transform: rotate(180deg);
}

.sidebar.collapsed .sidebar-label,
.sidebar.collapsed .brand-text {
  display: none;
}

.sidebar.collapsed .sidebar-link,
.sidebar.collapsed .sidebar-brand {
  justify-content: center;
  padding-left: 0;
  padding-right: 0;
}

/* Tablet: always the icon rail, regardless of the collapsed flag */
@media (max-width: 1023px) {
  .sidebar .sidebar-label,
  .sidebar .brand-text {
    display: none;
  }

  .sidebar .sidebar-link,
  .sidebar .sidebar-brand {
    justify-content: center;
    padding-left: 0;
    padding-right: 0;
  }

  /* Collapse toggle is meaningless when the rail is forced */
  .sidebar-collapse {
    display: none;
  }
}

/* Mobile: off-canvas drawer that always shows labels.
   `.sidebar.sidebar.sidebar` triples the class specificity so a persisted collapsed flag
   cannot hide them regardless of source order. */
@media (max-width: 767px) {
  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: min(280px, 85vw);
    transform: translateX(-100%);
    /* visibility removes the closed drawer from the tab order; the delayed hide lets the slide-out finish */
    visibility: hidden;
    transition: transform var(--duration) var(--ease), visibility 0s linear var(--duration);
    box-shadow: var(--shadow-lg);
    z-index: 110;
  }

  .sidebar.open {
    transform: translateX(0);
    visibility: visible;
    transition: transform var(--duration) var(--ease), visibility 0s;
  }

  .sidebar.sidebar.sidebar .sidebar-label {
    display: inline;
  }

  .sidebar.sidebar.sidebar .brand-text {
    display: flex;
  }

  .sidebar.sidebar.sidebar .sidebar-link {
    justify-content: flex-start;
    padding: 0 var(--space-3);
  }

  .sidebar.sidebar.sidebar .sidebar-brand {
    justify-content: flex-start;
    padding: var(--space-2) var(--space-2) var(--space-5);
  }
}
</style>
