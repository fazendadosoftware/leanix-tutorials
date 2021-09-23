<script setup>
import { ref, watch } from 'vue'
import '@leanix/reporting'

// variable that will store our workspace's baseUrl
const baseUrl = ref(null)
// variable for storing the workspace's applications
const applications = ref([])
// variable for storing the workspace's application lifecycle phases view model
const applicationLifecyclePhases = ref({})

// the state variable that will hold the matrix top row containing the column headers/names
const headerRow = ref([])
// the state variable that will hold our application rows
const rows = ref([])
// will dynamically set the number of columns in our grid, based on the lifecycle phases fetched from workspace
const gridStyle = ref('')

// the report initialization method
const initializeReport = async () => {
  const reportSetup = await lx.init()

  applicationLifecyclePhases.value = reportSetup.settings.viewModel.factSheets
    .find(({ type }) => type === 'Application')
    .fieldMetaData.lifecycle.values

  // we extract our workspace's baseUrl from the reportSetup object
  // and store it into the baseUrl state variable
  baseUrl.value = reportSetup.settings.baseUrl

  const config = {
    facets: [
      {
        key: 1,
        fixedFactSheetType: 'Application',
        attributes: ['name', 'type', 'lifecycle {asString phases {phase startDate}}'],
        callback: dataset => { applications.value = dataset }
      }
    ]
  }
  lx.ready(config)
}

// auxiliary method for translating our application lifecycle fields
const translateLifecycleField = ({ type, lifecycle }) => lifecycle
  ? lx.translateFieldValue(type, 'lifecycle', lifecycle.asString)
  : 'n/a'

const computeRows = () => {
  const lifecycleColumnKeys = Object.keys(applicationLifecyclePhases.value)

  headerRow.value = Object.entries(applicationLifecyclePhases.value)
    .reduce((accumulator, [key, { bgColor, color }]) => {
      const label = lx.translateFieldValue('Application', 'lifecycle', key)
      accumulator.push({
        key,
        label,
        classes: 'text-center font-bold py-2',
        style: `color:${color};background:${bgColor}`
      })
      return accumulator
    }, [{ key: 'pivot-cell' }])

  gridStyle.value = `grid-template-columns: 250px repeat(${headerRow.value.length - 1}, 150px)`

  rows.value = applications.value
    .map(({ id, type, name, lifecycle }) => {
      // first column containing the name of the application
      const headerColumn = {
        key: id,
        type,
        label: name,
        classes: 'hover:underline cursor-pointer border-r font-bold py-2'
      }
      let lifecyclePhaseColumns = []
      // if application doesn't have a lifecycle defined
      if (lifecycle === null) lifecyclePhaseColumns = lifecycleColumnKeys
        .map(key => ({ key: `${id}_${key}`, label: null, classes: 'border-r last:border-r-0' }))
      else {
        let { asString, phases } = lifecycle
        const { bgColor, color } = applicationLifecyclePhases.value[asString]
        phases = phases.reduce((accumulator, { phase, startDate }) => {
          accumulator[phase] = startDate
          return accumulator
        }, {})
        lifecyclePhaseColumns = lifecycleColumnKeys
          .map(key => ({
            key: `${id}_${key}`,
            label: phases[key] || null,
            classes: `border-r last:border-r-0 py-2`,
            style: key === asString ? `color:${color};background:${bgColor}80` : ''
          }))
        headerColumn.style = `color:${color};background:${bgColor}`
      }
      return [headerColumn, ...lifecyclePhaseColumns]
    })
}

const applicationClickEvtHandler = ({ type = null, key: id }) => {
  if (type === null) return
  const url = `${baseUrl.value}/factsheet/${type}/${id}`
  lx.openLink(url)
}

watch([applications], computeRows)
// we call our report initialization method here
initializeReport()
</script>

<template>
  <div class="container mx-auto h-screen text-xs">
    <div class="flex flex-col items-center h-full overflow-hidden">
      <div class="grid border-b border-r" :style="gridStyle">
        <div
          v-for="cell in headerRow"
          :key="cell.key"
          :class="cell.classes"
          :style="cell.style">
          {{cell.label}}
        </div>
      </div>
      <div class="h-full overflow-y-auto overflow-x-hidden">
        <div
          v-for="(row, rowIdx) in rows"
          :key="rowIdx"
          class="grid border-r border-l hover:bg-gray-100 transition-colors duration-150 ease-in-out"
          :style="gridStyle">
          <div
            v-for="cell in row"
            :key="cell.key"
            class="text-center border-r border-b"
            :class="cell.classes"
            :style="cell.style"
            @click="applicationClickEvtHandler(cell)">
            {{cell.label}}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
