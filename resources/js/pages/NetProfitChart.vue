<script setup lang="ts">
import { Line } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, LineElement, CategoryScale, LinearScale, PointElement } from 'chart.js'
import { computed } from 'vue'

// Register required Chart.js components
ChartJS.register(Title, Tooltip, Legend, LineElement, CategoryScale, LinearScale, PointElement)

const props = defineProps<{
  labels: string[],
  values: number[]
}>()

const data = computed(() => ({
  labels: props.labels,
  datasets: [
    {
      label: 'Net Profit',
      data: props.values,
      borderColor: '#2563eb', // Tailwind blue-600
      backgroundColor: 'rgba(37, 99, 235, 0.2)',
      tension: 0.2
    }
  ]
}))

const options = {
  responsive: true,
  plugins: {
    legend: { display: false }
  }
}
</script>

<template>
  <Line :data="data" :options="options" />
</template>
