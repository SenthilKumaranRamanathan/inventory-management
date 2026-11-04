<template>
  <div class="app-shell" :class="{ 'is-collapsed': effectiveCollapsed }">
    <aside
      id="primary-sidebar"
      class="sidebar"
      :class="{ 'is-open': isSidebarOpen, 'is-collapsed': effectiveCollapsed }"
      aria-label="Primary"
    >
      <div class="sidebar-brand">
        <span class="sidebar-brand-mark" aria-hidden="true">{{ brandMark }}</span>
        <h1 class="sidebar-brand-name">{{ t('nav.companyName') }}</h1>
        <span class="sidebar-brand-sub">{{ t('nav.subtitle') }}</span>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          class="sidebar-link"
          active-class=""
          exact-active-class="router-link-exact-active"
          aria-current-value="page"
          :title="effectiveCollapsed ? item.label : null"
          :data-tooltip="item.label"
        >
          <svg
            class="sidebar-icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="1.75"
            stroke-linecap="round"
            stroke-linejoin="round"
            aria-hidden="true"
          >
            <path v-for="d in item.icon" :key="d" :d="d" />
          </svg>
          <span class="sidebar-link-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <button
        type="button"
        class="sidebar-collapse-btn"
        :aria-label="effectiveCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        :aria-expanded="effectiveCollapsed ? 'false' : 'true'"
        aria-controls="primary-sidebar"
        @click="toggleCollapsed"
      >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <path d="M15 6l-6 6 6 6" />
        </svg>
      </button>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div v-if="isSidebarOpen" class="sidebar-backdrop" @click="closeSidebar"></div>

    <div class="main-column">
      <div class="mobile-bar">
        <button
          type="button"
          class="mobile-menu-btn"
          aria-label="Menu"
          aria-controls="primary-sidebar"
          :aria-expanded="isSidebarOpen ? 'true' : 'false'"
          @click="toggleSidebar"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" aria-hidden="true">
            <path d="M4 6h16M4 12h16M4 18h16" />
          </svg>
        </button>
        <span class="mobile-bar-title">{{ t('nav.companyName') }}</span>
      </div>

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
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue'
import { useRoute } from 'vue-router'
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
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const isSidebarOpen = ref(false)

    // Icons are 24x24 stroke paths rendered in the template
    const navItems = computed(() => [
      { path: '/', label: t('nav.overview'), icon: ['M3 3h7v7H3z', 'M14 3h7v7h-7z', 'M14 14h7v7h-7z', 'M3 14h7v7H3z'] },
      { path: '/inventory', label: t('nav.inventory'), icon: ['M21 8l-9-5-9 5v8l9 5 9-5z', 'M3 8l9 5 9-5', 'M12 13v8'] },
      { path: '/orders', label: t('nav.orders'), icon: ['M9 4h6v3H9z', 'M7 5H5v16h14V5h-2', 'M8 12h8', 'M8 16h6'] },
      { path: '/spending', label: t('nav.finance'), icon: ['M12 2v20', 'M17 6H9.5a3.5 3.5 0 000 7h5a3.5 3.5 0 010 7H6'] },
      { path: '/demand', label: t('nav.demandForecast'), icon: ['M3 17l6-6 4 4 8-8', 'M15 7h6v6'] },
      { path: '/reports', label: 'Reports', icon: ['M14 3H6v18h12V7z', 'M14 3v4h4', 'M9 13h6', 'M9 17h6'] }
    ])

    const closeSidebar = () => {
      isSidebarOpen.value = false
    }
    const toggleSidebar = () => {
      isSidebarOpen.value = !isSidebarOpen.value
    }
    const onKeydown = (e) => {
      if (e.key === 'Escape') closeSidebar()
    }
    watch(() => route.fullPath, closeSidebar)
    onMounted(() => window.addEventListener('keydown', onKeydown))
    onBeforeUnmount(() => window.removeEventListener('keydown', onKeydown))

    // Collapse rule: if the user has toggled manually (stored true/false), that
    // choice always wins. Otherwise the sidebar auto-collapses only on tablet
    // widths (768-1023px) and is expanded on desktop. Mobile (<768px) uses the
    // drawer and ignores collapse (enforced in CSS).
    const COLLAPSE_KEY = 'sidebar-collapsed'
    const readStoredCollapsed = () => {
      try {
        const v = window.localStorage.getItem(COLLAPSE_KEY)
        if (v === 'true') return true
        if (v === 'false') return false
      } catch (e) {
        // storage blocked or unavailable: fall back to auto behaviour
      }
      return null
    }
    const isCollapsed = ref(readStoredCollapsed()) // null = no manual choice
    const tabletQuery = typeof window !== 'undefined' && window.matchMedia
      ? window.matchMedia('(min-width: 768px) and (max-width: 1023px)')
      : null
    const isTablet = ref(tabletQuery ? tabletQuery.matches : false)
    const onTabletChange = (e) => {
      isTablet.value = e.matches
    }
    const effectiveCollapsed = computed(() =>
      isCollapsed.value !== null ? isCollapsed.value : isTablet.value
    )
    const toggleCollapsed = () => {
      isCollapsed.value = !effectiveCollapsed.value
      try {
        window.localStorage.setItem(COLLAPSE_KEY, String(isCollapsed.value))
      } catch (e) {
        // ignore: choice still applies for this session
      }
    }
    const brandMark = computed(() => (t('nav.companyName') || '').trim().charAt(0).toUpperCase())
    onMounted(() => {
      if (!tabletQuery) return
      isTablet.value = tabletQuery.matches
      tabletQuery.addEventListener('change', onTabletChange)
    })
    onBeforeUnmount(() => {
      if (tabletQuery) tabletQuery.removeEventListener('change', onTabletChange)
    })

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
      isSidebarOpen,
      closeSidebar,
      toggleSidebar,
      isCollapsed,
      effectiveCollapsed,
      toggleCollapsed,
      brandMark,
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
:root {
  /* Color */
  --color-bg: #f8fafc;
  --color-bg-muted: #f1f5f9;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-border-strong: #cbd5e1;
  --color-text: #0f172a;
  --color-text-secondary: #334155;
  --color-text-muted: #64748b;
  --color-accent: #2563eb;
  --color-accent-strong: #1e40af;
  --color-accent-soft: #eff6ff;
  --color-success: #059669;
  --color-success-soft: #d1fae5;
  --color-success-strong: #065f46;
  --color-warning: #d97706;
  --color-warning-soft: #fef3c7;
  --color-warning-strong: #92400e;
  --color-danger: #dc2626;
  --color-danger-soft: #fef2f2;
  --color-danger-badge: #fee2e2;
  --color-danger-border: #fecaca;
  --color-danger-strong: #991b1b;
  --color-info-soft: #dbeafe;
  --color-info-strong: #1e40af;
  --color-backdrop: rgba(15, 23, 42, 0.4);

  /* Spacing: 4px base */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-5: 1.25rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* Shape and depth */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-pill: 999px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 10px 25px rgba(15, 23, 42, 0.12);
  --focus-ring: 0 0 0 3px rgba(37, 99, 235, 0.25);

  /* Layout */
  --sidebar-width: 15rem;
  --sidebar-width-collapsed: 4rem;
  --content-max-width: 1400px;
  --topbar-height: 3.5rem;
  --topbar-offset: 0px;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: var(--color-bg);
  color: var(--color-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

a:focus-visible,
button:focus-visible,
select:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

/* Shell */
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  background: var(--color-bg);
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  z-index: 200;
  display: flex;
  flex-direction: column;
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  padding: var(--space-4);
}

.sidebar-brand {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  padding: var(--space-2) var(--space-3);
}

.sidebar-brand-name {
  font-size: 1.25rem;
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.sidebar-brand-sub {
  font-size: 0.813rem;
  color: var(--color-text-muted);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  margin-top: var(--space-6);
  flex: 1;
  overflow-y: auto;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--color-text-muted);
  text-decoration: none;
  font-weight: 500;
  font-size: 0.938rem;
}

.sidebar-link:hover {
  background: var(--color-bg);
  color: var(--color-text);
}

.sidebar-link.router-link-exact-active {
  background: var(--color-accent-soft);
  color: var(--color-accent);
  font-weight: 600;
}

.sidebar-icon {
  width: 1.25rem;
  height: 1.25rem;
  flex-shrink: 0;
}

/* Footer must not clip dropdowns: no overflow here */
.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-4);
  border-top: 1px solid var(--color-border);
  position: relative;
}

.main-column {
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.main-content {
  flex: 1;
  width: 100%;
  max-width: var(--content-max-width);
  margin: 0 auto;
  padding: var(--space-6) var(--space-8);
}

.mobile-bar,
.sidebar-backdrop {
  display: none;
}

/* Page building blocks */
.page-header {
  margin-bottom: var(--space-6);
}

.page-header h2 {
  font-size: 1.625rem;
  font-weight: 700;
  color: var(--color-text);
  margin-bottom: var(--space-1);
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--color-text-muted);
  font-size: 0.938rem;
}

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
}

.stat-card:hover {
  border-color: var(--color-border-strong);
  box-shadow: var(--shadow-md);
}

.stat-label {
  color: var(--color-text-muted);
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: var(--color-warning);
}

.stat-card.success .stat-value {
  color: var(--color-success);
}

.stat-card.danger .stat-value {
  color: var(--color-danger);
}

.stat-card.info .stat-value {
  color: var(--color-accent);
}

.card {
  background: var(--color-surface);
  border-radius: var(--radius-md);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-5);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: 1.063rem;
  font-weight: 700;
  color: var(--color-text);
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
  background: var(--color-bg);
  border-top: 1px solid var(--color-border);
  border-bottom: 1px solid var(--color-border);
}

th {
  text-align: left;
  padding: var(--space-2) var(--space-3);
  font-weight: 600;
  color: var(--color-text-muted);
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  white-space: nowrap;
}

td {
  padding: var(--space-2) var(--space-3);
  border-top: 1px solid var(--color-bg-muted);
  color: var(--color-text-secondary);
  font-size: 0.875rem;
}

tbody tr:hover {
  background: var(--color-bg);
}

.badge {
  display: inline-block;
  padding: var(--space-1) var(--space-2);
  border-radius: var(--radius-pill);
  font-size: 0.75rem;
  font-weight: 600;
  line-height: 1.25;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  background: var(--color-bg-muted);
  color: var(--color-text-secondary);
}

.badge.success,
.badge.increasing {
  background: var(--color-success-soft);
  color: var(--color-success-strong);
}

.badge.warning,
.badge.medium {
  background: var(--color-warning-soft);
  color: var(--color-warning-strong);
}

.badge.danger,
.badge.decreasing,
.badge.high {
  background: var(--color-danger-badge);
  color: var(--color-danger-strong);
}

.badge.info,
.badge.low {
  background: var(--color-info-soft);
  color: var(--color-info-strong);
}

.badge.stable {
  background: var(--color-bg-muted);
  color: var(--color-text-secondary);
}

.loading {
  text-align: center;
  padding: var(--space-8);
  color: var(--color-text-muted);
  font-size: 0.938rem;
}

.error {
  background: var(--color-danger-soft);
  border: 1px solid var(--color-danger-border);
  color: var(--color-danger-strong);
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: 0.938rem;
}

.sidebar-brand-mark,
.sidebar-collapse-btn {
  display: none;
}

.sidebar-link {
  position: relative;
}

/* Collapse toggle and collapsed (icons-only) mode: tablet and desktop only */
@media (min-width: 768px) {
  .sidebar-collapse-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    align-self: flex-end;
    width: 2.5rem;
    height: 2.5rem;
    margin-bottom: var(--space-2);
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-sm);
    color: var(--color-text-muted);
    cursor: pointer;
  }

  .sidebar-collapse-btn:hover {
    background: var(--color-bg);
    color: var(--color-text);
  }

  .sidebar-collapse-btn svg {
    width: 1.25rem;
    height: 1.25rem;
  }

  .sidebar.is-collapsed {
    padding: var(--space-4) var(--space-3);
  }

  .app-shell.is-collapsed {
    grid-template-columns: var(--sidebar-width-collapsed) minmax(0, 1fr);
  }

  .sidebar.is-collapsed .sidebar-collapse-btn {
    align-self: center;
  }

  .sidebar.is-collapsed .sidebar-collapse-btn svg {
    transform: rotate(180deg);
  }

  .sidebar.is-collapsed .sidebar-brand {
    align-items: center;
    padding: 0;
  }

  .sidebar.is-collapsed .sidebar-brand-mark {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 2.5rem;
    height: 2.5rem;
    border-radius: var(--radius-sm);
    background: var(--color-accent-soft);
    color: var(--color-accent);
    font-weight: 700;
    font-size: 1.125rem;
  }

  /* Visually hidden, still read by screen readers */
  .sidebar.is-collapsed .sidebar-brand-name,
  .sidebar.is-collapsed .sidebar-brand-sub,
  .sidebar.is-collapsed .sidebar-link-label {
    position: absolute;
    width: 1px;
    height: 1px;
    margin: -1px;
    padding: 0;
    border: 0;
    overflow: hidden;
    clip: rect(0 0 0 0);
    white-space: nowrap;
  }

  /* Tooltips need to escape the nav, so it cannot scroll while collapsed */
  .sidebar.is-collapsed .sidebar-nav {
    overflow: visible;
    align-items: center;
  }

  .sidebar.is-collapsed .sidebar-link {
    justify-content: center;
    width: 2.5rem;
    height: 2.5rem;
    padding: 0;
  }

  .sidebar.is-collapsed .sidebar-link::after {
    content: attr(data-tooltip);
    position: absolute;
    left: calc(100% + var(--space-2));
    top: 50%;
    transform: translateY(-50%);
    padding: var(--space-1) var(--space-2);
    border-radius: var(--radius-sm);
    background: var(--color-text);
    color: var(--color-surface);
    font-size: 0.813rem;
    font-weight: 500;
    white-space: nowrap;
    box-shadow: var(--shadow-md);
    z-index: 1000;
    pointer-events: none;
    opacity: 0;
    visibility: hidden;
  }

  .sidebar.is-collapsed .sidebar-link:hover::after,
  .sidebar.is-collapsed .sidebar-link:focus-visible::after {
    opacity: 1;
    visibility: visible;
  }
}

@media (min-width: 768px) and (prefers-reduced-motion: no-preference) {
  .app-shell {
    transition: grid-template-columns 0.2s ease;
  }
}

/* Mobile: off-canvas drawer */
@media (max-width: 767px) {
  :root {
    --topbar-offset: var(--topbar-height);
  }

  .app-shell {
    grid-template-columns: minmax(0, 1fr);
  }

  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    bottom: 0;
    height: 100%;
    width: var(--sidebar-width);
    max-width: 85vw;
    z-index: 300;
    transform: translateX(-100%);
    box-shadow: var(--shadow-lg);
    visibility: hidden;
  }

  .sidebar.is-open {
    transform: translateX(0);
    visibility: visible;
  }

  .sidebar-backdrop {
    display: block;
    position: fixed;
    inset: 0;
    z-index: 250;
    background: var(--color-backdrop);
  }

  .mobile-bar {
    display: flex;
    align-items: center;
    gap: var(--space-3);
    height: var(--topbar-height);
    padding: 0 var(--space-4);
    position: sticky;
    top: 0;
    z-index: 100;
    background: var(--color-surface);
    border-bottom: 1px solid var(--color-border);
  }

  .mobile-menu-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 2.25rem;
    height: 2.25rem;
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-sm);
    color: var(--color-text);
    cursor: pointer;
  }

  .mobile-menu-btn svg {
    width: 1.25rem;
    height: 1.25rem;
  }

  .mobile-bar-title {
    font-weight: 700;
    letter-spacing: -0.025em;
    color: var(--color-text);
  }

  .main-content {
    padding: var(--space-4);
  }
}

@media (max-width: 767px) and (prefers-reduced-motion: no-preference) {
  .sidebar {
    transition: transform 0.2s ease, visibility 0.2s;
  }
}

@media (prefers-reduced-motion: no-preference) {
  .sidebar-link,
  .stat-card,
  tbody tr {
    transition: background-color 0.15s ease, color 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;
  }
}
</style>
