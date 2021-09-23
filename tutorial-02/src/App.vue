<script setup>
import '@leanix/reporting'
import { ref, watch } from 'vue'

// reactive variable that will hold the workspace factsheet types
const factSheetTypes = ref([])
// reactive variable that stores some statistics for each factsheet type
const facetResultIndex = ref({})
// reactive variable that stores some statistics for each factsheet type fetch through the GraphQL API
const graphQLResultIndex = ref({})

// we define our report initialization method, it is an async one so that we
// can syncronously await for Promise results
const initializeReport = async () => {
  // Intialize our reporting framework by calling the lx.init method
  // https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#init
  const setup = await lx.init()

  // we extract the workspace factsheet types from the setup object and store
  // it in our factSheetTypes reactive variable
  // more info on vue ref api found here:
  // https://v3.vuejs.org/guide/reactivity-fundamentals.html#creating-standalone-reactive-values-as-refs
  factSheetTypes.value = Object.keys(setup.settings.dataModel.factSheets)
  // we initialize our grapQLResultIndex so that it preserves same factSheetType order of the facetResultIndex
  graphQLResultIndex.value = factSheetTypes.value.reduce((accumulator, factSheetType) => ({ ...accumulator, [factSheetType]: null }), {})

  // we map each factsheet type into a report facet that we will use
  // to extract the relevant statistics from our workspace
  // https://leanix.github.io/leanix-reporting/interfaces/lxr.reportfacetsconfig.html
  const facets = factSheetTypes.value
    .map((factSheetType, key) => ({
      key: key.toString(),
      fixedFactSheetType: factSheetType,
      attributes: ['completion { completion }'],
      facetFiltersChangedCallback: async filter => {
        graphQLResultIndex.value[factSheetType] = await fetchGraphQLData(factSheetType, filter)
      },
      // everytime this facet gets updated, e.g. when the user sets a filter
      // the callback method will be triggered and the corresponding factsheet type key
      // in the facetResultIndex object will be updated as well
      callback: factSheets => {
        const factSheetCount = factSheets.length
        const averageCompletion = ((factSheets
          .reduce((accumulator, { completion }) => { return accumulator + completion.completion }, 0) / factSheetCount) * 100).toFixed(2) + '%'
        const facetResult = {
          label: lx.translateFactSheetType(factSheetType, 'plural'),
          factSheetCount,
          averageCompletion
        }
        facetResultIndex.value[factSheetType] = facetResult
      }
    }))
  // finally we call the lx.ready method with our report configuration
  // https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#ready
  lx.ready({ facets })
}

// this method will call the LeanIX GraphQL API and fetch a list of factsheets of a given type
const fetchGraphQLData = async (factSheetType, filter) => {
  // Destructing assignment of filter object with alias for facets and fullTextSearch attributes
  const { facets: facetFilters, fullTextSearchTerm: fullTextSearch, directHits } = filter
  const mappedFilter = { facetFilters, fullTextSearch, ids: directHits.map(({ id }) => id) }
  const query = 'query($filter:FilterInput){allFactSheets(filter:$filter){edges{node{completion{completion}}}}}'
  const queryResult = await lx.executeGraphQL(query, { filter: mappedFilter })
    .then(({ allFactSheets }) => {
      const factSheets = allFactSheets.edges.map(({ node }) => node)
      const factSheetCount = factSheets.length
      const averageCompletion = ((factSheets.reduce((accumulator, { completion }) => { return accumulator + completion.completion }, 0) / factSheetCount) * 100).toFixed(2) + '%'
      return {
        label: lx.translateFactSheetType(factSheetType, 'plural'),
        factSheetCount,
        averageCompletion
      }
    })
  return queryResult
}

// this watcher will be triggered whenever our factSheetTypes array variable gets updated
// and will update the graphQLResultIndex for every factSheetType in that array
/*
watch(factSheetTypes, async factSheetTypes => {
  const factSheetTypeData = await Promise
    .all(factSheetTypes
      .map(factSheetType => fetchGraphQLData(factSheetType))
    )
  factSheetTypes
    .forEach((factSheetType, idx) => {
      graphQLResultIndex.value[factSheetType] = factSheetTypeData[idx]
    })
})
*/

// we call our report initialization method here...
initializeReport()
</script>

<template>
  <div class="container mx-auto h-screen">
    <div class="flex flex-wrap items-start justify-center -mx-8 ">
      <div class="flex flex-col items-center px-8">
        <span class="text-3xl font-bold uppercase py-4">Facets</span>
        <pre>{{facetResultIndex}}</pre>
      </div>
      <div class="flex flex-col items-center px-8">
        <span class="text-3xl font-bold uppercase py-4">GraphQL</span>
        <pre>{{graphQLResultIndex}}</pre>
      </div>
    </div>
  </div>
</template>