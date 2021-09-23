# TUTORIAL 02: Querying workspace data from a custom report

Custom reports are a great way for analyzing and communicating Enterprise Architecture insights of your organization in an effective way.

In this step-by-step tutorial we create a simple [LeanIX](https://www.leanix.net/en/) custom report that demonstrates how to fetch data from the LeanIX workspace using the two methods provided by the LeanIX Reporting API: Facet Filters and GraphQL queries.

The complete source-code for this project can be found [here](https://github.com/fazendadosoftware/leanix-tutorials/tree/master/tutorial-02).

## Pre-requisites for this tutorial:

* [NodeJS LTS](https://nodejs.org/en/) installed in your computer.
* Javascript knowledge
* [Vue](https://vuejs.org/) framework knowledge


## Getting started

Initialize a new project by running the following command and answering the questionnaire. For this tutorial we will be using the vue template:

```bash
npm init lxr@latest
```

After this procedure, you should end up with the following project structure:

![screenshot-01](https://i.imgur.com/XZU3AEn.png)

## Adjust the report boilerplate source code

We need to make some modifications in our project's boilerplate code. We start by deleting the unnecessary files:
-  *src/assets/logo.png*
-  *src/components/HelloWorld.vue*

Then we add [TailwindCSS](https://tailwindcss.com/), a CSS framework that provide several utility classes that we use during our tutorial for styling it. For that we follow the official [installation guide](https://tailwindcss.com/docs/guides/vue-3-vite) and perform the following steps:

1. Install Tailwind and its peer-dependencies using npm:

```bash
npm install -D tailwindcss@latest postcss@latest autoprefixer@latest
```

2. Next, generate your tailwind.config.js and postcss.config.js files:

```bash
npx tailwindcss init -p
```

3. In your tailwind.config.js file, configure the purge option with the paths to all of your pages and components so Tailwind can tree-shake unused styles in production builds:

```javascript
// tailwind.config.js
module.exports = {
  purge: ['./index.html', './src/**/*.{vue,js,ts,jsx,tsx}'],
  darkMode: false, // or 'media' or 'class'
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  },
  plugins: [],
}
```

4. Additionally, ensure your CSS file is being imported in your ./src/main.js file:

```javascript
// src/main.js
import { createApp } from 'vue'
import App from './App.vue'
import 'tailwindcss/tailwind.css'

createApp(App).mount('#app')
```

5. Finally, adjust the *./src/App.vue* file and set the *template* and *script* tags as follows:
```html
<template>
  <!-- we'll use this template tag for declaring our custom report html -->
  <div>Hi from LeanIX Custom Report</div>
</template>

<script setup>
  // all the state variables and business logic will be declared here
</script>

```

You may now start the development server now by running the following command:
```bash
npm run dev
```
**Note!**

When you run *npm run dev*, a local webserver is hosted on *localhost:3000* that allows connections via HTTPS. But since just a development SSL certificate is created the browser might show a warning that the connection is not secure. You could either allow connections to this host anyways, or create your own self-signed certificate: https://www.tonyerwin.com/2014/09/generating-self-signed-ssl-certificates.html#MacKeyChainAccess.

If you decide to add a security exception to your localhost, make sure you open a second browser tab and point it to https://localhost:3000. Once the security exception is added to your browser, reload the original url of your development server and open the development console. Your should see a screen similar to the one below:

![screenshot-02](https://i.imgur.com/DU1cnK0.png)

Nothing very exciting happens here. Notice however that our report loads, and is showing the message we defined inside the *template* tag of the *App.vue* file...

## Fetching workspace data using Facet Filters
Facet filtering is one of the two ways provided by the [leanix-reporting api](https://leanix.github.io/leanix-reporting/) that can be used to fetch data from a *workspace* into a *custom report*. In this section we'll setup our report so that we can use facet filtering to *compute* the *factsheet count* and *average completion ratio* individually for each factsheet type defined in our workspace.

### Setting up the report configuration
We edit the <code>src/App.vue</code> file and declare in the <code>script</code> section the variables <code>factSheetTypes</code> and <code>facetResultIndex</code>, and the <code>initializeReport</code> method:

```html
<script setup>
import '@leanix/reporting'
import { ref, watch } from 'vue'

// reactive variable that will hold the workspace factsheet types
const factSheetTypes = ref([])
// reactive variable that stores some statistics for each factsheet type
const facetResultIndex = ref({})

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


  // we map each factsheet type into a report facet that we will use
  // to extract the relevant statistics from our workspace
  // https://leanix.github.io/leanix-reporting/interfaces/lxr.reportfacetsconfig.html
  const facets = factSheetTypes.value
    .map((factSheetType, key) => ({
      key: key.toString(),
      fixedFactSheetType: factSheetType,
      attributes: ['completion { completion }'],
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

// we call our report initialization method here...
initializeReport()
</script>

<template>
  <!-- our template consists of a simple container that will display our facetResultsIndex variable -->
  <div class="container mx-auto h-screen">
    <div class="flex flex-wrap items-start justify-center">
      <div class="flex flex-col items-center">
        <span class="text-3xl font-bold uppercase py-4">Facets</span>
        <pre>{{facetResultIndex}}</pre>
      </div>
    </div>
  </div>
</template>

```
The <code>factSheetTypes</code> array contains all all factsheet types in our workspace. We'll hold the result of our completion analysis, for each factsheet type, in the <code>facetResultIndex</code> object. Moreover, we configure our report using the <code>initializeReport</code> method. We define a facet for each factsheet type in our workspace. The complete documentation for the **leanix-reporting api** can be found [here](https://leanix.github.io/leanix-reporting/).

Your report should now show the <code>factSheetTypeIndex</code> displayed in a pretty format:

![screenshot-03](https://i.imgur.com/fTZVQ0H.png)

### Filtering and bookmarking
Filtering and bookmarking with Facet Filters is done out of the box, using the standard LeanIX Pathfinder controls and without requiring any modifications in the custom report source code.
Filtering is done via the navigation panel on the left, by selecting the **Filter** tab and any combination of facets displayed below.

![screenshot-04](https://i.imgur.com/ZtfdrJP.png)

Once you set a combination of *facet filters* in your report, you can **bookmark** them by clicking on the **"Save as"** button on the top right corner and fill out the form details. Your bookmark will then be listed as a new report under the **Reports** tab of the left navigation column of the LeanIX Pathfinder application.

![screenshot-05](https://i.imgur.com/T8XPbCc.png)

## Fetching workspace data using GraphQL queries
Another way to fetch data from a workspace into a custom report is by using the [lx.executeGraphQL](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#executegraphql) method provided by the [leanix-reporting api](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html). Unlike the Facet Filter method we've seen before, which allows only data to be read from your workspace, the [lx.executeGraphQL](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#executegraphql) method allows also to mutate the workspace data. However, the integration of this method with the standard filtering and bookmarking controls is not out of the box like the Facet Filters method seen before. Nevertheless, it can be done with a special technique in our custom report, as we'll see ahead.

### Setting up the report configuration
We edit once again the <code>script</code> section of our <code>src/App.vue</code> file and additionally declare the a variable <code>graphQLResultIndex</code>, a method <code>fetchGraphQLData</code> and a <code>watcher</code> for the variable <code>factSheetTypes</code> as follows:
```html
<script setup>
import '@leanix/reporting'
import { ref, watch } from 'vue'

const factSheetTypes = ref([])
const facetResultIndex = ref({})

// reactive variable that stores some statistics for each factsheet type fetch through the GraphQL API
const graphQLResultIndex = ref({})

const initializeReport = async () => { /* no changes..., leave it as it is */ }

// this method will call the LeanIX GraphQL API and fetch a list of factsheets of a given type
const fetchGraphQLData = async factSheetType => {
  const query = 'query($factSheetType:FactSheetType){allFactSheets(factSheetType:$factSheetType){edges{node{completion{completion}}}}}'
  const queryResult = await lx.executeGraphQL(query, { factSheetType })
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
watch(factSheetTypes, async factSheetTypes => {
  const factSheetTypeData = await Promise
    .all(factSheetTypes.map(factSheetType => fetchGraphQLData(factSheetType)))
  factSheetTypes
    .forEach((factSheetType, idx) => {
      graphQLResultIndex.value[factSheetType] = factSheetTypeData[idx]
    })
})

// we call our report initialization method here...
initializeReport()
</script>
```
We also adjust our <code>template</code> tag in <code>src/App.vue</code> so that a second column showing the contents of the <code>graphQLResultIndex</code> will be displayed:

```html
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

```
Our report now should look like this:
![screenshot-06](https://i.imgur.com/7UiDz3D.png)

As you can verify, the results provided by Facets and GraphQL are identical. However, if you try to apply a filter, you'll notice that only the Facets column changes. This happens because we are only triggering the <code>fetchGraphQLData</code> method whenever our <code>factSheetTypes</code> variables change, which only happens once the report loads. Therefore the GraphQL part of our report, as it is, does not react to any changes in the filters. However, the [leanix-reporting api](https://leanix.github.io/leanix-reporting/) provides a way of **listening to filter changes and trigger a callback** through the facet configuration property [facetFiltersChangedCallback](https://leanix.github.io/leanix-reporting/interfaces/lxr.reportfacetsconfig.html#facetfilterschangedcallback) . We'll adapt then our source code to make our graphQL queries reactive to changes in filters.

For that, we'll adapt the <code>initializeReport</code> method and set, for each facet, a [facetFiltersChangedCallback](https://leanix.github.io/leanix-reporting/interfaces/lxr.reportfacetsconfig.html#facetfilterschangedcallback) that will trigger the <code>fetchGraphQLData</code> method on each facet update, i.e. when the user sets a filter. Notice as well that we make a slight change in the <code>fetchGraphQLData</code> so to include in our query the facet filtering parameters passed by the callback. Finally note as well that we delete the watcher we previously set.

```html
<script setup>

const factSheetTypes = ref([])
const facetResultIndex = ref({})
const graphQLResultIndex = ref({})

const initializeReport = async () => {
  const setup = await lx.init()

  factSheetTypes.value = Object.keys(setup.settings.dataModel.factSheets)

  const facets = factSheetTypes.value
    .map((factSheetType, key) => ({
      key: key.toString(),
      fixedFactSheetType: factSheetType,
      attributes: ['completion { completion }'],
      // the callback for changes in the facet filter that updates the graphQLResultIndex variable
      facetFiltersChangedCallback: async filter => {
        graphQLResultIndex.value[factSheetType] = await fetchGraphQLData(factSheetType, filter)
      },
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

// we call our report initialization method here...
initializeReport()
</script>
```
So now everytime a filter is applied to a facet, you can see the filtered result appearing in both Facets and GraphQL columns.
![screenshot-07](https://i.imgur.com/xVykgyT.png)

Congratulations, you have finalized this report!