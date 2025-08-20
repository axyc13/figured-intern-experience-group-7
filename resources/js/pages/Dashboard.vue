<script setup lang="ts">
import { Head, Link } from '@inertiajs/vue3'
import { ref, onMounted, computed, watch } from 'vue'

// --- State ---
const reportData = ref<any>(null)
const loading = ref(true)
const error = ref<string | null>(null)

// Chart.js (via CDN or install chart.js & vue-chartjs for production)
import { Chart, registerables } from 'chart.js'
Chart.register(...registerables)
import { onBeforeUnmount } from 'vue'

const chartRef = ref<HTMLCanvasElement | null>(null)
let chartInstance: Chart | null = null

// --- Fetch ---
async function fetchFinancialData(){
  try{
    loading.value = true
    const res = await fetch('/api/financial-report')
    if(!res.ok) throw new Error('Failed to fetch financial data')
    reportData.value = await res.json()
  }catch(e:any){
    error.value = e?.message || 'An error occurred'
  }finally{
    loading.value = false
  }
}

onMounted(fetchFinancialData)

// --- Helpers ---
function formatCurrency(n:number){
  return new Intl.NumberFormat('en-NZ', { style: 'currency', currency: 'NZD', maximumFractionDigits: 0 }).format(n || 0)
}

const lastColIndex = computed(() => (reportData.value?.columns?.length ?? 1) - 1)

function sectionById(id:string){
  return reportData.value?.sections?.find((s:any) => s.id === id)
}

// Totals
const incomeTotal = computed(() => sectionById('income')?.total?.values?.[lastColIndex.value] ?? 0)
const opexTotal = computed(() => sectionById('operating_expenses')?.total?.values?.[lastColIndex.value] ?? 0)
const netProfit = computed(() => reportData.value?.summary?.find((r:any)=>r.name==='Net Profit')?.values?.[lastColIndex.value] ?? 0)
const operSurplus = computed(() => reportData.value?.summary?.find((r:any)=>r.name==='Operating Surplus')?.values?.[lastColIndex.value] ?? 0)

// Recent (last 4 months, excluding Total)
const recentMonths = computed(() => (reportData.value?.columns || []).slice(-5, -1))
const recentNetProfit = computed(() => (reportData.value?.summary?.find((r:any)=>r.name==='Net Profit')?.values || []).slice(-5, -1))

// --- Chart Data ---
const chartLabels = computed(() =>
  (reportData.value?.columns || [])
    .filter((c:any) => c.month !== 'Total')
    .map((c:any) => c.month)
)

const netProfitByMonth = computed(() =>
  (reportData.value?.summary?.find((r:any)=>r.name==='Net Profit')?.values || []).slice(0, -1)
)

const incomeByMonth = computed(() =>
  sectionById('income')?.total?.values?.slice(0, -1) || []
)

const opexByMonth = computed(() =>
  sectionById('operating_expenses')?.total?.values?.slice(0, -1) || []
)

// --- Chart Render ---
function renderChart() {
  if (!chartRef.value || !reportData.value) return

  if (chartInstance) {
    chartInstance.destroy()
  }

  chartInstance = new Chart(chartRef.value, {
    type: 'line',
    data: {
      labels: chartLabels.value,
      datasets: [
        {
          label: 'Net Profit',
          data: netProfitByMonth.value,
          borderColor: '#2563eb',
          backgroundColor: 'rgba(37,99,235,0.1)',
          tension: 0.3,
          fill: false,
        },
        {
          label: 'Income',
          data: incomeByMonth.value,
          borderColor: '#22c55e',
          backgroundColor: 'rgba(34,197,94,0.1)',
          tension: 0.3,
          fill: false,
        },
        {
          label: 'Operating Expenses',
          data: opexByMonth.value,
          borderColor: '#ef4444',
          backgroundColor: 'rgba(239,68,68,0.1)',
          tension: 0.3,
          fill: false,
        }
      ]
    },
    options: {
      responsive: true,
      plugins: {
        legend: { display: true }
      },
      scales: {
        y: {
          beginAtZero: false,
          ticks: {
            callback: function(value: number) {
              return formatCurrency(value)
            }
          }
        }
      }
    }
  })
}

watch(reportData, () => {
  if (reportData.value) {
    setTimeout(renderChart, 100) // wait for DOM
  }
})

onBeforeUnmount(() => {
  if (chartInstance) chartInstance.destroy()
})
</script>

<template>
  <Head title="P&L Dashboard" />
  <div class="min-h-screen bg-gray-50 py-8 px-4">
    <div class="max-w-6xl mx-auto">
      <!-- Header -->
      <div class="mb-6 flex items-center justify-between">
        <div>
          <h1 class="text-2xl md:text-3xl font-bold text-gray-900">🌾 Windy Farm — Profit & Loss</h1>
          <p class="text-gray-600 text-sm" v-if="reportData">
            {{ reportData.company.report_type }} • {{ reportData.company.basis }} • {{ reportData.company.period }}
          </p>
        </div>
        <div class="flex items-center gap-3">
          <button @click="fetchFinancialData" class="px-3 py-2 rounded-lg border border-gray-300 bg-white hover:bg-gray-100 text-sm">Reload</button>
          <Link href="/" class="text-sm text-gray-600 hover:text-gray-900">← Back</Link>
        </div>
      </div>

      <!-- Loading -->
      <div v-if="loading" class="flex items-center justify-center py-16">
        <div class="animate-spin rounded-full h-10 w-10 border-b-2 border-blue-600"></div>
        <span class="ml-3 text-gray-600">Loading…</span>
      </div>

      <!-- Error -->
      <div v-else-if="error" class="bg-red-50 border border-red-200 rounded-lg p-6">
        <h2 class="text-red-800 font-semibold mb-2">Error Loading Data</h2>
        <p class="text-red-700">{{ error }}</p>
        <button @click="fetchFinancialData" class="mt-4 bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700">Try again</button>
      </div>

      <!-- Super basic dashboard -->
      <div v-else-if="reportData" class="space-y-6">
        <!-- KPI cards -->
        <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
          <div class="bg-white border border-gray-200 rounded-2xl p-4 shadow-sm">
            <div class="text-xs uppercase text-gray-500">Income (Total)</div>
            <div class="text-2xl font-semibold mt-1">{{ formatCurrency(incomeTotal) }}</div>
          </div>
          <div class="bg-white border border-gray-200 rounded-2xl p-4 shadow-sm">
            <div class="text-xs uppercase text-gray-500">Operating Expenses (Total)</div>
            <div class="text-2xl font-semibold mt-1">{{ formatCurrency(opexTotal) }}</div>
          </div>
          <div class="bg-white border border-gray-200 rounded-2xl p-4 shadow-sm">
            <div class="text-xs uppercase text-gray-500">Operating Surplus (Total)</div>
            <div class="text-2xl font-semibold mt-1" :class="operSurplus < 0 ? 'text-red-600' : ''">{{ formatCurrency(operSurplus) }}</div>
          </div>
          <div class="bg-white border border-gray-200 rounded-2xl p-4 shadow-sm">
            <div class="text-xs uppercase text-gray-500">Net Profit (Total)</div>
            <div class="text-2xl font-semibold mt-1" :class="netProfit < 0 ? 'text-red-600' : ''">{{ formatCurrency(netProfit) }}</div>
          </div>
        </div>

        <!-- Chart -->
        <div class="bg-white border border-gray-200 rounded-2xl p-6 shadow-sm">
          <h3 class="text-sm font-semibold text-gray-800 mb-3">Monthly Trends</h3>
          <div class="w-full overflow-x-auto">
            <canvas ref="chartRef" height="80"></canvas>
          </div>
        </div>

        <!-- Recent net profit table (tiny) -->
        <div class="bg-white border border-gray-200 rounded-2xl p-4 shadow-sm overflow-x-auto">
          <h3 class="text-sm font-semibold text-gray-800 mb-3">Recent Net Profit</h3>
          <table class="min-w-[420px] w-full text-sm">
            <thead class="text-gray-500">
              <tr>
                <th class="text-left p-2">Month</th>
                <th class="text-right p-2">Amount</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(m, idx) in recentMonths" :key="m.month" class="border-t">
                <td class="p-2">{{ m.month }}</td>
                <td class="p-2 text-right" :class="recentNetProfit[idx] < 0 ? 'text-red-600' : ''">
                  {{ formatCurrency(recentNetProfit[idx]) }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Link to raw API (handy for reviewers) -->
        <div class="text-center">
          <a href="/api/financial-report" target="_blank" class="inline-flex items-center gap-2 px-4 py-2 rounded-lg bg-gray-900 text-white hover:bg-black text-sm">
            🔗 View API Data
          </a>
        </div>
      </div>
    </div>
  </div>
</template>