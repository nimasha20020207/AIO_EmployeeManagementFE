<script setup lang="ts">
import { computed, reactive, ref, watch } from 'vue'
import {
  ArrowDownUp,
  ArrowLeft,
  ArrowRight,
  ArrowUpRight,
  Bell,
  BriefcaseBusiness,
  CalendarDays,
  Check,
  ChevronDown,
  Download,
  LayoutDashboard,
  Mail,
  MoreHorizontal,
  Plus,
  Search,
  Settings,
  SlidersHorizontal,
  Sparkles,
  UsersRound,
  X,
} from '@lucide/vue'

type EmployeeStatus = 'Active' | 'On leave'
type Employee = {
  id: number
  name: string
  role: string
  department: string
  email: string
  status: EmployeeStatus
  joined: string
  color: string
}

const employees = ref<Employee[]>([
  { id: 1, name: 'Olivia Rhye', role: 'Product Designer', department: 'Design', email: 'olivia.rhye@acme.com', status: 'Active', joined: 'Jan 12, 2024', color: 'lavender' },
  { id: 2, name: 'Phoenix Baker', role: 'Engineering Manager', department: 'Engineering', email: 'phoenix.baker@acme.com', status: 'Active', joined: 'Jan 18, 2024', color: 'peach' },
  { id: 3, name: 'Lana Steiner', role: 'Product Manager', department: 'Product', email: 'lana.steiner@acme.com', status: 'On leave', joined: 'Feb 02, 2024', color: 'sky' },
  { id: 4, name: 'Demi Wilkinson', role: 'Frontend Developer', department: 'Engineering', email: 'demi.wilkinson@acme.com', status: 'Active', joined: 'Feb 14, 2024', color: 'rose' },
  { id: 5, name: 'Candice Wu', role: 'UX Researcher', department: 'Design', email: 'candice.wu@acme.com', status: 'Active', joined: 'Mar 01, 2024', color: 'mint' },
  { id: 6, name: 'Natali Craig', role: 'HR Manager', department: 'People', email: 'natali.craig@acme.com', status: 'Active', joined: 'Mar 08, 2024', color: 'gold' },
  { id: 7, name: 'Drew Cano', role: 'Backend Developer', department: 'Engineering', email: 'drew.cano@acme.com', status: 'Active', joined: 'Mar 11, 2024', color: 'blue' },
  { id: 8, name: 'Orlando Diggs', role: 'Marketing Lead', department: 'Marketing', email: 'orlando.diggs@acme.com', status: 'On leave', joined: 'Mar 17, 2024', color: 'peach' },
])

const navItems = [
  { label: 'Overview', icon: LayoutDashboard },
  { label: 'Employees', icon: UsersRound },
  { label: 'Attendance', icon: CalendarDays },
  { label: 'Departments', icon: BriefcaseBusiness },
]
const departments = ['Engineering', 'Design', 'Product', 'People', 'Marketing', 'Finance']
const activeView = ref('Overview')
const searchTerm = ref('')
const departmentFilter = ref('All departments')
const statusFilter = ref('All status')
const page = ref(1)
const pageSize = 5
const sortAscending = ref(true)
const showEmployeeModal = ref(false)
const editingEmployeeId = ref<number | null>(null)
const showNotifications = ref(false)
const notice = ref('')
const employeeForm = reactive({
  name: '',
  role: '',
  department: 'Engineering',
  email: '',
  status: 'Active' as EmployeeStatus,
})

const activeCount = computed(() => employees.value.filter((employee) => employee.status === 'Active').length)
const onLeaveCount = computed(() => employees.value.filter((employee) => employee.status === 'On leave').length)
const departmentCount = computed(() => new Set(employees.value.map((employee) => employee.department)).size)
const filteredEmployees = computed(() => {
  const query = searchTerm.value.trim().toLowerCase()
  const result = employees.value.filter((employee) => {
    const matchesQuery = !query || [employee.name, employee.role, employee.department, employee.email]
      .some((value) => value.toLowerCase().includes(query))
    const matchesDepartment = departmentFilter.value === 'All departments' || employee.department === departmentFilter.value
    const matchesStatus = statusFilter.value === 'All status' || employee.status === statusFilter.value
    return matchesQuery && matchesDepartment && matchesStatus
  })

  return result.sort((first, second) => sortAscending.value
    ? first.name.localeCompare(second.name)
    : second.name.localeCompare(first.name))
})
const pageCount = computed(() => Math.max(1, Math.ceil(filteredEmployees.value.length / pageSize)))
const paginatedEmployees = computed(() => {
  const start = (page.value - 1) * pageSize
  return filteredEmployees.value.slice(start, start + pageSize)
})
const pageLabel = computed(() => {
  if (!filteredEmployees.value.length) return '0 results'
  const start = (page.value - 1) * pageSize + 1
  const end = Math.min(page.value * pageSize, filteredEmployees.value.length)
  return `${start}–${end} of ${filteredEmployees.value.length}`
})
const departmentStats = computed(() => {
  const totals = departments.map((name) => ({
    name,
    count: employees.value.filter((employee) => employee.department === name).length,
  })).filter((department) => department.count > 0)
  const maximum = Math.max(1, ...totals.map((department) => department.count))
  return totals.map((department) => ({ ...department, width: `${Math.max(14, (department.count / maximum) * 100)}%` }))
})

watch(searchTerm, () => {
  page.value = 1
})

function notify(message: string) {
  notice.value = message
  window.setTimeout(() => {
    if (notice.value === message) notice.value = ''
  }, 3000)
}

function changeView(view: string) {
  activeView.value = view
  page.value = 1
}

function resetEmployeeForm() {
  employeeForm.name = ''
  employeeForm.role = ''
  employeeForm.department = 'Engineering'
  employeeForm.email = ''
  employeeForm.status = 'Active'
  editingEmployeeId.value = null
}

function openAddEmployee() {
  resetEmployeeForm()
  showEmployeeModal.value = true
}

function openEditEmployee(employee: Employee) {
  editingEmployeeId.value = employee.id
  employeeForm.name = employee.name
  employeeForm.role = employee.role
  employeeForm.department = employee.department
  employeeForm.email = employee.email
  employeeForm.status = employee.status
  showEmployeeModal.value = true
}

function saveEmployee() {
  const name = employeeForm.name.trim()
  const email = employeeForm.email.trim()
  if (!name || !employeeForm.role.trim() || !email) return

  if (editingEmployeeId.value !== null) {
    const employee = employees.value.find((item) => item.id === editingEmployeeId.value)
    if (employee) {
      Object.assign(employee, {
        name,
        role: employeeForm.role.trim(),
        department: employeeForm.department,
        email,
        status: employeeForm.status,
      })
      notify('Employee details updated')
    }
  } else {
    employees.value.unshift({
      id: Date.now(),
      name,
      role: employeeForm.role.trim(),
      department: employeeForm.department,
      email,
      status: employeeForm.status,
      joined: new Intl.DateTimeFormat('en-US', { month: 'short', day: '2-digit', year: 'numeric' }).format(new Date()),
      color: ['lavender', 'peach', 'sky', 'rose', 'mint', 'gold'][employees.value.length % 6],
    })
    page.value = 1
    notify('Employee added to your team')
  }
  showEmployeeModal.value = false
}

function removeEmployee(employee: Employee) {
  if (!window.confirm(`Remove ${employee.name} from the team?`)) return
  employees.value = employees.value.filter((item) => item.id !== employee.id)
  page.value = Math.min(page.value, pageCount.value)
  notify('Employee removed')
}

function exportEmployees() {
  const headers = ['Name', 'Role', 'Department', 'Email', 'Status', 'Date joined']
  const rows = filteredEmployees.value.map((employee) => [
    employee.name, employee.role, employee.department, employee.email, employee.status, employee.joined,
  ])
  const csv = [headers, ...rows]
    .map((row) => row.map((value) => `"${value.replace(/"/g, '""')}"`).join(','))
    .join('\r\n')
  const url = URL.createObjectURL(new Blob([csv], { type: 'text/csv;charset=utf-8' }))
  const link = document.createElement('a')
  link.href = url
  link.download = 'employees.csv'
  link.click()
  URL.revokeObjectURL(url)
  notify('Employee list exported')
}

function setDepartment(event: Event) {
  departmentFilter.value = (event.target as HTMLSelectElement).value
  page.value = 1
}

function setStatus(event: Event) {
  statusFilter.value = (event.target as HTMLSelectElement).value
  page.value = 1
}
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar">
      <a class="brand" href="#" aria-label="AIO home" @click.prevent="changeView('Overview')">
        <span class="brand-mark"><span></span><span></span><span></span><span></span></span>
        <span class="brand-name">aio<span>.</span></span>
      </a>

      <div class="workspace-switcher">
        <div class="workspace-avatar">A</div>
        <div class="workspace-copy">
          <span class="workspace-label">Workspace</span>
          <strong>AIO Studio</strong>
        </div>
        <ChevronDown :size="15" />
      </div>

      <div class="nav-label">WORKSPACE</div>
      <nav class="main-nav" aria-label="Main navigation">
        <button
          v-for="item in navItems"
          :key="item.label"
          class="nav-link"
          :class="{ 'is-active': activeView === item.label }"
          @click="item.label === 'Overview' || item.label === 'Employees' ? changeView(item.label) : notify(`${item.label} view is coming soon`)"
        >
          <component :is="item.icon" :size="18" :stroke-width="1.8" />
          <span>{{ item.label }}</span>
          <span v-if="item.label === 'Employees'" class="nav-count">{{ employees.length }}</span>
        </button>
      </nav>

      <div class="sidebar-bottom">
        <div class="help-card">
          <div class="help-icon"><Sparkles :size="17" /></div>
          <strong>Need a hand?</strong>
          <p>Our team is here to help you make the most of AIO.</p>
          <button @click="notify('Our support team will be in touch soon')">Visit help center <ArrowUpRight :size="14" /></button>
        </div>
        <button class="nav-link settings-link" @click="notify('Settings are coming soon')">
          <Settings :size="18" :stroke-width="1.8" />
          <span>Settings</span>
        </button>
        <button class="profile-card" @click="notify('Signed in as Alex Morgan')">
          <div class="avatar avatar-profile">AM</div>
          <span class="profile-copy"><strong>Alex Morgan</strong><small>HR Administrator</small></span>
          <MoreHorizontal :size="19" />
        </button>
      </div>
    </aside>

    <main class="main-content">
      <header class="topbar">
        <div class="breadcrumbs"><span>Workspace</span><span class="crumb-slash">/</span><strong>{{ activeView }}</strong></div>
        <div class="topbar-actions">
          <label class="global-search">
            <Search :size="16" />
            <input v-model="searchTerm" type="search" placeholder="Search anything..." aria-label="Search employees" />
            <kbd>⌘ K</kbd>
          </label>
          <div class="notification-wrap">
            <button class="icon-button notification-button" aria-label="Notifications" @click="showNotifications = !showNotifications">
              <Bell :size="18" />
              <span class="notification-dot"></span>
            </button>
            <div v-if="showNotifications" class="notification-popover">
              <strong>You’re all caught up</strong>
              <p>New team updates will show up here.</p>
            </div>
          </div>
          <div class="topbar-avatar">AM</div>
        </div>
      </header>

      <div class="page-content">
        <section class="page-heading">
          <div>
            <div class="eyebrow"><span class="eyebrow-dot"></span> PEOPLE OPERATIONS</div>
            <h1>{{ activeView === 'Employees' ? 'Employee directory' : 'People overview' }}</h1>
            <p>{{ activeView === 'Employees' ? 'Manage your team and keep everyone moving forward.' : 'Here’s what’s happening with your team today.' }}</p>
          </div>
          <div class="heading-actions">
            <button class="button button-secondary" @click="exportEmployees"><Download :size="16" /> Export</button>
            <button class="button button-primary" @click="openAddEmployee"><Plus :size="17" /> Add employee</button>
          </div>
        </section>

        <template v-if="activeView === 'Overview'">
          <section class="metrics-grid" aria-label="Team metrics">
            <article class="metric-card">
              <div class="metric-top"><span>Total employees</span><span class="metric-icon icon-purple"><UsersRound :size="17" /></span></div>
              <div class="metric-value">{{ employees.length }}<span class="metric-change"><ArrowUpRight :size="14" /> 12%</span></div>
              <div class="metric-foot">Compared to last month</div>
              <div class="metric-spark spark-purple"><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i></div>
            </article>
            <article class="metric-card">
              <div class="metric-top"><span>Active members</span><span class="metric-icon icon-green"><Check :size="17" /></span></div>
              <div class="metric-value">{{ activeCount }}<span class="metric-change"><ArrowUpRight :size="14" /> 8.2%</span></div>
              <div class="metric-foot">Compared to last month</div>
              <div class="metric-spark spark-green"><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i></div>
            </article>
            <article class="metric-card">
              <div class="metric-top"><span>On leave</span><span class="metric-icon icon-amber"><CalendarDays :size="17" /></span></div>
              <div class="metric-value">{{ onLeaveCount }}<span class="metric-neutral">this month</span></div>
              <div class="metric-foot">Back to work soon</div>
              <div class="metric-spark spark-amber"><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i></div>
            </article>
            <article class="metric-card">
              <div class="metric-top"><span>Departments</span><span class="metric-icon icon-blue"><BriefcaseBusiness :size="17" /></span></div>
              <div class="metric-value">{{ departmentCount }}<span class="metric-neutral">teams</span></div>
              <div class="metric-foot">Across the organization</div>
              <div class="metric-spark spark-blue"><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i></div>
            </article>
          </section>

          <section class="insights-grid">
            <article class="panel department-panel">
              <div class="panel-heading">
                <div><h2>Team distribution</h2><p>Employees by department</p></div>
                <button class="more-button" aria-label="More department options" @click="notify('Department report is up to date')"><MoreHorizontal :size="20" /></button>
              </div>
              <div class="department-list">
                <div v-for="(department, index) in departmentStats" :key="department.name" class="department-row">
                  <div class="department-row-top"><span><i class="legend-dot" :class="`legend-${index % 5}`"></i>{{ department.name }}</span><strong>{{ department.count }}</strong></div>
                  <div class="progress-track"><span :class="`progress-${index % 5}`" :style="{ width: department.width }"></span></div>
                </div>
              </div>
              <button class="text-link" @click="changeView('Employees')">View all employees <ArrowRight :size="14" /></button>
            </article>

            <article class="panel growth-panel">
              <div class="panel-heading">
                <div><h2>Team growth</h2><p>New team members over time</p></div>
                <button class="period-select" @click="notify('Showing the last 6 months')">Last 6 months <ChevronDown :size="14" /></button>
              </div>
              <div class="chart-summary"><strong>+{{ employees.length }}</strong><span class="chart-positive"><ArrowUpRight :size="14" /> 16.8%</span><span class="chart-summary-note">vs. previous period</span></div>
              <div class="chart-wrap">
                <div class="chart-y-labels"><span>12</span><span>9</span><span>6</span><span>3</span><span>0</span></div>
                <div class="chart-main">
                  <div class="chart-gridlines"><i></i><i></i><i></i><i></i><i></i></div>
                  <svg class="growth-chart" viewBox="0 0 540 170" preserveAspectRatio="none" role="img" aria-label="Employee growth trend chart">
                    <defs><linearGradient id="chartFill" x1="0" x2="0" y1="0" y2="1"><stop offset="0%" stop-color="#7664e8" stop-opacity=".19" /><stop offset="100%" stop-color="#7664e8" stop-opacity="0" /></linearGradient></defs>
                    <path d="M0 135 C35 129 46 139 78 119 S125 111 155 117 S202 91 233 101 S280 94 310 75 S357 84 389 56 S436 69 464 46 S513 39 540 19 V170 H0 Z" fill="url(#chartFill)" />
                    <path d="M0 135 C35 129 46 139 78 119 S125 111 155 117 S202 91 233 101 S280 94 310 75 S357 84 389 56 S436 69 464 46 S513 39 540 19" fill="none" stroke="#7968e9" stroke-width="3" vector-effect="non-scaling-stroke" />
                    <circle cx="540" cy="19" r="5" fill="#fff" stroke="#7968e9" stroke-width="3" vector-effect="non-scaling-stroke" />
                  </svg>
                  <div class="chart-x-labels"><span>Oct</span><span>Nov</span><span>Dec</span><span>Jan</span><span>Feb</span><span>Mar</span></div>
                </div>
              </div>
            </article>
          </section>
        </template>

        <section class="panel employee-panel">
          <div class="panel-heading directory-heading">
            <div><h2>{{ activeView === 'Overview' ? 'Team members' : 'All employees' }}</h2><p>{{ activeView === 'Overview' ? 'Your people at a glance' : `${employees.length} people in your workspace` }}</p></div>
            <div class="directory-tools">
              <label class="table-search"><Search :size="15" /><input v-model="searchTerm" type="search" placeholder="Search employees" aria-label="Search employees in table" /></label>
              <label class="filter-select"><SlidersHorizontal :size="15" /><select :value="departmentFilter" aria-label="Filter by department" @change="setDepartment"><option>All departments</option><option v-for="department in departments" :key="department">{{ department }}</option></select><ChevronDown :size="13" /></label>
              <label class="filter-select status-filter"><select :value="statusFilter" aria-label="Filter by status" @change="setStatus"><option>All status</option><option>Active</option><option>On leave</option></select><ChevronDown :size="13" /></label>
            </div>
          </div>
          <div class="table-scroll">
            <table>
              <thead>
                <tr>
                  <th><button class="sort-button" @click="sortAscending = !sortAscending">Name <ArrowDownUp :size="13" /></button></th>
                  <th>Department</th>
                  <th>Status</th>
                  <th>Date joined</th>
                  <th><span class="sr-only">Actions</span></th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="employee in paginatedEmployees" :key="employee.id">
                  <td>
                    <div class="employee-identity">
                      <div class="avatar" :class="`avatar-${employee.color}`">{{ employee.name.split(' ').map((part) => part[0]).slice(0, 2).join('') }}</div>
                      <div class="employee-copy"><strong>{{ employee.name }}</strong><span>{{ employee.role }} <span class="email-separator">·</span> <span class="employee-email">{{ employee.email }}</span></span></div>
                    </div>
                  </td>
                  <td><span class="department-pill"><i></i>{{ employee.department }}</span></td>
                  <td><span class="status-pill" :class="employee.status === 'Active' ? 'status-active' : 'status-leave'"><i></i>{{ employee.status }}</span></td>
                  <td class="date-cell">{{ employee.joined }}</td>
                  <td>
                    <div class="row-actions">
                      <button aria-label="Edit employee" title="Edit employee" @click="openEditEmployee(employee)"><Settings :size="15" /></button>
                      <button aria-label="Remove employee" title="Remove employee" @click="removeEmployee(employee)"><X :size="16" /></button>
                    </div>
                  </td>
                </tr>
                <tr v-if="paginatedEmployees.length === 0">
                  <td colspan="5" class="empty-state"><div class="empty-icon"><Search :size="19" /></div><strong>No employees found</strong><span>Try adjusting your search or filters.</span></td>
                </tr>
              </tbody>
            </table>
          </div>
          <footer class="table-footer">
            <span>{{ pageLabel }} employees</span>
            <div class="pagination">
              <button :disabled="page <= 1" aria-label="Previous page" @click="page--"><ArrowLeft :size="15" /> Previous</button>
              <button :disabled="page >= pageCount" aria-label="Next page" @click="page++">Next <ArrowRight :size="15" /></button>
            </div>
          </footer>
        </section>
        <footer class="page-footer"><span>© 2024 AIO Studio</span><span><i></i> All systems operational</span></footer>
      </div>
    </main>

    <div v-if="showEmployeeModal" class="modal-backdrop" @click.self="showEmployeeModal = false">
      <section class="employee-modal" role="dialog" aria-modal="true" :aria-labelledby="'employee-modal-title'">
        <div class="modal-heading">
          <div><div class="modal-icon"><UsersRound :size="18" /></div><h2 id="employee-modal-title">{{ editingEmployeeId === null ? 'Add an employee' : 'Edit employee' }}</h2><p>{{ editingEmployeeId === null ? 'Add someone new to your workspace.' : 'Update this team member’s information.' }}</p></div>
          <button class="modal-close" aria-label="Close dialog" @click="showEmployeeModal = false"><X :size="18" /></button>
        </div>
        <form class="employee-form" @submit.prevent="saveEmployee">
          <label>Full name<input v-model="employeeForm.name" required maxlength="80" placeholder="e.g. Jamie Smith" /></label>
          <label>Work email<div class="input-with-icon"><Mail :size="16" /><input v-model="employeeForm.email" required type="email" maxlength="120" placeholder="jamie@company.com" /></div></label>
          <label>Job title<input v-model="employeeForm.role" required maxlength="80" placeholder="e.g. Product Designer" /></label>
          <div class="form-row">
            <label>Department<select v-model="employeeForm.department"><option v-for="department in departments" :key="department">{{ department }}</option></select></label>
            <label>Status<select v-model="employeeForm.status"><option>Active</option><option>On leave</option></select></label>
          </div>
          <div class="modal-actions"><button type="button" class="button button-secondary" @click="showEmployeeModal = false">Cancel</button><button type="submit" class="button button-primary"><Plus v-if="editingEmployeeId === null" :size="16" /><Check v-else :size="16" />{{ editingEmployeeId === null ? 'Add employee' : 'Save changes' }}</button></div>
        </form>
      </section>
    </div>

    <Transition name="toast"><div v-if="notice" class="toast-message"><Check :size="17" />{{ notice }}</div></Transition>
  </div>
</template>
