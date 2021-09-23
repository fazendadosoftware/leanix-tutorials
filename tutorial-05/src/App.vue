<script setup>
import { ref } from 'vue'
import '@leanix/reporting'

// variable to hold the graphql query response
const response = ref(null)
// array that will hold the transformed response, in form of rows
const rows = ref([])
// array to store the table's columns key and label
const columns = ref([])
// variable to hold the computed average completion ratio for all factsheets
const avgCompletion = ref('n/a')


const initializeReport = async () => {
  await lx.init()
  lx.ready({})
}

const fetchGraphQLData = async () => {
  const query = `
    {
      allFactSheets(factSheetType: Application) {
        edges {
          node {
            name
            completion {
              completion
            }
          }
        }
      }
    }`
  lx.showSpinner()
  try {
    response.value = await lx.executeGraphQL(query)
    mapResponseToRows()
  } finally {
    lx.hideSpinner()
  }
}

const mapResponseToRows = () => {
  if (response.value === null) return
  // destructure the this.response state variable, extracting the allFactSheets.edges array
  rows.value = response.value.allFactSheets.edges // <- this is an Array
    // .map(edge => edge.node) <- the previous mapping operation
    .map(edge => {
      let { name, completion } = edge.node
      const  completionValue = completion.completion // we'll store the percentage value for computing later the total average completion ratio of our entire dataset
      completion = (completion.completion * 100).toFixed(1) + '%' // the percentage representation, rounded with 1 decimal place
      return { name, completion, completionValue }
    })
    .filter(row => row.completionValue < 0.1) // <- our filtering method
  computeTableColumns() // <-- call the computeTableColumns method here!
  // We'll compute the sum of the completion ratio of all our applications
  const completionSum = rows.value
    .reduce((accumulator, row) =>  accumulator + row.completionValue, 0)
  // and divide it by the number of applications and store it in percentage notation, rounded to 1 decimal places.
  avgCompletion.value = ((completionSum / rows.value.length) * 100).toFixed(1) + '%'
}

const computeTableColumns = () => {
  const  columnKeys = ['name', 'completion']
  columns.value = columnKeys
    .map(key  => ({ key, label:  lx.translateField('Application', key) }))
}

// we call our initializeReport method here...
initializeReport()
fetchGraphQLData()
</script>

<template>
  <div class="mx-auto h-screen">
    <div class="h-full flex flex-col pt-4">
      <div class="flex overflow-hidden h-full -pl-16">
        <div class="w-1/3 flex flex-col border mr-4 rounded bg-red-100">
          <div class="text-center py-2 text-xl uppercase font-semibold border-b bg-red-600 text-white rounded-t">
            GraphQL Query Response
          </div>
          <pre class="px-4 text-sm overflow-auto">
            {{response}}
          </pre>
        </div>
        <div class="w-1/3 flex flex-col border mr-4 rounded bg-green-100">
          <div class="text-center py-2 text-xl uppercase font-semibold border-b bg-green-600 text-white rounded-t">
            Transformed Data
          </div>
          <pre class="px-4 text-sm overflow-auto">
            {{rows}}
          </pre>
        </div>
        <div class="w-1/3 flex flex-col border mr-4 rounded bg-yellow-100">
          <div class="text-center py-2 text-xl uppercase font-semibold border-b bg-yellow-600 text-white rounded-t">
            Table View
          </div>
          <div class="overflow-auto px-4">
            <table class="table-fixed w-full text-sm text-center">
              <thead>
                <tr>
                  <th
                    v-for="column in columns"
                    :key="column.key"
                    class="px-4 py-1 text-base uppercase bg-white border">
                    {{column.label}}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="(row, idx) in rows"
                  :key="idx"
                  class="bg-white">
                  <td
                    v-for="column in columns"
                    :key="column.key"
                    class="border px-4 py-2">
                    {{row[column.key]}}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
      <div class="flex items-center justify-center h-48">
        <div class="bg-blue-600 text-white px-4 py-2 flex flex-col items-center rounded shadow">
          <div class="text-3xl uppercase font-bold">
            Average completion
          </div>
          <div class="text-6xl leading-none">
            {{avgCompletion}}
          </div>
          </div>
      </div>
    </div>
  </div>
</template>
