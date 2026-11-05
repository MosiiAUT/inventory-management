<template>
  <!-- Grid shell: sidebar column + content column -->
  <div class="app-shell" :class="{ 'is-collapsed': collapsed }">
    <AppSidebar
      :collapsed="collapsed"
      :drawer-open="drawerOpen"
      @toggle-collapse="toggleCollapse"
    />

    <!-- Backdrop only rendered on mobile (CSS) while the drawer is open -->
    <div v-if="drawerOpen" class="sidebar-backdrop" @click="closeDrawer(true)" />

    <div class="app-content">
      <!--
        Former header passengers (LanguageSwitcher, ProfileMenu) live in the topbar rather
        than the sidebar footer: their dropdowns open downward/right-aligned and would be
        clipped by the sidebar's overflow and the viewport bottom.
      -->
      <header class="app-topbar">
        <button
          ref="menuButton"
          class="topbar-menu-btn"
          type="button"
          aria-controls="app-sidebar"
          :aria-expanded="drawerOpen"
          :aria-label="t('nav.openNavigation')"
          @click="openDrawer"
        >
          <svg viewBox="0 0 24 24" width="20" height="20" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" aria-hidden="true">
            <path d="M4 6h16M4 12h16M4 18h16" />
          </svg>
        </button>
        <div class="topbar-spacer" />
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </header>
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, computed, watch, nextTick } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'
import AppSidebar from './components/AppSidebar.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
    AppSidebar
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()

    // Sidebar collapse state persists across sessions; storage may be unavailable (private mode)
    const COLLAPSE_KEY = 'app-sidebar-collapsed'
    const readCollapsed = () => {
      try {
        return localStorage.getItem(COLLAPSE_KEY) === '1'
      } catch (e) {
        return false
      }
    }
    const collapsed = ref(readCollapsed())
    const toggleCollapse = () => {
      collapsed.value = !collapsed.value
      try {
        localStorage.setItem(COLLAPSE_KEY, collapsed.value ? '1' : '0')
      } catch (e) {
        // Ignore: the toggle still works for this session
      }
    }

    // Mobile drawer
    const drawerOpen = ref(false)
    const menuButton = ref(null)
    const openDrawer = () => { drawerOpen.value = true }
    // returnFocus is true only for user-initiated closes (Esc, backdrop); route changes
    // and viewport resizes must not steal focus from wherever the user is now
    let returnFocusOnClose = false
    const closeDrawer = (returnFocus = false) => {
      returnFocusOnClose = returnFocus === true
      drawerOpen.value = false
    }

    // Navigating closes the drawer (path only: query/hash changes should not)
    watch(() => route.path, () => closeDrawer())

    // Lock body scroll while the drawer is open; restore the previous value on close
    let previousOverflow = ''
    watch(drawerOpen, (open, wasOpen) => {
      if (open) {
        previousOverflow = document.body.style.overflow
        document.body.style.overflow = 'hidden'
      } else if (wasOpen) {
        document.body.style.overflow = previousOverflow
        // Return focus to the hamburger so keyboard users keep their place
        if (returnFocusOnClose) nextTick(() => menuButton.value && menuButton.value.focus())
        returnFocusOnClose = false
      }
    })

    const onKeydown = (e) => {
      if (e.key === 'Escape' && drawerOpen.value) closeDrawer(true)
    }

    // The drawer only exists below the desktop/tablet breakpoint. If the viewport grows past it
    // while the drawer is open, drawerOpen would stay true with the body scroll still locked
    // and no visible way to close it, so close it when the media query starts matching.
    const DRAWER_BREAKPOINT_QUERY = '(min-width: 768px)'
    let mql = null
    const onBreakpointChange = (e) => {
      if (e.matches) closeDrawer()
    }

    onMounted(() => {
      window.addEventListener('keydown', onKeydown)
      mql = window.matchMedia(DRAWER_BREAKPOINT_QUERY)
      mql.addEventListener('change', onBreakpointChange)
    })
    onBeforeUnmount(() => {
      window.removeEventListener('keydown', onKeydown)
      if (mql) mql.removeEventListener('change', onBreakpointChange)
      // Only touch overflow if we are the ones who locked it
      if (drawerOpen.value) document.body.style.overflow = previousOverflow
    })
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      collapsed,
      toggleCollapse,
      drawerOpen,
      menuButton,
      openDrawer,
      closeDrawer,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/*
 * SaaS design-token layer. Paste into the app's global stylesheet.
 * Map the color values to the project's existing palette instead of
 * introducing new brand colors; the defaults below are a neutral slate/blue.
 */
:root {
  /* Spacing: 4px scale */
  --space-0: 0;
  --space-1: 0.25rem;  /* 4px  */
  --space-2: 0.5rem;   /* 8px  */
  --space-3: 0.75rem;  /* 12px */
  --space-4: 1rem;     /* 16px */
  --space-5: 1.25rem;  /* 20px */
  --space-6: 1.5rem;   /* 24px */
  --space-8: 2rem;     /* 32px */
  --space-10: 2.5rem;  /* 40px */
  --space-12: 3rem;    /* 48px */

  /* Radius */
  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 10px;
  --radius-xl: 14px;
  --radius-full: 9999px;

  /* Shadows: keep subtle; borders do most of the separation work */
  --shadow-xs: 0 1px 2px rgba(15, 23, 42, 0.04);
  --shadow-sm: 0 1px 3px rgba(15, 23, 42, 0.06), 0 1px 2px rgba(15, 23, 42, 0.04);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 12px 32px rgba(15, 23, 42, 0.12);

  /* Neutrals */
  --color-bg: #f8fafc;            /* app background behind cards */
  --color-surface: #ffffff;       /* cards, sidebar, topbar */
  --color-surface-muted: #f1f5f9; /* hover rows, subtle fills */
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-text: #0f172a;
  --color-text-body: #1e293b;     /* default body copy (slightly softer than headings) */
  --color-text-muted: #64748b;
  --color-text-subtle: #94a3b8;

  /* Accent */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-primary-soft: #eff6ff;  /* active nav item background */
  --color-focus: rgba(37, 99, 235, 0.35);

  /* Status */
  --color-success: #16a34a;
  --color-success-soft: #dcfce7;
  --color-info: #2563eb;
  --color-info-soft: #dbeafe;
  --color-warning: #ca8a04;
  --color-warning-soft: #fef9c3;
  --color-danger: #dc2626;
  --color-danger-soft: #fee2e2;

  /* Typography */
  --font-sans: 'Inter', system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 0.9375rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.625rem;
  --leading-tight: 1.25;
  --leading-normal: 1.5;
  --tracking-tight: -0.02em;

  /* Layout */
  --sidebar-w: 248px;
  --sidebar-w-collapsed: 72px;
  --topbar-h: 60px;
  --content-max-w: 1600px;
  --page-padding: var(--space-8);

  /* Motion */
  --ease: cubic-bezier(0.2, 0, 0, 1);
  --duration: 150ms;
}

@media (max-width: 767px) {
  :root {
    --page-padding: var(--space-4);
  }
}

@media (prefers-reduced-motion: reduce) {
  :root {
    --duration: 0ms;
  }
}

/* Shared focus ring: apply to every interactive element */
:where(a, button, input, select, textarea, [tabindex]):focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px var(--color-focus);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: var(--font-sans);
  background: var(--color-bg);
  color: var(--color-text-body);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Grid shell. minmax(0, 1fr) lets wide tables/charts shrink instead of overflowing the grid */
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-w) minmax(0, 1fr);
  min-height: 100vh;
  transition: grid-template-columns var(--duration) var(--ease);
}

.app-shell.is-collapsed {
  grid-template-columns: var(--sidebar-w-collapsed) minmax(0, 1fr);
}

.app-content {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

/* z-index sits above FilterBar (90) so the profile/language dropdowns overlay it */
.app-topbar {
  position: sticky;
  top: 0;
  z-index: 95;
  display: flex;
  align-items: center;
  gap: var(--space-3);
  height: var(--topbar-h);
  padding: 0 var(--page-padding);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}

.topbar-spacer {
  flex: 1;
}

.topbar-menu-btn {
  display: none;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  background: none;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  color: var(--color-text-muted);
  cursor: pointer;
}

.topbar-menu-btn:hover {
  background: var(--color-surface-muted);
  color: var(--color-text);
}

.sidebar-backdrop {
  display: none;
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: var(--content-max-w);
  margin: 0 auto;
  padding: var(--space-6) var(--page-padding) var(--space-12);
}

@media (max-width: 1023px) {
  .app-shell,
  .app-shell.is-collapsed {
    grid-template-columns: var(--sidebar-w-collapsed) minmax(0, 1fr);
  }
}

@media (max-width: 767px) {
  .app-shell,
  .app-shell.is-collapsed {
    grid-template-columns: minmax(0, 1fr);
  }

  .topbar-menu-btn {
    display: inline-flex;
  }

  /* Compact passengers on small screens: avatar/icon only (names stay in the dropdowns) */
  .app-topbar .profile-name,
  .app-topbar .language-label {
    display: none;
  }

  .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: 100;
    background: rgba(15, 23, 42, 0.4);
  }
}

.page-header {
  margin-bottom: var(--space-6);
}

.page-header h2 {
  font-size: var(--text-2xl);
  font-weight: 700;
  color: var(--color-text);
  margin-bottom: var(--space-1);
  letter-spacing: var(--tracking-tight);
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--space-5);
  margin-bottom: var(--space-6);
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-lg);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: border-color var(--duration) var(--ease), box-shadow var(--duration) var(--ease);
}

.stat-card:hover {
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-md);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  font-variant-numeric: tabular-nums;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

.stat-card.info .stat-value {
  color: #2563eb;
}

.card {
  background: var(--color-surface);
  border-radius: var(--radius-lg);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-5);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--color-surface-muted);
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: var(--space-3) var(--space-4);
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: var(--space-3) var(--space-4);
  font-variant-numeric: tabular-nums;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: var(--color-bg);
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
