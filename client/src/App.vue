<template>
  <div class="app-shell" :class="{ 'drawer-open': drawerOpen, 'is-collapsed': collapsed }">
    <aside id="primary-sidebar" class="sidebar">
      <div class="sidebar-brand">
        <span class="brand-mark" aria-hidden="true"><Boxes :size="18" /></span>
        <div class="brand-text">
          <span class="brand-name">{{ t('nav.companyName') }}</span>
          <span class="brand-sub">{{ t('nav.subtitle') }}</span>
        </div>
      </div>

      <nav class="sidebar-nav" aria-label="Main">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="nav-item"
          :title="t(item.labelKey)"
          :aria-label="t(item.labelKey)"
        >
          <component :is="item.icon" class="nav-icon" :size="18" aria-hidden="true" />
          <span class="nav-label">{{ t(item.labelKey) }}</span>
        </router-link>
      </nav>

      <!-- Hidden in drawer mode (<=768px) where the full-width drawer is always used -->
      <button
        class="sidebar-toggle"
        type="button"
        :aria-label="toggleLabel"
        :title="toggleLabel"
        :aria-expanded="!collapsed"
        aria-controls="primary-sidebar"
        @click="toggleCollapsed"
      >
        <PanelLeftOpen v-if="collapsed" class="nav-icon" :size="18" aria-hidden="true" />
        <PanelLeftClose v-else class="nav-icon" :size="18" aria-hidden="true" />
        <span class="nav-label">{{ toggleLabel }}</span>
      </button>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div class="scrim" @click="drawerOpen = false"></div>

    <div class="app-content">
      <header class="topbar">
        <button
          class="menu-button"
          type="button"
          :aria-label="t('nav.menu')"
          :aria-expanded="drawerOpen"
          aria-controls="primary-sidebar"
          @click="drawerOpen = !drawerOpen"
        >
          <Menu :size="20" aria-hidden="true" />
        </button>
        <FilterBar />
      </header>
      <main class="page">
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
import { ref, watch, onMounted, onBeforeUnmount, computed } from 'vue'
import { useRoute } from 'vue-router'
import { LayoutDashboard, Boxes, ClipboardList, Wallet, TrendingUp, FileBarChart, Menu, PanelLeftClose, PanelLeftOpen } from '@lucide/vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
    Menu,
    Boxes,
    PanelLeftClose,
    PanelLeftOpen
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()
    const drawerOpen = ref(false)

    // Single source for the sidebar; targets and label keys match the old top nav.
    const navItems = [
      { to: '/', labelKey: 'nav.overview', icon: LayoutDashboard },
      { to: '/inventory', labelKey: 'nav.inventory', icon: Boxes },
      { to: '/orders', labelKey: 'nav.orders', icon: ClipboardList },
      { to: '/spending', labelKey: 'nav.finance', icon: Wallet },
      { to: '/demand', labelKey: 'nav.demandForecast', icon: TrendingUp },
      { to: '/reports', labelKey: 'nav.reports', icon: FileBarChart }
    ]

    // Collapsed (icons-only) state. Two inputs:
    //  - collapsedPref: the user's explicit choice (null = none), persisted in localStorage.
    //  - isNarrow: viewport <= 1024px, the default when there is no explicit choice.
    // An explicit choice always wins; otherwise the viewport decides.
    const STORAGE_KEY = 'sidebar-collapsed'
    const readPref = () => {
      try {
        const v = localStorage.getItem(STORAGE_KEY)
        return v === '1' ? true : v === '0' ? false : null
      } catch (e) {
        return null // storage blocked: behave as "no saved choice"
      }
    }
    const writePref = (value) => {
      try { localStorage.setItem(STORAGE_KEY, value ? '1' : '0') } catch (e) { /* ignore */ }
    }
    const collapsedPref = ref(readPref())
    const mql = typeof window !== 'undefined' && window.matchMedia
      ? window.matchMedia('(max-width: 1024px)')
      : null
    const isNarrow = ref(mql ? mql.matches : false)
    // The listener keeps the default in sync when the window is resized or rotated
    const onViewportChange = (e) => { isNarrow.value = e.matches }
    onMounted(() => {
      if (!mql) return
      isNarrow.value = mql.matches
      mql.addEventListener('change', onViewportChange)
    })
    onBeforeUnmount(() => { if (mql) mql.removeEventListener('change', onViewportChange) })

    const collapsed = computed(() => (collapsedPref.value !== null ? collapsedPref.value : isNarrow.value))
    const toggleLabel = computed(() => t(collapsed.value ? 'nav.expandSidebar' : 'nav.collapseSidebar'))
    const toggleCollapsed = () => {
      collapsedPref.value = !collapsed.value
      writePref(collapsedPref.value)
    }

    // Close the mobile drawer after navigating, and on Escape
    watch(() => route.path, () => { drawerOpen.value = false })
    const onKeydown = (e) => { if (e.key === 'Escape') drawerOpen.value = false }
    onMounted(() => window.addEventListener('keydown', onKeydown))
    onBeforeUnmount(() => window.removeEventListener('keydown', onKeydown))

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
      navItems,
      drawerOpen,
      collapsed,
      toggleLabel,
      toggleCollapsed,
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
*, *::before, *::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  line-height: var(--leading-normal);
  background: var(--color-bg);
  color: var(--color-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

/* ---------- Shell ---------- */
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  transition: grid-template-columns 200ms ease;
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  gap: var(--space-6);
  padding: var(--space-4);
  background: var(--color-sidebar-bg);
  color: var(--color-sidebar-text);
  z-index: var(--z-sidebar);
}

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-2) 0;
}

.brand-mark {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border-radius: var(--radius-sm);
  background: var(--color-accent);
  color: var(--color-sidebar-text-active);
}

.brand-text { display: flex; flex-direction: column; min-width: 0; }
.brand-name {
  font-size: var(--text-lg);
  font-weight: 600;
  color: var(--color-sidebar-text-active);
  line-height: var(--leading-tight);
}
.brand-sub {
  font-size: var(--text-xs);
  color: var(--color-text-subtle);
  line-height: var(--leading-tight);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  flex: 1;
  overflow-y: auto;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  min-height: 40px;
  border-radius: var(--radius-sm);
  color: var(--color-sidebar-text);
  font-size: var(--text-base);
  font-weight: 500;
  text-decoration: none;
  transition: background 150ms ease, color 150ms ease;
}
.nav-label { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.nav-item:hover {
  background: var(--color-sidebar-hover);
  color: var(--color-sidebar-text-active);
}
.nav-item.router-link-exact-active {
  background: var(--color-sidebar-active);
  color: var(--color-sidebar-text-active);
}
.nav-icon { flex-shrink: 0; color: var(--color-text-subtle); transition: color 150ms ease; }
.nav-item:hover .nav-icon { color: var(--color-sidebar-text-active); }
.nav-item.router-link-exact-active .nav-icon { color: #a5b4fc; }
.sidebar .nav-item:focus-visible,
.sidebar button:focus-visible { outline-color: #a5b4fc; }

/* Collapse toggle: styled like a nav item, sits between the nav and the footer */
.sidebar-toggle {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  width: 100%;
  padding: var(--space-2) var(--space-3);
  min-height: 40px;
  background: transparent;
  border: 0;
  border-radius: var(--radius-sm);
  color: var(--color-sidebar-text);
  font-family: inherit;
  font-size: var(--text-base);
  font-weight: 500;
  cursor: pointer;
  transition: background 150ms ease, color 150ms ease;
}
.sidebar-toggle:hover { background: var(--color-sidebar-hover); color: var(--color-sidebar-text-active); }
.sidebar-toggle:hover .nav-icon { color: var(--color-sidebar-text-active); }

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-4);
  border-top: 1px solid var(--color-sidebar-border);
}
.sidebar-footer > * { width: 100%; }
.sidebar-footer button { width: 100%; }

/* Footer dropdowns open upward and anchor to the left edge of the sidebar */
.sidebar .sidebar-footer .dropdown-menu {
  top: auto;
  bottom: calc(100% + var(--space-2));
  left: 0;
  right: auto;
}

.scrim { display: none; }

.app-content { min-width: 0; display: flex; flex-direction: column; }

.topbar {
  position: sticky;
  top: 0;
  z-index: var(--z-topbar);
  display: flex;
  align-items: center;
  gap: var(--space-4);
  min-height: var(--topbar-height);
  padding: var(--space-3) var(--space-8);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}
.topbar > .filters-bar { flex: 1; min-width: 0; }

.menu-button {
  display: none;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  flex-shrink: 0;
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-sm);
  color: var(--color-text);
  cursor: pointer;
}
.menu-button:hover { background: var(--color-surface-muted); }

.page {
  flex: 1;
  width: 100%;
  max-width: var(--page-max-width);
  margin: 0 auto;
  padding: var(--space-8);
}

/* Tablet padding tweaks (independent of sidebar state) */
@media (max-width: 1024px) {
  .topbar { padding: var(--space-3) var(--space-6); }
  .page { padding: var(--space-6); }
}

/* Icons-only rail. Class-driven so the toggle and the viewport default share one path.
   Scoped to >768px: the phone drawer always shows the full sidebar. */
@media (min-width: 769px) {
  .app-shell.is-collapsed { grid-template-columns: var(--sidebar-width-collapsed) minmax(0, 1fr); }
  .is-collapsed .sidebar { align-items: center; padding: var(--space-4) var(--space-2); }
  .is-collapsed .brand-text,
  .is-collapsed .nav-label { display: none; }
  .is-collapsed .sidebar-brand { padding: var(--space-2) 0 0; }
  .is-collapsed .sidebar-nav,
  .is-collapsed .sidebar-toggle,
  .is-collapsed .sidebar-footer { width: 100%; }
  .is-collapsed .nav-item,
  .is-collapsed .sidebar-toggle { justify-content: center; padding: var(--space-2); }
  .is-collapsed .sidebar .sidebar-footer .language-label,
  .is-collapsed .sidebar .sidebar-footer .profile-name,
  .is-collapsed .sidebar .sidebar-footer .chevron { display: none; }
  .is-collapsed .sidebar .sidebar-footer .language-button,
  .is-collapsed .sidebar .sidebar-footer .profile-button { justify-content: center; padding: var(--space-2); }
  /* The rail is only 72px wide, so open dropdowns to its right instead of over the rail.
     .sidebar has no overflow clipping (only .sidebar-nav scrolls) and z-index: var(--z-dropdown)
     on the menu keeps it above the topbar/page; bottom-anchoring keeps it inside the viewport. */
  .is-collapsed .sidebar .sidebar-footer .dropdown-menu {
    left: calc(100% + var(--space-3));
    bottom: 0;
  }
}

/* Phone: off-canvas drawer (breakpoint matches the 768px tablet/phone split) */
@media (max-width: 768px) {
  .app-shell { grid-template-columns: minmax(0, 1fr); }
  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: var(--sidebar-width);
    align-items: stretch;
    padding: var(--space-4);
    transform: translateX(-100%);
    transition: transform 200ms ease;
  }
  .drawer-open .sidebar { transform: translateX(0); box-shadow: var(--shadow-lg); }
  .brand-text, .nav-label { display: flex; }
  .nav-label { display: inline; }
  .sidebar-brand { padding: var(--space-2) var(--space-2) 0; }
  .sidebar-toggle { display: none; }
  .nav-item { justify-content: flex-start; padding: var(--space-2) var(--space-3); }
  .sidebar .sidebar-footer .language-label,
  .sidebar .sidebar-footer .profile-name { display: inline; }
  .sidebar .sidebar-footer .chevron { display: block; }
  .sidebar .sidebar-footer .language-button,
  .sidebar .sidebar-footer .profile-button { justify-content: flex-start; padding: var(--space-2) var(--space-3); }

  .scrim {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.45);
    opacity: 0;
    pointer-events: none;
    transition: opacity 200ms ease;
    z-index: var(--z-scrim);
  }
  .drawer-open .scrim { opacity: 1; pointer-events: auto; }

  .menu-button { display: inline-flex; }
  .topbar { position: static; padding: var(--space-3) var(--space-4); align-items: flex-start; }
  .page { padding: var(--space-4); }
}

@media (prefers-reduced-motion: reduce) {
  .app-shell, .sidebar, .scrim { transition: none; }
}

/* ---------- Shared components ---------- */
.page-header { margin-bottom: var(--space-6); }
.page-header h2 {
  font-size: var(--text-2xl);
  font-weight: 600;
  color: var(--color-text);
  line-height: var(--leading-tight);
  letter-spacing: -0.02em;
  margin-bottom: var(--space-1);
}
.page-header p { color: var(--color-text-muted); font-size: var(--text-base); }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: var(--space-4);
  margin-bottom: var(--space-6);
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-md);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: border-color 150ms ease, box-shadow 150ms ease;
}
.stat-card:hover { border-color: var(--color-border-strong); box-shadow: var(--shadow-md); }

.stat-label {
  color: var(--color-text-muted);
  font-size: var(--text-xs);
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-bottom: var(--space-2);
}
.stat-value {
  font-size: var(--text-2xl);
  font-weight: 600;
  color: var(--color-text);
  letter-spacing: -0.02em;
  line-height: var(--leading-tight);
  font-variant-numeric: tabular-nums;
}
.stat-card.warning .stat-value { color: var(--color-warning); }
.stat-card.success .stat-value { color: var(--color-success); }
.stat-card.danger .stat-value { color: var(--color-danger); }
.stat-card.info .stat-value { color: var(--color-info); }

.card {
  background: var(--color-surface);
  border-radius: var(--radius-md);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-4);
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: var(--space-3) var(--space-4);
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
}
.card-title {
  font-size: var(--text-lg);
  font-weight: 600;
  color: var(--color-text);
  line-height: var(--leading-tight);
}

.table-container { overflow-x: auto; }
table { width: 100%; border-collapse: collapse; }
thead { background: var(--color-surface-muted); }
th {
  text-align: left;
  padding: var(--space-2) var(--space-3);
  font-weight: 600;
  color: var(--color-text-muted);
  font-size: var(--text-xs);
  text-transform: uppercase;
  letter-spacing: 0.04em;
  white-space: nowrap;
}
td {
  padding: var(--space-3);
  border-top: 1px solid var(--color-surface-muted);
  color: var(--color-text);
  font-size: var(--text-base);
  font-variant-numeric: tabular-nums;
}
tbody tr { transition: background-color 150ms ease; }
tbody tr:hover { background: var(--color-bg); }

.badge {
  display: inline-block;
  padding: var(--space-1) var(--space-2);
  border-radius: var(--radius-full);
  font-size: var(--text-xs);
  font-weight: 600;
  letter-spacing: 0.02em;
  text-transform: uppercase;
  line-height: var(--leading-tight);
}
.badge.success, .badge.increasing { background: var(--color-success-soft); color: var(--color-success-text); }
.badge.warning, .badge.medium { background: var(--color-warning-soft); color: var(--color-warning-text); }
.badge.danger, .badge.decreasing, .badge.high { background: var(--color-danger-soft); color: var(--color-danger-text); }
.badge.info, .badge.low { background: var(--color-info-soft); color: var(--color-info-text); }
.badge.stable { background: var(--color-accent-soft); color: var(--color-accent-hover); }

.loading {
  text-align: center;
  padding: var(--space-10);
  color: var(--color-text-muted);
  font-size: var(--text-base);
}
.error {
  background: var(--color-danger-soft);
  border: 1px solid var(--color-danger);
  color: var(--color-danger-text);
  padding: var(--space-4);
  border-radius: var(--radius-sm);
  margin: var(--space-4) 0;
  font-size: var(--text-base);
}
</style>
