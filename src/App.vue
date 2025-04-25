<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { Card, Select, Menubar } from 'primevue'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'

// Year dropdown options
const yearOptions = Array.from({ length: 2024 - 1984 + 1 }, (_, i) => ({
  label: `${1984 + i}`,
  value: 1984 + i,
}))

const selectedYear = ref<number>(2024)
const svgRef = ref<SVGSVGElement | null>(null)

onMounted(async () => {
  const us = await fetch('https://cdn.jsdelivr.net/npm/us-atlas@3/counties-10m.json').then((r) =>
    r.json(),
  )

  const svg = d3
    .select(svgRef.value)
    .attr('viewBox', '0 0 975 610')
    .attr('width', 960)
    .attr('height', 610)
    .style('display', 'block')

  // Keep your Albers projection
  const projection = d3
    .geoAlbersUsa()
    .scale(1280)
    .translate([975 / 2, 610 / 2])

  const path = d3.geoPath(projection)

  const g = svg
    .append('g')
    .attr('fill', 'none')
    .attr('stroke', '#fff')
    .attr('stroke-linejoin', 'round')
    .attr('stroke-linecap', 'round')

  // Counties borders by state
  g.append('path')
    .attr('stroke', '#fff')
    .attr('stroke-width', 0.5)
    .attr(
      'd',
      path(
        topojson.mesh(
          us,
          us.objects.counties,
          (a, b) => a !== b && (((a.id as number) / 1000) | 0) === (((b.id as number) / 1000) | 0),
        ),
      ),
    )

  // State borders
  g.append('path')
    .attr('stroke', '#fff')
    .attr('stroke-width', 0.5)
    .attr('d', path(topojson.mesh(us, us.objects.states, (a, b) => a !== b)))

  // Nation outline
  g.append('path')
    .attr('d', path(topojson.feature(us, us.objects.nation)))
    .attr('stroke', '#fff')
    .attr('stroke-width', 0.5)
})
</script>

<template>
  <div class="min-h-screen flex flex-col">
    <!-- Navigation Bar -->
    <Menubar class="mt-4 mx-4">
      <template #start>
        <span class="text-xl font-semibold px-4">Wildfire Map</span>
      </template>
      <template #end>
        <Select
          v-model="selectedYear"
          :options="yearOptions"
          optionLabel="label"
          :defaultValue="selectedYear"
          :placeholder="String(selectedYear)"
          class="w-32 mx-4"
        />
      </template>
    </Menubar>

    <!-- Map Container -->
    <main class="flex-1 flex flex-col p-4">
      <Card
        id="map-container"
        class="w-full flex-1 rounded-lg shadow overflow-hidden border border-b-neutral-200"
      >
        <template #content>
          <Suspense>
            <svg ref="svgRef" viewBox="0 0 975 610"></svg>
          </Suspense>
        </template>
      </Card>
    </main>
  </div>
</template>
