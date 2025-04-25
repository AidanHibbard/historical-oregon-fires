<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { Card, Select, Menubar, InputText } from 'primevue'
import * as d3 from 'd3'
import * as topojson from 'topojson-client'

// -- Year dropdown (1984–2024) ---------------------
const yearOptions = Array.from({ length: 2024 - 1984 + 1 }, (_, i) => ({
  label: `${1984 + i}`,
  value: 1984 + i,
}))
const selectedYear = ref<number>(2024)
const query = ref('')

// -- Fire metadata ----------------------------------
const fireCount = ref(0)
const totalAcres = ref(0)
const selectedFire = ref<any>(null) // Track the selected fire

// -- SVG ref & common vars --------------------------
const svgRef = ref<SVGSVGElement | null>(null)
const WIDTH = 975
const HEIGHT = 610

let projection: d3.GeoProjection
let pathGen: d3.GeoPath<any, d3.GeoPermissibleObjects>
let svg: d3.Selection<SVGSVGElement, unknown, null, undefined>
let zoomGroup: d3.Selection<SVGGElement, unknown, null, undefined>

// -- Draw base map once ----------------------------
onMounted(async () => {
  svg = d3
    .select(svgRef.value!)
    .attr('viewBox', `0 0 ${WIDTH} ${HEIGHT}`)
    .attr('width', '100%')
    .attr('height', '100%')
    .style('background', '#18181b')

  zoomGroup = svg.append('g').attr('class', 'zoom-group')

  const us = await fetch('https://cdn.jsdelivr.net/npm/us-atlas@3/counties-10m.json').then((r) =>
    r.json(),
  )

  const nation = topojson.feature(us, us.objects.nation)
  projection = d3.geoAlbersUsa().fitSize([WIDTH, HEIGHT], nation)
  pathGen = d3.geoPath(projection)

  zoomGroup
    .append('path')
    .attr('d', pathGen(nation))
    .attr('stroke', '#ffffff')
    .attr('stroke-width', 0.5)

  zoomGroup
    .append('path')
    .datum(topojson.mesh(us, us.objects.states, (a, b) => a !== b))
    .attr('d', pathGen)
    .attr('fill', 'none')
    .attr('stroke', '#ffffff')
    .attr('stroke-width', 0.5)

  zoomGroup
    .append('path')
    .datum(
      topojson.mesh(
        us,
        us.objects.counties,
        (a, b) =>
          a !== b && Math.floor((a.id as number) / 1000) === Math.floor((b.id as number) / 1000),
      ),
    )
    .attr('d', pathGen)
    .attr('fill', 'none')
    .attr('stroke', '#aaa')
    .attr('stroke-width', 0.25)

  svg.call(
    d3
      .zoom<SVGSVGElement, unknown>()
      .scaleExtent([1, 8])
      .on('zoom', (event) => {
        zoomGroup.attr('transform', event.transform)
      }),
  )
})

// -- Watch for year changes & overlay fires -------
watch(
  selectedYear,
  async (yr) => {
    if (!zoomGroup || !pathGen) return

    zoomGroup.selectAll<SVGGElement, unknown>('.fire-layer').remove()

    const url = `/historical-oregon-fires/json/fires/us_fires_${yr.label}.topojson`
    let topo
    try {
      topo = await fetch(url).then((r) => r.json())
    } catch {
      console.error('Failed to load', url)
      return
    }

    const fires = topojson.feature(topo, topo.objects[`us_fires_${yr.label}`])

    fireCount.value = fires.features.length
    totalAcres.value = fires.features.reduce(
      (sum, feat) => sum + (feat.properties?.BurnBndAc ?? 0),
      0,
    )

    const fireLayer = zoomGroup.append('g').attr('class', 'fire-layer')

    fireLayer
      .selectAll('path')
      .data(fires.features)
      .join('path')
      .attr('d', pathGen)
      .attr('fill', 'red')
      .attr('fill-opacity', 0.4)
      .attr('stroke', 'darkred')
      .attr('stroke-width', 0.2)
      .on('click', (event, d) => {
        selectedFire.value = d.properties // Set selected fire details
      })

    fireLayer
      .selectAll('circle')
      .data(fires.features)
      .join('circle')
      .attr('cx', (d) => pathGen.centroid(d)[0])
      .attr('cy', (d) => pathGen.centroid(d)[1])
      .attr('r', 0.5)
      .attr('fill', 'red')
      .attr('stroke', 'white')
      .attr('stroke-width', 0.2)
      .on('click', (event, d) => {
        selectedFire.value = d.properties // Set selected fire details
      })
  },
  { immediate: true },
)
</script>

<template>
  <div class="min-h-screen flex flex-col">
    <Menubar class="mt-4 mx-4">
      <template #start>
        <span class="text-xl font-semibold px-4">Wildfire Map</span>
      </template>
      <template #end>
        <InputText v-model="query" placeholder="Find fire" />
        <Select
          v-model="selectedYear"
          :options="yearOptions"
          optionLabel="label"
          class="w-32 mx-4"
          placeholder="select year"
        />
      </template>
    </Menubar>

    <main class="flex-1 flex flex-col p-4">
      <!-- Map & Stats -->
      <div class="flex gap-4">
        <Card
          id="map-container"
          class="w-full flex-1 rounded-lg shadow overflow-hidden border border-b-neutral-200 max-h-[90vh]"
        >
          <template #content>
            <Suspense>
              <svg ref="svgRef" viewBox="0 0 975 610"></svg>
            </Suspense>
          </template>
        </Card>

        <!-- Metadata Panel -->
        <div class="w-64 bg-neutral-100 dark:bg-neutral-900 rounded-lg shadow p-4 text-sm">
          <p class="text-md">Total Fires: {{ fireCount }}</p>
          <p class="text-md">Total Acres Burned: {{ totalAcres.toLocaleString() }}</p>
          <hr />

          <!-- Fire details -->
          <div v-if="selectedFire" class="mt-4">
            <p><strong>Fire Name:</strong> {{ selectedFire?.Incid_Name }}</p>
            <p>
              <strong>Acreage Burned:</strong> {{ selectedFire?.BurnBndAc?.toLocaleString() }} acres
            </p>
            <p><strong>Comment:</strong> {{ selectedFire?.Comment || 'N/A' }}</p>
            <p>
              <strong>Ignition Date:</strong>
              {{
                selectedFire?.Ig_Date ? new Date(selectedFire.Ig_Date).toLocaleDateString() : 'N/A'
              }}
            </p>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>
