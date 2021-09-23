# TUTORIAL 04: implementing a matrix-layout custom report

Custom reports are a great way for analyzing and communicating Enterprise Architecture insights of your organization in an effective way.

In this step-by-step tutorial we create a [LeanIX](https://www.leanix.net/en/) custom report that demonstrates how to design a matrix-layout data visualization. More specifically, we'll display a matrix of applications vs their lifecycle phase start dates, if defined, as in the picture below.

![screenshot-01](https://i.imgur.com/DknXfoz.png)

The complete source-code for this project can be found [here](https://github.com/fazendadosoftware/leanix-tutorials/tree/master/tutorial-04).


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
We want to implement a matrix-layout report which will show a list of applications that exist in our workspace versus the start date of each corresponding lifecycle phase, if defined.

We'll divide our report implementation into two parts: querying the workspace data and visualizing the results.

### Querying the workspace data
In order to build our application lifecycle matrix, we'll fetch from our workspace a list of applications using the [facet filter data fetching interface](https://leanix.github.io/leanix-reporting/interfaces/lxr.reportfacetsconfig.html) provided by the [leanix-reporting api](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html). We would like also to store the <code>baseUrl</code> of our workspace in a state variable, so that we can navigate later into the applications by clicking on them. Finally, we also will want to extract the color code for the different Application lifecycle phases, which we'll derive from the workspace view model contained in the [reportSetup](https://leanix.github.io/leanix-reporting/interfaces/lxr.reportsetup.html) object and store it into the state variable <code>applicationLifecyclePhases</code>.
In order to do so, we'll edit our <code>src/App.vue</code> file and include in the <code>script</code> section three state variables: <code>baseUrl</code>, <code>applications</code> and <code>applicationLifecyclePhases</code>. We'll also add two methods: <code>initializeReport</code> and the helper method <code>translateLifecycleField</code> that we'll use for translating our application lifecycle fields in our template:

```html
<script setup>
import { ref } from 'vue'
import '@leanix/reporting'

// variable that will store our workspace's baseUrl
const baseUrl = ref(null)
// variable for storing the workspace's applications
const applications = ref([])
// variable for storing the workspace's application lifecycle phases view model
const applicationLifecyclePhases = ref({})

// the report initialization method
const initializeReport = async () => {
  const reportSetup = await lx.init()

  // we extract our application lifecycle phases color code here...
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

// we call our report initialization method here
initializeReport()
</script>
```

In order to take a peek at the application list that is being fetch from our workspace, we'll change the <code>template</code> section of our <code>src/App.vue</code> file as follows:

```html
<template>
  <div class="container mx-auto h-screen">
    <div
      v-for="application in applications"
      :key="application.id"
      class="flex mb-2 text-xs">
      <div class="w-1/3 font-bold mr-6">{{application.name}}</div>
      <div>{{translateLifecycleField(application)}}</div>
    </div>
  </div>
</template>
```

Your report should now be showing a list of application names and the current lifecycle phase, as in the picture below:

![screenshot-03](https://i.imgur.com/soxzmJ2.png)

Notice that this list is filterable, and it gets updated as soon as you set a new filtering criteria in the report facet.

Altough the list looks interesting, it is still yet very different from the matrix-view report we aim to implement. Our matrix report will have one column for the application name, and one to each lifecycle phase defined in our workspace. Our next job in this tutorial will be to create a method that maps our application list into a set of matrix rows to be rendered in our report.

Edit the <code>script</code> section of our <code>src/Application.vue</code> file and add three new state variables - <code>headerRow</code>, <code>rows</code> and <code>gridStyle</code>, two methods: the <code>computeRows</code> method and the <code>applicationClickEvtHandler</code> method, and a <code>watcher</code> for the <code>applications</code> state variable as follows:

```html
<script setup>
import { ref, watch } from 'vue'
import '@leanix/reporting'

const baseUrl = ref(null)
const applications = ref([])
const applicationLifecyclePhases = ref({})

// the state variable that will hold the matrix top row containing the column headers/names
const headerRow = ref([])
// the state variable that will hold our application rows
const rows = ref([])
// will dynamically set the number of columns in our grid, based on the lifecycle phases fetched from workspace
const gridStyle = ref('')

// the report initialization method
const initializeReport = async () => { /* nothing changes here */ }

// our method for computing our matrix rows
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

// our application click event handler which will open a factsheet preview on
// when the user clicks on the application name cell
const applicationClickEvtHandler = ({ type = null, key: id }) => {
  if (type === null) return
  const url = `${baseUrl.value}/factsheet/${type}/${id}`
  lx.openLink(url)
}

// we set a watcher here for calling the computeRows method on every update of the applications variable
watch(applications, computeRows)

initializeReport()
</script>
```

We also adjust the <code>template</code> section of our <code>src/Application.vue</code> file for showing our matrix as follows:

```html
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
```

Now just run <code>npm run dev</code> and enjoy your matrix report!

![screenshot-04](https://i.imgur.com/fMsb97O.png)

Congratulations, you have finalized this tutorial!
