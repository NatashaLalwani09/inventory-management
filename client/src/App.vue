<template>
  <div class="app">
    <!-- Sidebar -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <!-- Toggle button -->
      <button class="sidebar-toggle" @click="sidebarCollapsed = !sidebarCollapsed">
        <span v-if="sidebarCollapsed">&#187;</span>
        <span v-else>&#171;</span>
      </button>

      <!-- Logo block -->
      <div class="sidebar-logo" v-show="!sidebarCollapsed">
        <div class="logo-name">{{ t('nav.companyName') }}</div>
        <div class="logo-sub">{{ t('nav.subtitle') }}</div>
      </div>

      <!-- Nav items -->
      <nav class="sidebar-nav">
        <router-link
          to="/"
          class="nav-item"
          :class="{ active: $route.path === '/' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <rect x="3" y="3" width="7" height="7"/>
              <rect x="14" y="3" width="7" height="7"/>
              <rect x="3" y="14" width="7" height="7"/>
              <rect x="14" y="14" width="7" height="7"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">{{ t('nav.overview') }}</span>
        </router-link>

        <router-link
          to="/inventory"
          class="nav-item"
          :class="{ active: $route.path === '/inventory' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <path d="M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z"/>
              <path d="m3.3 7 8.7 5 8.7-5"/>
              <path d="M12 22V12"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">{{ t('nav.inventory') }}</span>
        </router-link>

        <router-link
          to="/orders"
          class="nav-item"
          :class="{ active: $route.path === '/orders' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <path d="M9 5H7a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h10a2 2 0 0 0 2-2V7a2 2 0 0 0-2-2h-2"/>
              <rect x="9" y="3" width="6" height="4" rx="2"/>
              <path d="M9 12h6"/>
              <path d="M9 16h4"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">{{ t('nav.orders') }}</span>
        </router-link>

        <router-link
          to="/spending"
          class="nav-item"
          :class="{ active: $route.path === '/spending' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <line x1="12" y1="1" x2="12" y2="23"/>
              <path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">{{ t('nav.finance') }}</span>
        </router-link>

        <router-link
          to="/demand"
          class="nav-item"
          :class="{ active: $route.path === '/demand' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <polyline points="22 7 13.5 15.5 8.5 10.5 2 17"/>
              <polyline points="16 7 22 7 22 13"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">{{ t('nav.demandForecast') }}</span>
        </router-link>

        <router-link
          to="/reports"
          class="nav-item"
          :class="{ active: $route.path === '/reports' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <line x1="18" y1="20" x2="18" y2="10"/>
              <line x1="12" y1="20" x2="12" y2="4"/>
              <line x1="6" y1="20" x2="6" y2="14"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">Reports</span>
        </router-link>

        <router-link
          to="/restocking"
          class="nav-item"
          :class="{ active: $route.path === '/restocking' }"
        >
          <span class="nav-icon">
            <svg viewBox="0 0 24 24">
              <polyline points="1 4 1 10 7 10"/>
              <path d="M3.51 15a9 9 0 1 0 .49-3.5"/>
            </svg>
          </span>
          <span class="nav-label" v-show="!sidebarCollapsed">Restocking</span>
        </router-link>
      </nav>

      <!-- Bottom section -->
      <div class="sidebar-bottom">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <!-- Content area -->
    <div
      class="content-area"
      :style="{ marginLeft: sidebarCollapsed ? '64px' : '220px' }"
    >
      <div class="filter-bar-wrapper">
        <FilterBar />
      </div>
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
import { ref, onMounted, computed } from 'vue'
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
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const sidebarCollapsed = ref(false)

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

    onMounted(() => {
      loadTasks()
      sidebarCollapsed.value = window.innerWidth < 768
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      sidebarCollapsed
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ── Layout ─────────────────────────────────────────────── */

.app {
  display: flex;
  min-height: 100vh;
}

/* Sidebar */
.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  width: 220px;
  background: #0f172a;
  border-right: 1px solid #1e293b;
  display: flex;
  flex-direction: column;
  z-index: 100;
  transition: width 0.25s ease;
  overflow: hidden;
}

.sidebar.collapsed {
  width: 64px;
}

/* Toggle button */
.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding: 16px 12px;
  color: #475569;
  cursor: pointer;
  border-bottom: 1px solid #1e293b;
  background: none;
  border-top: none;
  border-left: none;
  border-right: none;
  width: 100%;
  font-size: 0.75rem;
}

.sidebar-toggle:hover {
  color: #94a3b8;
}

.sidebar.collapsed .sidebar-toggle {
  justify-content: center;
}

/* Logo block */
.sidebar-logo {
  padding: 16px;
  border-bottom: 1px solid #1e293b;
}

.sidebar-logo .logo-name {
  font-size: 0.938rem;
  font-weight: 700;
  color: #f8fafc;
  white-space: nowrap;
}

.sidebar-logo .logo-sub {
  font-size: 0.7rem;
  color: #475569;
  white-space: nowrap;
  margin-top: 2px;
}

/* Nav items */
.sidebar-nav {
  flex: 1;
  overflow-y: auto;
  padding: 8px 0;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  color: #64748b;
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 6px;
  margin: 2px 8px;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
}

.nav-item:hover {
  background: #1e293b;
  color: #cbd5e1;
}

.nav-item.active {
  background: #1e293b;
  color: #f8fafc;
  border-left: 3px solid #6366f1;
}

.nav-icon {
  display: flex;
  align-items: center;
  flex-shrink: 0;
}

.nav-icon svg {
  width: 18px;
  height: 18px;
  stroke: currentColor;
  fill: none;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
  flex-shrink: 0;
}

.nav-label {
  transition: opacity 0.2s;
}

.sidebar.collapsed .nav-item {
  justify-content: center;
  padding: 10px;
  margin: 2px 6px;
}

.sidebar.collapsed .nav-item.active {
  border-left: none;
  border-radius: 8px;
  background: #6366f1;
  color: #fff;
}

/* Bottom section */
.sidebar-bottom {
  border-top: 1px solid #1e293b;
  padding: 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

/* Content area */
.content-area {
  transition: margin-left 0.25s ease;
  min-height: 100vh;
  background: #f8fafc;
  display: flex;
  flex-direction: column;
  flex: 1;
}

.filter-bar-wrapper {
  position: sticky;
  top: 0;
  z-index: 50;
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
}

.main-content {
  flex: 1;
  padding: 24px 32px;
  max-width: 1600px;
  width: 100%;
}

/* ── Typography & component styles ──────────────────────── */

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
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
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
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
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
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
