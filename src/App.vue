<script setup lang="ts">
import { computed, onMounted, reactive, ref, watch } from 'vue'
import {
  ArrowDownUp,
  ArrowUpRight,
  BriefcaseBusiness,
  Check,
  Download,
  LayoutDashboard,
  Mail,
  Plus,
  Search,
  Settings,
  UsersRound,
  X,
} from '@lucide/vue'

type Designation = 'Intern' | 'Associate' | 'Senior' | 'Manager'

type Employee = {
  id: number
  name: string
  email: string
  phone: string | null
  designation: Designation
  monthly_salary_package: number
  monthly_tax_value: number
  yearly_increasing_bonus: number
  monthly_net_salary: number
  yearly_net_salary: number
}

type EmployeeForm = {
  name: string
  email: string
  phone: string
  designation: Designation
  monthly_salary_package: string
}

type ApiCollection<T> = { data: T[] }
type PayrollReport = ApiCollection<Employee> & {
  summary: {
    employee_count: number
    monthly_salary_package_total: number
    monthly_tax_value_total: number
    monthly_net_salary_total: number
    yearly_increasing_bonus_total: number
    yearly_net_salary_total: number
  }
}

const designations: Designation[] = ['Intern', 'Associate', 'Senior', 'Manager']
const apiBaseUrl = (import.meta.env.VITE_API_BASE_URL || '/api').replace(/\/$/, '')
const employees = ref<Employee[]>([])
const searchTerm = ref('')
const loadingEmployees = ref(true)
const loadError = ref('')
const showEmployeeModal = ref(false)
const editingEmployeeId = ref<number | null>(null)
const savingEmployee = ref(false)
const formError = ref('')
const notice = ref('')
const activeView = ref('Overview')
let searchTimer = 0
let latestLoad = 0

const employeeForm = reactive<EmployeeForm>({
  name: '',
  email: '',
  phone: '',
  designation: 'Intern',
  monthly_salary_package: '',
})

const numberFormat = new Intl.NumberFormat('en-US', {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
})

const totalMonthlyNet = computed(() => employees.value.reduce((sum, employee) => sum + employee.monthly_net_salary, 0))
const totalYearlyNet = computed(() => employees.value.reduce((sum, employee) => sum + employee.yearly_net_salary, 0))
const totalMonthlyTax = computed(() => employees.value.reduce((sum, employee) => sum + employee.monthly_tax_value, 0))

async function apiRequest<T>(path: string, options: RequestInit = {}): Promise<T> {
  const response = await fetch(`${apiBaseUrl}${path}`, {
    ...options,
    headers: {
      Accept: 'application/json',
      ...(options.body ? { 'Content-Type': 'application/json' } : {}),
      ...options.headers,
    },
  })
  const payload = response.status === 204 ? null : await response.json()

  if (!response.ok) {
    const validationMessages = payload?.errors
      ? Object.values(payload.errors as Record<string, string[]>).flat().join(' ')
      : ''
    throw new Error(validationMessages || payload?.message || `Request failed (${response.status})`)
  }

  return payload as T
}

async function loadEmployees(phone = '') {
  const requestId = ++latestLoad
  loadingEmployees.value = true
  loadError.value = ''

  try {
    const query = phone.trim() ? `?phone=${encodeURIComponent(phone.trim())}` : ''
    const response = await apiRequest<ApiCollection<Employee>>(
      query ? `/employees/search${query}` : '/employees',
    )
    // Ignore stale search responses so results stay in sync with the latest phone query.
    if (requestId === latestLoad) employees.value = response.data
  } catch (error) {
    if (requestId === latestLoad) {
      loadError.value = error instanceof Error ? error.message : 'Unable to load employees'
    }
  } finally {
    if (requestId === latestLoad) loadingEmployees.value = false
  }
}

watch(searchTerm, (phone) => {
  window.clearTimeout(searchTimer)
  searchTimer = window.setTimeout(() => void loadEmployees(phone), 250)
})

onMounted(() => void loadEmployees())

function formatMoney(value: number): string {
  return numberFormat.format(value)
}

function notify(message: string) {
  notice.value = message
  window.setTimeout(() => {
    if (notice.value === message) notice.value = ''
  }, 3500)
}

function resetEmployeeForm() {
  employeeForm.name = ''
  employeeForm.email = ''
  employeeForm.phone = ''
  employeeForm.designation = 'Intern'
  employeeForm.monthly_salary_package = ''
  editingEmployeeId.value = null
  formError.value = ''
}

function openAddEmployee() {
  resetEmployeeForm()
  showEmployeeModal.value = true
}

function openEditEmployee(employee: Employee) {
  resetEmployeeForm()
  editingEmployeeId.value = employee.id
  employeeForm.phone = employee.phone ?? ''
  employeeForm.monthly_salary_package = String(employee.monthly_salary_package)
  showEmployeeModal.value = true
}

async function saveEmployee() {
  formError.value = ''
  savingEmployee.value = true

  const isEditing = editingEmployeeId.value !== null
  const payload = isEditing
    ? {
        phone: employeeForm.phone.trim(),
        monthly_salary_package: employeeForm.monthly_salary_package,
      }
    : {
        name: employeeForm.name.trim(),
        email: employeeForm.email.trim(),
        phone: employeeForm.phone.trim(),
        designation: employeeForm.designation,
        monthly_salary_package: employeeForm.monthly_salary_package,
      }

  try {
    const path = isEditing ? `/employees/${editingEmployeeId.value}` : '/employees'
    await apiRequest<Employee>(path, {
      method: isEditing ? 'PATCH' : 'POST',
      body: JSON.stringify(payload),
    })
    showEmployeeModal.value = false
    notify(isEditing ? 'Employee payroll details updated' : 'Employee added')
    await loadEmployees(searchTerm.value)
  } catch (error) {
    formError.value = error instanceof Error ? error.message : 'Unable to save employee'
  } finally {
    savingEmployee.value = false
  }
}

async function removeEmployee(employee: Employee) {
  if (!window.confirm(`Delete ${employee.name}? This cannot be undone.`)) return

  try {
    await apiRequest<null>(`/employees/${employee.id}`, { method: 'DELETE' })
    await loadEmployees(searchTerm.value)
    notify('Employee deleted')
  } catch (error) {
    notify(`Could not delete employee: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}

async function exportReport() {
  try {
    const report = await apiRequest<PayrollReport>('/employees/report')
    const headers = [
      'Name',
      'Email',
      'Phone',
      'Designation',
      'Monthly salary package',
      'Monthly tax value',
      'Yearly increasing bonus',
      'Monthly net salary',
      'Yearly net salary',
    ]
    const rows = report.data.map((employee) => [
      employee.name,
      employee.email,
      employee.phone ?? '',
      employee.designation,
      employee.monthly_salary_package,
      employee.monthly_tax_value,
      employee.yearly_increasing_bonus,
      employee.monthly_net_salary,
      employee.yearly_net_salary,
    ])
    const csv = [headers, ...rows]
      .map((row) => row.map((value) => `"${String(value).replace(/"/g, '""')}"`).join(','))
      .join('\r\n')
    const url = URL.createObjectURL(new Blob([csv], { type: 'text/csv;charset=utf-8' }))
    const link = document.createElement('a')
    link.href = url
    link.download = 'employee-payroll-report.csv'
    link.click()
    URL.revokeObjectURL(url)
    notify('Payroll report exported')
  } catch (error) {
    notify(`Could not export report: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}
</script>

<template>
  <div class="app-shell payroll-app">
    <aside class="sidebar">
      <a class="brand" href="#" aria-label="AIO home" @click.prevent="activeView = 'Overview'">
        <span class="brand-mark"><span></span><span></span><span></span><span></span></span>
        <span class="brand-name">aio<span>.</span></span>
      </a>

      <div class="workspace-switcher">
        <div class="workspace-avatar">A</div>
        <div class="workspace-copy">
          <span class="workspace-label">Workspace</span>
          <strong>AIO Studio</strong>
        </div>
      </div>

      <div class="nav-label">EMPLOYEE MANAGEMENT</div>
      <nav class="main-nav" aria-label="Main navigation">
        <button
          v-for="item in [
            { label: 'Overview', icon: LayoutDashboard },
            { label: 'Employees', icon: UsersRound },
            { label: 'Reports', icon: BriefcaseBusiness },
          ]"
          :key="item.label"
          class="nav-link"
          :class="{ 'is-active': activeView === item.label }"
          @click="activeView = item.label"
        >
          <component :is="item.icon" :size="18" :stroke-width="1.8" />
          <span>{{ item.label }}</span>
          <span v-if="item.label === 'Employees'" class="nav-count">{{ employees.length }}</span>
        </button>
      </nav>
    </aside>

    <main class="main-content">
      <header class="topbar">
        <div class="breadcrumbs"><span>Workspace</span><span class="crumb-slash">/</span><strong>{{ activeView }}</strong></div>
        <div class="topbar-actions">
          <label class="global-search">
            <Search :size="16" />
            <input v-model="searchTerm" type="search" placeholder="Search by phone..." aria-label="Search employees by phone" />
          </label>
          <div class="topbar-avatar">HR</div>
        </div>
      </header>

      <div class="page-content">
        <section class="page-heading">
          <div>
            <div class="eyebrow"><span class="eyebrow-dot"></span> PEOPLE OPERATIONS</div>
            <h1>{{ activeView === 'Reports' ? 'Payroll reports' : 'Employee management' }}</h1>
            <p>Manage employee details and review monthly and yearly payroll figures.</p>
          </div>
          <div class="heading-actions">
            <button class="button button-secondary" @click="exportReport"><Download :size="16" /> Export report</button>
            <button class="button button-primary" @click="openAddEmployee"><Plus :size="17" /> Add employee</button>
          </div>
        </section>

        <section class="metrics-grid payroll-metrics" aria-label="Payroll summary">
          <article class="metric-card">
            <div class="metric-top"><span>Total employees</span><span class="metric-icon icon-purple"><UsersRound :size="17" /></span></div>
            <div class="metric-value">{{ employees.length }}</div>
            <div class="metric-foot">Employees in the directory</div>
          </article>
          <article class="metric-card">
            <div class="metric-top"><span>Monthly net payroll</span><span class="metric-icon icon-green"><Check :size="17" /></span></div>
            <div class="metric-value money-value">{{ formatMoney(totalMonthlyNet) }}</div>
            <div class="metric-foot">After monthly tax deductions</div>
          </article>
          <article class="metric-card">
            <div class="metric-top"><span>Monthly tax total</span><span class="metric-icon icon-amber"><BriefcaseBusiness :size="17" /></span></div>
            <div class="metric-value money-value">{{ formatMoney(totalMonthlyTax) }}</div>
            <div class="metric-foot">Tax deducted from salaries</div>
          </article>
          <article class="metric-card">
            <div class="metric-top"><span>Yearly net payroll</span><span class="metric-icon icon-blue"><ArrowUpRight :size="17" /></span></div>
            <div class="metric-value money-value">{{ formatMoney(totalYearlyNet) }}</div>
            <div class="metric-foot">Includes yearly designation bonus</div>
          </article>
        </section>

        <section class="panel employee-panel">
          <div class="panel-heading directory-heading">
            <div>
              <h2>Employee payroll</h2>
              <p>Salary, tax, bonus, and take-home totals</p>
            </div>
            <div class="directory-tools">
              <label class="table-search">
                <Search :size="15" />
                <input v-model="searchTerm" type="search" placeholder="Search by phone" aria-label="Search employees by phone" />
              </label>
            </div>
          </div>

          <div class="table-scroll payroll-table-scroll">
            <table class="payroll-table">
              <thead>
                <tr>
                  <th><span class="sort-button">Employee <ArrowDownUp :size="13" /></span></th>
                  <th>Phone</th>
                  <th>Designation</th>
                  <th>Monthly package</th>
                  <th>Monthly tax</th>
                  <th>Yearly bonus</th>
                  <th>Monthly net</th>
                  <th>Yearly net</th>
                  <th><span class="sr-only">Actions</span></th>
                </tr>
              </thead>
              <tbody>
                <tr v-if="loadingEmployees">
                  <td colspan="9" class="empty-state">Loading employees…</td>
                </tr>
                <tr v-else-if="loadError">
                  <td colspan="9" class="empty-state">Could not load employees: {{ loadError }}</td>
                </tr>
                <tr v-for="employee in employees" :key="employee.id">
                  <td>
                    <div class="employee-identity">
                      <div class="avatar avatar-lavender">{{ employee.name.split(' ').map((part) => part[0]).slice(0, 2).join('') }}</div>
                      <div class="employee-copy">
                        <strong>{{ employee.name }}</strong>
                        <span class="employee-email">{{ employee.email }}</span>
                      </div>
                    </div>
                  </td>
                  <td class="date-cell">{{ employee.phone || 'Not provided' }}</td>
                  <td><span class="designation-pill">{{ employee.designation }}</span></td>
                  <td class="money-cell">{{ formatMoney(employee.monthly_salary_package) }}</td>
                  <td class="money-cell">{{ formatMoney(employee.monthly_tax_value) }}</td>
                  <td class="money-cell">{{ formatMoney(employee.yearly_increasing_bonus) }}</td>
                  <td class="money-cell"><strong>{{ formatMoney(employee.monthly_net_salary) }}</strong></td>
                  <td class="money-cell"><strong>{{ formatMoney(employee.yearly_net_salary) }}</strong></td>
                  <td>
                    <div class="row-actions">
                      <button aria-label="Edit phone and salary" title="Edit phone and salary" @click="openEditEmployee(employee)"><Settings :size="15" /></button>
                      <button aria-label="Delete employee" title="Delete employee" @click="removeEmployee(employee)"><X :size="16" /></button>
                    </div>
                  </td>
                </tr>
                <tr v-if="!loadingEmployees && !loadError && employees.length === 0">
                  <td colspan="9" class="empty-state">
                    <div class="empty-icon"><Search :size="19" /></div>
                    <strong>No employees found</strong>
                    <span>{{ searchTerm ? 'Try another phone number.' : 'Add an employee to get started.' }}</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <footer class="table-footer"><span>{{ employees.length }} employee{{ employees.length === 1 ? '' : 's' }}</span></footer>
        </section>
        <footer class="page-footer"><span>© 2026 AIO Studio</span><span><i></i> Payroll figures calculated by the API</span></footer>
      </div>
    </main>

    <div v-if="showEmployeeModal" class="modal-backdrop" @click.self="showEmployeeModal = false">
      <section class="employee-modal payroll-modal" role="dialog" aria-modal="true" aria-labelledby="employee-modal-title">
        <div class="modal-heading">
          <div>
            <div class="modal-icon"><UsersRound :size="18" /></div>
            <h2 id="employee-modal-title">{{ editingEmployeeId === null ? 'Add an employee' : 'Edit employee payroll' }}</h2>
            <p>{{ editingEmployeeId === null ? 'Payroll values are calculated automatically.' : 'Update the employee phone and monthly salary.' }}</p>
          </div>
          <button class="modal-close" aria-label="Close dialog" @click="showEmployeeModal = false"><X :size="18" /></button>
        </div>

        <form class="employee-form" @submit.prevent="saveEmployee">
          <template v-if="editingEmployeeId === null">
            <label>Full name<input v-model.trim="employeeForm.name" required maxlength="120" placeholder="e.g. Jamie Smith" /></label>
            <label>Work email
              <div class="input-with-icon"><Mail :size="16" /><input v-model.trim="employeeForm.email" required type="email" maxlength="255" placeholder="jamie@company.com" /></div>
            </label>
            <label>Designation
              <select v-model="employeeForm.designation" required>
                <option v-for="designation in designations" :key="designation" :value="designation">{{ designation }}</option>
              </select>
            </label>
          </template>

          <label>Phone number<input v-model.trim="employeeForm.phone" required maxlength="32" type="tel" placeholder="e.g. +1 555 010 1234" /></label>
          <label>Monthly salary package
            <input v-model="employeeForm.monthly_salary_package" required min="0" max="999999999.99" step="0.01" type="number" inputmode="decimal" placeholder="0.00" />
          </label>

          <p v-if="formError" class="form-error" role="alert">{{ formError }}</p>
          <div class="modal-actions">
            <button type="button" class="button button-secondary" @click="showEmployeeModal = false">Cancel</button>
            <button type="submit" class="button button-primary" :disabled="savingEmployee">
              <Plus v-if="editingEmployeeId === null" :size="16" />
              <Check v-else :size="16" />
              {{ savingEmployee ? 'Saving…' : editingEmployeeId === null ? 'Add employee' : 'Save changes' }}
            </button>
          </div>
        </form>
      </section>
    </div>

    <Transition name="toast">
      <div v-if="notice" class="toast-message"><Check :size="17" />{{ notice }}</div>
    </Transition>
  </div>
</template>
