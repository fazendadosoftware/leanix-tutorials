<script setup>
import '@leanix/reporting'
import { ref, watch, computed } from 'vue'
// Chart.js dependency
import Chart from 'chart.js/auto'
// tinygradient dependency
import tinygradient from 'tinygradient'

// reactive variable that will hold the workspace factsheet types
const factSheetTypes = ref([])
// reactive variable will hold the selected factsheet type
const selectedFactSheetType = ref(null)
// reactive variable that will hold the computed averageCompletion statistic
const averageCompletion = ref(null)

// holder variable for our chart (non-reactive)
let chart

// reactive holder for our canvas element
const chartCanvas = ref(null)

// we define our report initialization method, it is an async one so that we
// can syncronously await for Promise results
const initializeReport = async () => {
  const setup = await lx.init()
  // we extract here the list of all factsheet types available in the workspace and store it
  factSheetTypes.value = Object.keys(setup.settings.dataModel.factSheets)
  // and select the first factsheet type of that list
  selectedFactSheetType.value = factSheetTypes.value?.[0] || null
  const config = {
    menuActions: {
      // we enable here the standard "Settings" button
      showConfigure: true,
      // and set the callback for opening the configuration modal defined ahead
      configureCallback: openReportConfigurationModal
    }
  }
  // we configure our custom report with an empty configuration object
  return lx.ready(config)
}

// method for opening up the configuration modal and show a single select input containing the list of available factsheet types
const openReportConfigurationModal = async () => {
  const fields = {
    factSheetType: {
      type: 'SingleSelect',
      label: 'FactSheet Type',
      options: factSheetTypes.value
        .map(factSheetType => ({
          value: factSheetType,
          label: lx.translateFactSheetType(factSheetType)
        })
        )
    }
  }
  const initialValues = { factSheetType: selectedFactSheetType.value }
  const values = await lx.openFormModal(fields, initialValues)
  if (values) selectedFactSheetType.value = values.factSheetType
}

// method that will query the workspace for the completion value of the selectedFactSheetType
const fetchGraphQLData = async () => {
  const query = 'query($factSheetType:FactSheetType){allFactSheets(factSheetType:$factSheetType){edges{node{completion{completion}}}}}'
    try {
    lx.showSpinner()
    averageCompletion.value = await lx.executeGraphQL(query, { factSheetType: selectedFactSheetType.value })
      .then(({ allFactSheets }) => {
        const completionSum = allFactSheets.edges.reduce((accumulator, { node }) =>  accumulator += node.completion.completion, 0)
        const factSheetCount = allFactSheets.edges.length
        const averageCompletion = completionSum / factSheetCount
        return averageCompletion
      })
  } finally {
    lx.hideSpinner()
  }
}

// Our method for updating the chart
const updateChart = () => {
  const gradient = tinygradient([
    { color: 'red', pos: 0 },
    { color: 'yellow', pos: 0.3 },
    { color: 'green', pos: 1 }
  ])
  const data = [averageCompletion.value, 1 - averageCompletion.value]
  const backgroundColor = [gradient.rgbAt(averageCompletion.value).toHexString(), 'rgba(0, 0, 0, 0.1)']
  if (typeof chart === 'undefined') {
    const config = {
      type: 'doughnut',
      data: { datasets: [{ data, backgroundColor }] },
      options: {
        circumference: 180,
        rotation: -90,
        tooltips: { enabled: false },
        hover: { mode: null }
      }
    }
    const ctx = chartCanvas.value.getContext('2d')
    chart = new Chart(ctx, config)
  } else {
    chart.data.datasets[0] = { data, backgroundColor }
    chart.update()
  }
}

// computed variable for translating the selected factsheet type
const factSheetTypeLabel = computed(() => selectedFactSheetType.value !== null
  ? lx.translateFactSheetType(selectedFactSheetType.value, 'plural')
  : null)

// watcher that will trigger the fetchGraphQLData on every selectedFactSheetType update
watch(selectedFactSheetType, fetchGraphQLData)

// watcher that will trigger the updateChart method on every averageCompletion variable update
watch(averageCompletion, updateChart)

// we call our report initialization method here...
initializeReport()
</script>

<template>
  <div class="container mx-auto text-md text-gray-800">
    <!-- chart container -->
    <div class="relative flex flex-col flex-wrap items-center mt-16 -mx-8 mt-16">
      <!-- chart title -->
      <div class="text-4xl mb-2">Average Completion Ratio for</div>
      <!-- chart subtitle -->
      <div class="text-6xl font-bold">
        {{factSheetTypeLabel}}
      </div>
      <!-- chart legend -->
      <div class="absolute bottom-12 font-bold text-4xl">
        {{(averageCompletion * 100).toFixed(0)}}%
      </div>
      <!-- canvas container -->
      <div>
        <canvas ref="chartCanvas"/>
      </div>
    </div>
  </div>
</template>