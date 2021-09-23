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
            <canvas v-show="obsoleteITComponents.length > 0" ref="barChart"/>
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

<script setup>
import { ref, watch, onMounted } from 'vue'
import '@leanix/reporting'
import moment from 'moment'
import Chart from 'chart.js/auto'
import pdfMake from 'pdfmake/build/pdfmake'

const reportSetup = ref(null)
const itComponents = ref([])
const obsoleteITComponents = ref([])
const columns = ref([
  { key: 'obsolescenceDate', label: 'Obsolescence Date' },
  { key: 'category', label: 'Category' },
  { key: 'name', label: 'IT Component' }
])
const dataUrl = ref(null)
const startDate = ref(moment().format('YYYY-MM-DD'))
const endDate = ref(moment().add(1, 'months').endOf('month').format('YYYY-MM-DD'))
const analysis = ref('')
const barChart = ref(null)

const initializeReport = async () => {
  reportSetup.value = await lx.init()
  lx.ready({})
}

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

// watcher that will trigger the computeObsolescencies method on every update of
// startDate, endDate and itComponents state variables
watch([startDate, endDate, itComponents], computeObsolescencies)

// we initialize our report here...
initializeReport()
// and trigger the fetchDataset method
fetchDataset()
</script>