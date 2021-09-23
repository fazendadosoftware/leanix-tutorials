# TUTORIAL 03: Visualizing workspace data

Custom reports are a great way for analyzing and communicating Enterprise Architecture insights of your organization in an effective way.

In this step-by-step tutorial we create a simple [LeanIX](https://www.leanix.net/en/) custom report that demonstrates how to integrate a third party library for visualizing workspace data. More specifically, we'll display on a half-pie chart the average completion ratio for a specific factsheet type, configurable by the user, as in the picture below.

![screenshot-01](https://i.imgur.com/OzJpbIS.png)

The complete source-code for this project can be found [here](https://github.com/fazendadosoftware/leanix-tutorials/tree/master/tutorial-03).

  

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

## Report design
In our report we want to analyze the average completion ratio for a specific factsheet type, configurable by the user. In order to do so, we'll use the [lx.openFormModal](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#openformmodal) method of the [leanix-reporting api](https://leanix.github.io/leanix-reporting/). For the rest of this tutorial, we'll divide our report implementation into three parts: setting up the report configuration workflow, querying workspace data and visualizing the results.

### Setting up the report configuration
We want to allow the user to select the factsheet type to be analyzed by the report. For that, we will be using the standard **"Settings"** button enabled by the [showConfigure](https://leanix.github.io/leanix-reporting/interfaces/lxr.reportconfiguration.html#menuactions) flag in the report [configuration](https://leanix.github.io/leanix-reporting/interfaces/lxr.reportconfiguration.html). This **"Settings"** button will trigger a callback that opens a modal containing a dropdown list of all factsheet types available in the workspace.

We edit the <code>src/App.vue</code> file and declare in the script section the following two state variables: <code>factSheetTypes</code> and <code>selectedFactSheetType</code>, and two methods: <code>initializeReport</code> and <code>openReportConfigurationModal</code>:

```html
<script setup>
import '@leanix/reporting'
import { ref, watch } from 'vue'

// reactive variable that will hold the workspace factsheet types
const factSheetTypes = ref([])
// reactive variable will hold the selected factsheet type
const selectedFactSheetType = ref(null)

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

// we call our report initialization method here...
initializeReport()
</script>

<template>
  <div class="container mx-auto h-screen">
    <div>{{selectedFactSheetType}}</div>
  </div>
</template>
```
Notice now that the **Settings** button appears on the top-right corner of the report, and that when clicking on it the report configuration modal shows up. Confirm that when selecting a different factsheet type, the placeholder element, situated on the top-left corner of the report, gets updated accordingly.

![screenshot-03](https://i.imgur.com/OCjfkSz.png)

### Querying the workspace data
Having the report configuration workflow in place, it's time to implement the data querying mechanism for fetching data from the workspace. For each selected factsheet type, we want to compute an *factsheet type average completion ratio* defined as the sum of all factsheets completion ratios divided by the number of factsheets. We'll add to the <code>script</code> section of our <code>src/App.vue</code> file a state variable <code>averageCompletion</code>, a method <code>fetchGraphQLData</code> and a <code>watcher</code> for the <code>selectedFactSheetType</code> state variable:
```html
<script setup>
import '@leanix/reporting'
import { ref, watch } from 'vue'

const factSheetTypes = ref([])
const selectedFactSheetType = ref(null)

// reactive variable that will hold the computed averageCompletion statistic
const averageCompletion = ref(null)

const initializeReport = async () => { /* nothing changes here... */ }

const openReportConfigurationModal = async () => { /* nothing changes here... */ }

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

// watcher that will trigger the fetchGraphQLData on every selectedFactSheetType update
watch(selectedFactSheetType, fetchGraphQLData)

// we call our report initialization method here...
initializeReport()
</script>
```

We'll also adjust the <code>template</code> section of our <code>src/App.vue</code> file as follows: 

```html
<template>
  <div class="container mx-auto h-screen">
    <div>{{selectedFactSheetType}} avg completion = {{(averageCompletion * 100).toFixed(0)}}%</div>
  </div>
</template>

```
Launching our report now, and switching between factsheet types, verify that the average completion percentage, shown on the top-left corner of the report, gets updated accordingly.

![screenshot-04](https://i.imgur.com/Gp9S2Nd.png)

Now that we have the data querying mechanism for our report in place, lets proceed to the data visualization part!

### Visualizing the results
For the last part of this tutorial, we'll use the [Chart.JS](https://www.chartjs.org/) to display the average completion ratio as an *half-pie* chart.

We'll start by installing the following dependencies to our project:

```bash
npm install chart.js tinygradient
```

We start by editing the <code>script</code> section of our <code>src/App.vue</code> file and add the *Chart.js* and *tinygradient* dependencies to our code. We declare as well the <code>chart</code> and <code>chartCanvas</code> variables and the <code>updateChart</code> method. As we want to change the chart color according the completeness ratio, we build a linear color gradient using the [tinygradient](https://github.com/mistic100/tinygradient) library. In our example, we'll be using a single dataset composed by two data points with the values of the average completeness ratio (corresponding to the colored bar of the chart) and the 1-complement value of this average (corresponding to the faded-gray bar). More details on the ChartJS api can be found on the [documentation](https://www.chartjs.org/docs/latest/).
Finally, we add also a <code>watcher</code> for the <code>averageCompletion</code> variable which will trigger the <code>updateChart</code> method on every update.

```html
<script setup>
import '@leanix/reporting'
import { ref, watch, computed } from 'vue'
// Chart.js dependency
import Chart from 'chart.js/auto'
// tinygradient dependency
import tinygradient from 'tinygradient'

const factSheetTypes = ref([])
const selectedFactSheetType = ref(null)
const averageCompletion = ref(null)

// holder variable for our chart (non-reactive)
let chart

// reactive holder for our canvas element
const chartCanvas = ref(null)

const initializeReport = async () => { /* nothing changes here */ }

const openReportConfigurationModal = async () => { /* nothing changes here */ }

const fetchGraphQLData = async () => { /* nothing changes here */ }

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

watch(selectedFactSheetType, fetchGraphQLData)
// watcher that will trigger the updateChart method on every averageCompletion variable update
watch(averageCompletion, updateChart)

initializeReport()
</script>
```

We edit as well the <code>template</code> section of our <code>src/App.vue</code> file and add a div container element for holding the chart canvas, title, subtitle and legend, as below:
```html
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
```

Now just run <code>npm run dev</code> to launch the development server again and observe your report chart updating while you switch between factsheet types in the Settings menu.

![screenshot-05](https://i.imgur.com/OzJpbIS.png)

Congratulations, you have finalized this tutorial!