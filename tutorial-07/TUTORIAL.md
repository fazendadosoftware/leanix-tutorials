# TUTORIAL 07: Creating and exporting a custom PDF document from workspace data

Custom reports are a great way of creating specific views of your LeanIX workspace data. However, in a corporate environment, the user may need to document and communicate those views, analyses, and findings to other stakeholders in a format that follows a certain set of corporate documentation guidelines and policies. Thus, the ability to export information from LeanIX into a popular document format such as PDF, for example, that follows a certain template that meets the corporate guidelines, may save some time to the user.
In this step-by-step tutorial, we'll create a [LeanIX](https://www.leanix.net/en/) custom report project that demonstrates how to export its content into a PDF document that follows a pre-defined template. More specifically, we'll generate an obsolescence report in which the user sets a start and end date for the analysis, and gets both a chart and a list of all IT Components in his workspace that transition into the "End of Life" lifecycle phase during the specified date range.

![screenshot-01](https://i.imgur.com/eZtC7GW.png)

The complete source-code for this project can be found [here](https://github.com/fazendadosoftware/leanix-tutorials/tree/master/tutorial-07).


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

5. Install the following dependencies as well:

```bash
npm install moment chart.js pdfmake
```

6. Finally, adjust the *./src/App.vue* file and set the *template* and *script* tags as follows:
```html
<template>
  <!-- we'll use this template tag for declaring our custom report html -->
  <div>Hi from LeanIX Custom Report</div>
</template>

<script setup>
import { ref, watch } from 'vue'
import '@leanix/reporting'
import moment from 'moment'
import Chart from 'chart.js/auto'
import pdfMake from 'pdfmake/build/pdfmake'

// state variable that will hold our workspace setup
const reportSetup = ref(null)
// state variable that will hold our dataset
const itComponents = ref([])
// state variable that will hold the computed obsolete it components
const obsoleteITComponents = ref([])
// variable that define the columns shown in our it component table
const columns = ref([
  { key: 'obsolescenceDate', label: 'Obsolescence Date' },
  { key: 'category', label: 'Category' },
  { key: 'name', label: 'IT Component' }
])
const dataUrl = ref(null)
// variable that will hold the startDate value
const startDate = ref(moment().format('YYYY-MM-DD'))
// variable that will hold the endDate value
const endDate = ref(moment().add(1, 'months').endOf('month').format('YYYY-MM-DD'))
const analysis = ref('')
// variable that will reference the canvas barChart element
const barChart = ref(null)

const initializeReport = async () => {
  reportSetup.value = await lx.init()
  lx.ready({})
}

const fetchDataset = () => {
  // to be implemented
}

const computeObsolescencies = () => {
  // to be implemented
}

const generateBarChart = () => {
  // to be implemented
}

const generatePDF = () => {
  // to be implemented
}

// watcher that will trigger the computeObsolescencies method on every update of
// startDate, endDate and itComponents state variables
watch([startDate, endDate, itComponents], computeObsolescencies)

// we initialize our report here...
initializeReport()
// and trigger the fetchDataset method
fetchDataset()
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

Now that we have the base structure for our Custom Report, let's proceed into its design!

## Custom Report Design

We'll organize our custom report in two columns. On the leftmost  column we'll place two datepicker inputs, through which the use can set the start and end dates of the analysis. Also, we'll include on top of our left column an action button that will trigger the generation of the PDF document. Moreover, and also on the left column, we'll have three rows. On the first row we'll display a Bar Chart showing, by category - Hardware, Software or Service, the number of IT Components that get obsolete during the selected period. On the second row we'll display a scrollable table listing all those same IT Components. Finally, the third row will consist of a text area in which the user can write a couple of words on the obsolescence situation.

![screenshot-03](https://svgshare.com/i/PdL.svg)

As a first step, we'll edit the <code>template</code> section of our <code>src/App.vue</code> file and add the basic template of our report as follows:

```html
<template>
  <div class="h-screen flex p-4 gap-4">
    <!-- Left Column Container -->
    <div class="w-1/2 overflow-auto py-4">
      <!-- Top ActionBar Container -->
      <div class="flex justify-end mb-4 gap-2">
        <div class="flex text-sm gap-4 py-2 justify-center items-center">
          <!-- StartDate DatePicker -->
          <div>
            <label for="startDate" class="font-medium">Start Date</label>
            <input
              v-model="startDate"
              type="date"
              id="startDate"
              required
              class="border rounded text-xs">
          </div>
          <!-- /StartDate DatePicker -->
          <!-- EndDate DatePicker -->
          <div>
            <label for="endDate" class="font-medium">End Date</label>
            <input
              v-model="endDate"
              type="date"
              id="endDate"
              required
              class="border rounded text-xs">
          </div>
          <!-- /EndDate DatePicker -->
        </div>
        <div class="flex-1"></div>
        <!-- GeneratePDF Button -->
        <button
          @click="generatePDF"
          type="button"
          class="inline-flex gap-1 items-center px-2 py-1 border border-transparent text-xs leading-4 font-medium rounded text-white bg-green-600 hover:bg-green-500 transition ease-in-out duration-150">
          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"  xmlns="http://www.w3.org/2000/svg">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 21h10a2 2 0 002-2V9.414a1 1 0 00-.293-.707l-5.414-5.414A1 1 0 0012.586 3H7a2 2 0 00-2 2v14a2 2 0 002 2z"></path>
          </svg>
          Generate PDF
        </button>
         <!-- /GeneratePDF Button -->
      </div>
      <!-- /Top ActionBar Container -->
      <div class="flex flex-col gap-6">
        <!-- Chart Container -->
        <div>
          <div class="text-center font-medium mb-2">Obsolescences per IT Component Category</div>
          <div class="h-56 relative border rounded shadow">
            <canvas v-show="obsoleteITComponents.length > 0" ref="barChart"></canvas>
            <div v-show="obsoleteITComponents.length === 0" class="italic absolute" style="top: 50%; left: 50%; transform: translate(-50%)">
              Nothing to display...
            </div>
          </div>
        </div>
        <!-- /Chart Container -->
        <!-- List Container -->
        <div>
          <div class="text-center font-medium mb-2" x-text="'IT Component Obsolescences (' + obsoleteITComponents.length + ')'"></div>
          <div class="h-40 overflow-auto border shadow rounded relative">
            <table x-show="obsoleteITComponents.length" class="table-auto w-full bg-white border-separate border border-t-0 border-grey" :style="'border-spacing: 0'">
              <thead>
                <tr>
                  <th
                    v-for="(column, idx) in columns"
                    :key="idx"
                    class="border-r border-b border-grey sticky top-0 bg-gray-100 font-normal p-1">
                    {{column.label}}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="(row, idx) in obsoleteITComponents"
                  :key="idx">
                  <td
                    v-for="column in columns"
                    :key="column.key"
                    class="p-1 border-b border-r">
                    {{row[column.key].label || row[column.key]}}
                  </td>
                </tr>
              </tbody>
            </table>
            <div v-if="!obsoleteITComponents.length" class="italic absolute" style="top: 50%; left: 50%; transform: translate(-50%)">
              No IT Component obsolescence is scheduled for the selected period...
            </div>
          </div>
        </div>
        <!-- /List Container -->
        <!-- TextArea Container -->
        <div>
          <div class="text-center font-medium mb-2">Analysis</div>
            <textarea
              placeholder="Write your comment here..."
              v-model="analysis"
              rows="6"
              class="bg-white w-full shadow border rounded p-2 text-xs">
            </textarea>
        </div>
        <!-- /TextArea Container -->
      </div>
    </div>
    <!-- /Left Column Container -->
    <!-- Right Column Container -->
    <div class="bg-green-100 w-1/2 border">
      <template v-if="dataUrl">
        <!-- PDF Viewer Container -->
        <embed :src="dataUrl" type="application/pdf" width="100%" height="100%">
        <!-- /PDF Viewer Container -->
      </template>
    </div>
    <!-- /Right Column Container -->
  </div>
</template>
```

Your report should look like this now:

![screenshot-04](https://i.imgur.com/LsWkaIX.png)

## Business Logic
The business logic for our report will consist of four main functionalities:
- fetching data
- computing obsolete it components
- generate the bar chart
- generate the PDF document

In this chapter, we'll cover the implementation of each, individually.

### Fetching Data
The first method we'll cover is the <code>fetchDataset</code>. With this method we fetch information on all IT Components existing in our workspace. More specifically, we are interested in getting three attributes of each IT Component: name, category and lifecycle. We'll use the [lx.executeGraphQL](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#executegraphql) method to fetch our dataset and store it using the *itComponents* state variable. Also, recall that this method is triggered only once when the report loads. Proceed by copying and pasting the <code>fetchDataset</code> method given below into to the placeholder located in the <code>script</code> section of your <code>src/App.vue</code> file.

```html
<script setup>
const fetchDataset = async () => {
  const query = `{
    allFactSheets(factSheetType: ITComponent) {
      edges {
        node {
          id
          name
          ... on ITComponent {
            category
            lifecycle {
              lifecyclePhase:asString
              phases {
                phase
                startDate
              }
            }
          }
        }
      }
    }
  }`
  itComponents.value = await lx.executeGraphQL(query)
    .then(data => data.allFactSheets.edges.map(({ node }) => node))
}
</script>
```

### Computing Obsolescencies
Once we have fetched the information on all IT Components, it is time to compute the obsolescency state for each during the user-defined date range. By obsolete, we mean that the IT Component transitions into the "End of Life" lifecycle phase within the date range specified by the user. Therefore, our <code>computeObsolescencies</code> method will traverse the <code>itComponents</code> array in order to filter in obsolete IT Components. The result of this filtering will be stored on the <code>obsoleteITComponent</code> state variable. Moreover, after this filtering operation is completed, it triggers the <code>generateBarChart</code> method which we'll cover ahead. Also, keep in mind that the <code>computeObsolescencies</code> method is triggered by the watcher we have defined at the end of the <code>script</code> section of our <code>src/App.vue</code> file. As before, copy and past the code snippet below into the appropriate placeholder located in the <code>script</code> section of your <code>src/App.vue</code> file.

```html
<script setup>
const computeObsolescencies = () => {
  const itComponentViewModel = reportSetup.value.settings.viewModel.factSheets
    .find(({ type }) => type === 'ITComponent')
  
  const categoryFieldMetaDataIndex = itComponentViewModel.fieldMetaData.category.values

  obsoleteITComponents.value = itComponents.value
    .reduce((accumulator, itComponent) => {
      let { lifecycle, category } = itComponent
      if (lifecycle === null) return accumulator
      const { phases } = lifecycle
      let { startDate: obsolescenceDate } = phases.find(({ phase }) => phase === 'endOfLife') || {}
      if (!obsolescenceDate) return accumulator
      obsolescenceDate = moment(obsolescenceDate, 'YYYY-MM-DD')
      if (obsolescenceDate && obsolescenceDate.isBetween(startDate.value, endDate.value)) {
        const label = category === null
          ? 'Not defined'
          : lx.translateFieldValue('ITComponent', 'category', category)
        const { bgColor, color } = categoryFieldMetaDataIndex[category] || {}
        category = { key: category, label, bgColor, color }
        accumulator.push({ ...itComponent, category, obsolescenceDate })
      }
      return accumulator
    }, [])
    .sort(({ obsolescenceDate: A }, { obsolescenceDate: B }) => A.unix() < B.unix() ? -1 : A.unix() > B.unix() ? 1 : 0)
    .map(itComponent => ({
      ...itComponent,
      obsolescenceDate: itComponent.obsolescenceDate.format('YYYY-MM-DD')
    }))

  generateBarChart ()
}
</script>
```

Now, if you observe your report, you should see the list of obsolete IT Components being shown on the second row of the leftmost column. If nothing shows up, try tweaking the start and end dates of your analysis in order to capture some obsolescence event!
![screenshot-05](https://i.imgur.com/mBV2Exw.png)
Good work, let's carry on and generate the chart!

### Generating the Bar Chart
Having computed the list of IT Components that get obsolete during the date range specified by the user, we want to display that information on a *bar chart* that shows the obsolescency frequency of each IT Component category during that period. Therefore we'll create an index of obsolete IT Component categories, based on our <code>obsoleteITComponent</code> state variable, and use it for generating a [Bar Chart](https://www.chartjs.org/docs/latest/charts/bar.html) using the popular [Chart.js](https://www.chartjs.org/) library. The chart will be rendered in the *canvas* element, referenced as *"barChart"*, that we created before in our html template.

Again, copy and past the code snippet below into the appropriate placeholder located in the <code>script</code> section of your <code>src/App.vue</code> file.

```html
<script setup>
const generateBarChart = () => {
  const categoryIndex = obsoleteITComponents.value
    .reduce((accumulator, itComponent) => {
      const { category } = itComponent
      if (!accumulator[category.key]) accumulator[category.key] = { ...category, items: [] }
      accumulator[category.key].items.push(itComponent)
      return accumulator
    }, {})

  const options = {
    responsive: true,
    maintainAspectRatio: false,
    devicePixelRatio: 2,
    events: [],
    legend: false,
    plugins: {
      legend: false
    },
    scales: {
      yAxes: [
        {
          ticks: {
            beginAtZero: true,
            stepSize: 1
          }
        }
      ]
    }
  }

  const keys = Object.keys(categoryIndex)

  const data = {
    labels: keys.map(key => categoryIndex[key].label),
    datasets: [
      {
        backgroundColor: keys.map(key => categoryIndex[key].bgColor),
        data: keys.map(key => categoryIndex[key].items.length)
      }
    ]
  }

  new Chart(barChart.value, { type: 'bar', data, options })
}
</script>
```

Our custom report should be displaying now the chart! If not, try tweaking on your start and end dates until an IT Component of your workspace gets obsolete during in that period.
![](https://i.imgur.com/IcLvF5f.png)

Having included the chart in our report, we'll proceed in our tutorial by generating the PDF document out of it!

### Generating the PDF Document
We want our exported PDF document to follow a certain style and structure, and to include the bar chart, the list, and the text content that we have in our custom report. For that, we'll make use of the versatile [PDFmake](http://pdfmake.org) library. This library is well [documented](https://pdfmake.github.io/docs/) and offers an interesting [playground](http://pdfmake.org/playground.html) that can be used to explore the variety of available features. In our case, we'll define the PDF document structure in the *dd* variable of the *generatePDF* method. We'll tweak our document's design by setting the desired page size, orientation, margins, images, fonts, and default styles and content. We highly recommend consulting the [PDFMake](https://pdfmake.github.io/docs/0.1/) documentation for further details on its API and features.
Copy and past the <code>generatePDF</code> method given below into the <code>script</code> section our <code>src/App.vue</code> file:

```html
<script setup>
const generatePDF = () => {
  const dd = {
    pageSize: 'A4',
    pageOrientation: 'portrait',
    pageMargins: 1 * 72, // 2.54 cm - 1 in converted to points, pageMargins is specified in points...
    images: { barChart: barChart.value.toDataURL() },
    defaultStyle: { font: 'Roboto', fontSize: 9 },
    styles: {
      titleBlock: { alignment: 'center', margin: [0, 0, 0, 40] },
      header: { fontSize: 18, bold: true, alignment: 'center' },
      subheader: { fontSize: 15, bold: true, alignment: 'center', margin: [0, 0, 0, 20] },
      section: { margin: [0, 0, 0, 50] }
    },
    content: [
      {
        style: 'titleBlock',
        stack: [
          { text: 'Obsolescence Report', style: 'header' },
          {
            fontSize: 10,
            margin: [0, 15],
            italics: true,
            text: [
              { text: startDate.value, bold: true },
              ' to ',
              { text: endDate.value, bold: true }
            ]
          }
        ]
    },
    {
      style: 'section',
      stack: [
        { text: 'Obsolescences per IT Component Category', style: 'subheader' },
        obsoleteITComponents.value.length
          ? { image: 'barChart', fit: [420, 200] }
          : { text: 'No obsolete it components during this period...', italics: true, alignment: 'center' }
      ]
    },
    {
      style: 'section',
      stack: [
        { text: `IT Component Obsolescences`, style: 'subheader' },
        !obsoleteITComponents.value.length
          ? { text: 'No obsolete it components during this period...', italics: true, alignment: 'center' }
          : {
        table: {
          widths: ['auto', 'auto', '*'],
          headerRows: 1,
          body: [
            columns.value.map(({ label }) => label),
            ...obsoleteITComponents.value
              .reduce((accumulator, itComponent) => {
                const row = columns.value.map(({ key }) => itComponent[key].label || itComponent[key])
                accumulator.push(row)
                return accumulator
              }, [])
            ]
          }
        }
      ]
    },
    {
      style: 'section',
      stack: [
      { text: 'Analysis', style: 'subheader' },
      analysis.value
        ? { text: analysis.value, alignment: 'justify '}
        : { text: 'no comments', alignment: 'center', italics: 'true' }
      ]
    }
    ]
  }
  const fonts = {
    Roboto: {
      normal: 'https://cdnjs.cloudflare.com/ajax/libs/pdfmake/0.1.66/fonts/Roboto/Roboto-Regular.ttf',
      bold: 'https://cdnjs.cloudflare.com/ajax/libs/pdfmake/0.1.66/fonts/Roboto/Roboto-Medium.ttf',
      italics: 'https://cdnjs.cloudflare.com/ajax/libs/pdfmake/0.1.66/fonts/Roboto/Roboto-Italic.ttf',
      bolditalics: 'https://cdnjs.cloudflare.com/ajax/libs/pdfmake/0.1.66/fonts/Roboto/Roboto-MediumItalic.ttf'
    }
  }
  const pdfDocGenerator = pdfMake.createPdf(dd, undefined, fonts)
  pdfDocGenerator.getDataUrl(_dataUrl => { dataUrl.value = _dataUrl })
}
</script>
```

If if you click on the *Generate PDF* button you should be able to see on the right column a preview of the exported document!
![](https://i.imgur.com/MFgphd3.png)

## Conclusions and next steps
In this tutorial we have covered a way of exporting a LeanIX Custom Report into a PDF document that follows a specific design and layout. This is a very interesting technique that can be used for generating normalized documentation, straight out of your LeanIX workspace, which meets your organization's policies, designs or guidelines. As an interesting follow-up exercise for the reader, we recommend to adjust the design and structure of the PDF template provided in this tutorial to your organization requirements. Thank you and good work!