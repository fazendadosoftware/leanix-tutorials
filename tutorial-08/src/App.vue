<script setup>
import { ref, watch } from 'vue'
import '@leanix/reporting'

const pageSizes = [100, 500, 1000, 15000]
const pageSize = ref(pageSizes[0])
const endCursor = ref(null)
const totalFactSheetCount = ref(null)
const downloadedFactSheetCount = ref(null)
const completionRatio = ref(null)
const timeToFirstResults = ref(null)
const totalDownloadingTime = ref(null)
const factSheetsPerSecond = ref(null)
const t0 = ref(null)
let timer = null

const initializeReport = async () => {
  await lx.init()
  lx.ready({})
}

const reset = () => {
  endCursor.value = null
  totalFactSheetCount.value = null
  downloadedFactSheetCount.value = null
  completionRatio.value = null
  timeToFirstResults.value = null
  totalDownloadingTime.value = null
  factSheetsPerSecond.value = null
  t0.value = null
  // stop the timer
  clearInterval(timer)
  timer = null
  fetchPage()
}

const fetchPage = async () => {
  const first = pageSize.value
  const after = endCursor.value
  const query = `
    query ($first: Int = 15000, $after: String) {
      allFactSheets(first: $first, after: $after) {
        totalCount
        pageInfo { hasNextPage endCursor }
        edges { node { id } }
      }
    }
  `
  const variables = { first, after }
  const _t0 = performance.now()
  const result = await lx.executeGraphQL(query, variables)
  const deltaT = performance.now() - _t0
  // if the user changed the pageSize while the query was being fetched from the server, and a reset was performed to the state, then discard the results of the query
  if (endCursor.value !== after) return
  // if this is the first page of our dataset (endCursor still null)
  if (after === null) {
    // save t0 into our state variable
    t0.value = _t0
    // compute the timeToFirst results, in miliseconds
    timeToFirstResults.value = Math.round(deltaT) + ' ms'
    // start the timer
    if (timer) clearInterval(timer)
    // set the first value of totalDownloadingTime and factSheetsPerSecond
    totalDownloadingTime.value = timeToFirstResults.value
    factSheetsPerSecond.value = Math.round(downloadedFactSheetCount.value / (deltaT / 1000))
    // and update them every second
    timer = setInterval(() => {
      const deltaT = performance.now() - t0.value
      totalDownloadingTime.value = Math.round(deltaT) + ' ms'
      factSheetsPerSecond.value = Math.round(downloadedFactSheetCount.value / (deltaT / 1000))
    }, 1000)
  }
  // we destructure the query response object into totalCount, hasNextPage, endCursor and edges variables
  const { allFactSheets: { totalCount, pageInfo: { hasNextPage = false, endCursor: _endCursor }, edges } } = result
  totalFactSheetCount.value = totalCount
  // update the downloadedFactSheetCount value
  downloadedFactSheetCount.value += edges.length
  // update the completion ratio
  completionRatio.value = Math.round((downloadedFactSheetCount.value / totalFactSheetCount.value) * 100) + '%'
  // if we just fetched the last page of our dataset then...
  if (hasNextPage === false) {
    // stop the timer
    clearInterval(timer)
    const deltaT = performance.now() - t0.value
    // update the final totalDownloadingTime and factSheetsPerSecond values accordingly
    totalDownloadingTime.value = Math.round(deltaT) + ' ms'
    factSheetsPerSecond.value = Math.round(downloadedFactSheetCount.value / (deltaT / 1000))
  } else endCursor.value = _endCursor // update our endCursor
}

// watcher for pageSize variable, triggers reset method
watch(pageSize, reset)
// watcher for endCursor variable, triggers fetchPage method
watch(endCursor, fetchPage, { immediate: true })

initializeReport()
</script>

<template>
  <div class="h-screen text-base flex flex-col items-center justify-center">
    <div class="flex flex-col items-center mb-10">
      <div class="text-4xl font-semibold">LeanIX Tutorial</div>
      <div class="text-2xl font-light">Querying paginated data in large workspaces</div>
    </div>
    <div class="w-full max-w-lg grid grid-cols-2 space-y-4 space-x-8 rounded p-4 bg-gray-100 shadow">
      <div class="col-span-2"></div>
      <label class="text-2xl font-semibold">Page size</label>
      <div class="relative">
        <select
          v-model.number="pageSize"
          class="block appearance-none w-full bg-white border border-gray-200 text-gray-700 py-3 px-4 pr-8 rounded leading-tight focus:outline-none focus:bg-white focus:border-gray-500">
          <option
            v-for="size in pageSizes"
            :selected="pageSize === size"
            :value="size"
            :key="size">
            {{size}}
          </option>
        </select>
        <div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2 text-gray-700">
          <svg class="fill-current h-4 w-4"  xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20"><path d="M9.293 12.95l.707.707L15.657 8l-1.414-1.414L10 10.828 5.757 6.586 4.343 8z"/></svg>
        </div>
      </div>
      <div class="col-span-2 border-t"/>
      <label class="font-semibold">Workspace factsheet count:</label>
      <div>{{totalFactSheetCount}}</div>
      <label class="font-semibold">Downloaded factsheets:</label>
      <div>{{downloadedFactSheetCount}}</div>
      <label class="font-semibold">Completion ratio:</label>
      <div>{{completionRatio}}</div>
      <label class="font-semibold">Time to first results:</label>
      <div>{{timeToFirstResults}}</div>
      <label class="font-semibold">Total downloading time:</label>
      <div>{{totalDownloadingTime}}</div>
      <label class="font-semibold">Factsheets per second:</label>
      <div>{{factSheetsPerSecond}}</div>
    </div>
  </div>
</template>