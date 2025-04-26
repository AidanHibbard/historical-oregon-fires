<script setup lang="ts">
import { onMounted, ref } from 'vue'
import Chart from 'primevue/chart'
import { Card } from 'primevue'

interface SummaryRecord {
  year: number
  total_fires: number
  total_acres: number
}

// Chart data refs
const fireChartData = ref({ labels: [] as string[], datasets: [] })
const acresChartData = ref({ labels: [] as string[], datasets: [] })

// Chart options (you can tweak styling here)
const commonOptions = {
  responsive: true,
  plugins: {
    legend: { position: 'top' as const },
    tooltip: { mode: 'index' as const, intersect: false },
  },
  scales: {
    x: { title: { display: true, text: 'Year' } },
    y: { title: { display: true } },
  },
}

onMounted(async () => {
  // 1) load summary
  const response = await fetch('/historical-oregon-fires/json/wildfire_summary.json')
  const data: SummaryRecord[] = await response.json()

  // 2) sort by year (just in case)
  data.sort((a, b) => a.year - b.year)

  // 3) extract labels & values
  const labels = data.map((d) => String(d.year))
  const fireCounts = data.map((d) => d.total_fires)
  const acreValues = data.map((d) => d.total_acres)

  // 4) build chart models
  fireChartData.value = {
    labels,
    datasets: [
      {
        label: 'Number of Fires',
        backgroundColor: '#42A5F5',
        borderColor: '#1E88E5',
        data: fireCounts,
      },
    ],
  }

  acresChartData.value = {
    labels,
    datasets: [
      {
        label: 'Acres Burned',
        backgroundColor: 'rgba(255,99,132,0.5)',
        borderColor: 'rgba(255,99,132,1)',
        data: acreValues,
        fill: true,
      },
    ],
  }
})
</script>

<template>
  <div class="mt-6 grid grid-cols-1 md:grid-cols-2 gap-4">
    <!-- Fires per Year -->
    <Card class="p-4 rounded">
      <template #title>
        <h3 class="text-lg font-semibold mb-2">Fires Per Year</h3>
      </template>
      <template #content>
        <Chart type="bar" :data="fireChartData" :options="commonOptions" />
      </template>
    </Card>

    <!-- Acres Burned Per Year -->
    <Card class="p-4 rounded">
      <template #title>
        <h3 class="text-lg font-semibold mb-2">Acres Burned Per Year</h3>
      </template>
      <template #content>
        <Chart type="line" :data="acresChartData" :options="commonOptions" />
      </template>
    </Card>
  </div>
</template>
