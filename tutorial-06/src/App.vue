<script setup>
import { ref } from 'vue'
import '@leanix/reporting'
import pptxgen from 'pptxgenjs'

const fields = ref([
  { key: 'keyPartners', label: 'Key Partners', classes: 'col-start-1 col-span-2 row-start-1 row-span-2' },
  { key: 'keyActivities', label: 'Key Activities', classes: 'col-start-3 col-span-2 row-start-1 row-span-1' },
  { key: 'keyResources', label: 'Key Resources', classes: 'col-start-3 col-span-2 row-start-2 row-span-1' },
  { key: 'valuePropositions', label: 'Value Propositions', classes: 'col-start-5 col-span-2 row-start-1 row-span-2' },
  { key: 'customerRelationships', label: 'Customer Relationships', classes: 'col-start-7 col-span-2 row-span-1' },
  { key: 'channels', label: 'Channels', classes: 'col-start-7 col-span-2 row-span-1' },
  { key: 'customerSegments', label: 'Customer Segments', classes: 'col-start-9 col-span-2 row-start-1 row-span-2' },
  { key: 'costStructure', label: 'Cost Structure', classes: 'col-span-5 row-start-3 row-span-1' },
  { key: 'revenueStreams', label: 'Revenue Streams', classes: 'col-span-5 row-start-3 row-span-1' }
])

const businessModelCanvasContent = ref({})
const container = ref(null)

const initializeReport = async () => {
  await lx.init()
  lx.ready({})
}

const onFileChange = evt => {
  const files = evt.target.files || evt.dataTransfer.files
  if (!files.length) return
  const reader = new FileReader()
  reader.onload = e => { businessModelCanvasContent.value = JSON.parse(e.target.result) }
  reader.readAsText(files[0])
}

const saveFile = () => {
  const data = JSON.stringify(businessModelCanvasContent.value, null, 2)
  const blob = new  Blob([data], { type: 'text/plain' })
  const e = document.createEvent('MouseEvents')
  const a = document.createElement('a')
  a.download = 'businessModelCanvas.json'
  a.href = window.URL.createObjectURL(blob)
  a.dataset.downloadurl = ['text/json', a.download, a.href].join(':')
  e.initEvent('click', true, false, window, 0, 0, 0, 0, 0, false, false, false, false, 0, null)
  a.dispatchEvent(e)
}

const exportToPPT = () => {
  // get an handle to our Business Model Canvas container element
  const containerEl = container.value
  // get the origin coordinates - x0, y0, width and height of it
  const { x: x0, y: y0, width: containerWidth, height: containerHeight } = containerEl.getBoundingClientRect()

  // auxiliar method for normalizing an element geometry relatively
  // to our business model canvas container, in terms of percentage
  const getNormalizedElBbox = el  => {
    let { x, y, width, height } = el.getBoundingClientRect()
    const  bbox = {
      x: ((x - x0) / containerWidth) * 100,
      y: ((y - y0) / containerHeight) * 100,
      width: (width / containerWidth) * 100,
      height: (height / containerHeight) * 100,
    }
    // round the values of our bbox object attributes to decimal places
    // and append to them a '%' character, as required by the PptxGenJS API
    const normalizedBbox = Object.entries(bbox)
      .reduce((accumulator, [key, value]) => ({ ...accumulator, [key]:  value.toFixed(2) + '%'}), {})
    return normalizedBbox
  }

  // For each Business Model Canvas container field, marked with the directive 'field'
  const fields = Array.from(containerEl.querySelectorAll('[field]'))
    .map(fieldEl  => {
      // get the normalized geometry and shape attributes of its outer container
      const containerBbox = {
        ...getNormalizedElBbox(fieldEl),
        line: { line:  '000000', lineSize:  '1' }
      }
      // get an handle to the field label, marked with the 'field-label' directive
      const  labelEl = fieldEl.querySelectorAll('[field-label')[0]
      // extract its text content
      let { textContent: text = '' } = labelEl
      // get the normalized geometry and text attributes of its content
      const labelBbox = {
        ...getNormalizedElBbox(labelEl),
        textOpts: { autoFit:  true, fontSize:  7, bold:  true, align:  'left', valign:  'top' },
        text
      }

      // get an handle to the field content, marked with the 'field-content' directive
      const contentEl = fieldEl.querySelectorAll('[field-content')[0]
      // extract its value
      text = contentEl.value || ''
      // get the normalized geometry and text attributes of its content
      const contentBbox = {
        ...getNormalizedElBbox(contentEl),
        textOpts: { autoFit: true, fontSize: 7, align: 'left', valign: 'top' },
        text
      }
      // return an array representing the field's container, label and content geometries
      return [containerBbox, labelBbox, contentBbox]
    })

  // create a new presentation
  const pres = new pptxgen()
  // add a slide to the presentation
  const slide = pres.addSlide()
  // for each mapped field of our business model canvas
  fields.forEach(field  => {
    field
      // add a shape if the section corresponds to the field container
      // or a text if the section corresponds to the field's label or content
      .forEach(section => {
        const { x, y, width: w, height: h, text, line = {}, textOpts = {} } = section
        const { rect: shapeType } = pres.ShapeType
        const shapeOpts = {x, y, w, h, ...line, ...textOpts }
        typeof text === 'string'
          ? slide.addText(text, { shape:  shapeType, ...shapeOpts })
          : slide.addShape(shapeType, shapeOpts)
      })
  })
  // and finally save the presentation
  pres.writeFile('BusinessModelCanvas.pptx')
}

// we initialize our report here...
initializeReport()
</script>

<template>
  <div class="container mx-auto h-screen flex flex-col p-8">

    <!-- the Action Bar container -->
    <div class="mb-4 flex justify-end gap-1">

      <!-- the "Load" button -->
      <label>
        <span class="cursor-pointer inline-flex items-center px-2 py-1 border border-transparent text-xs leading-4 font-semibold tracking-wide rounded text-white bg-red-600 hover:bg-red-500 transition ease-in-out duration-150">
          Load
        </span>
        <input @change="onFileChange" type="file" class="hidden" accept=".json">
      </label>

      <!-- the "Save" button -->
      <span class="inline-flex rounded-md shadow-sm">
        <button @click="saveFile" type="button"
        class="inline-flex items-center px-2 py-1 border border-transparent text-xs leading-4 font-semibold tracking-wide rounded text-white bg-green-600 hover:bg-green-500 transition ease-in-out duration-150">
          Save
        </button>
      </span>

      <!-- the "Export to PPT" button -->
      <span class="inline-flex rounded-md shadow-sm">
        <button @click="exportToPPT" type="button"
          class="inline-flex items-center px-2 py-1 border border-transparent text-xs leading-4 font-semibold tracking-wide rounded text-white bg-indigo-600 hover:bg-indigo-500 transition ease-in-out duration-150">
          Export to PPT
        </button>
      </span>
    </div>

    <!-- the Business Model Canvas container -->
    <div
      class="grid h-full border-t border-r rounded border-gray-400 text-gray-800 text-sm font-semibold"
      ref="container">
      <!-- recursive template for the grid fields -->
        <div
          v-for="field in fields"
          :key="field.key"
          :field="field.key"
          :class="field.classes"
          class="border-l border-b border-gray-400 p-2 flex flex-col">
          <!-- the field label -->
          <div
            field-label
            class="px-1 text-base mb-1 text-gray-700 truncate">
            {{field.label}}
          </div>
          <!-- the field input textarea, editable by the user -->
          <textarea
            field-content
            v-model="businessModelCanvasContent[field.key]"
            class="text-sm tracking-wide bg-gray-100 hover:bg-gray-200 focus:bg-gray-200 transition-color duration-250 w-full flex-1 border border-dotted rounded p-2"/>
        </div>
    </div>
  </div>
</template>