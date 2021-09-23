# TUTORIAL 05: transforming GraphQL data using Javascript Array methods

While developing a LeanIX custom report, the developer frequently needs to operate on the [data queried from his workspace](https://dev.leanix.net/docs/custom-report-querying-data) for different reasons such as transforming it into a proper format to render a table, filtering it according a certain criteria or simply compute a statistic. For this purpose, JavaScript provides a set of powerful [Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array#) methods ([map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map), [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) and [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)) that allow to implement such data transformations  in a very eloquent way. Moreover, and due to Javascript [functional programming](https://en.wikipedia.org/wiki/Functional_programming) features, those methods can be chained in sequence, thus allowing the creation of very powerful data processing blocks.

In this step-by-step tutorial, we'll create a [LeanIX](https://www.leanix.net/en/) custom report that demonstrates how to transform GraphQL data into a format suitable to render a table, filter it according a certain criteria, and compute an statistic. More specifically, we’ll fetch a list of workspace Applications, and display their names and completeness ratio as a table, filter it according a minimum completion threshold, and compute the average completeness ratio for all of them, as in the picture below:

![screenshot-01](https://i.imgur.com/GPSOuv0.png)

The complete source-code for this project can be found [here](https://github.com/fazendadosoftware/leanix-tutorials/tree/master/tutorial-05).


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

## Setup the project's source-code baseline

Now that we have all the project boilerplate code in place, it's time to setup our project's source code baseline. Start by editing the <code>script</code> section of our <code>src/App.vue</code> file and replace its content with the content below:

```html
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
  lx.ready()
}

const fetchGraphQLData = async () => {
  // to be implemented
}

const mapResponseToRows = () => {
  // to be implemented
}

const computeTableColumns = () => {
  // to be implemented
}

// we call our initializeReport method here...
initializeReport()
</script>
```

We also edit the <code>template</code> section of the <code>src/App.vue</code> file and add the following layout:
```html
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
```

You may start the development server now by running the following command:

```bash
npm run dev
```

When you run *npm run dev*, a local webserver is hosted on *localhost:3000* that allows connections via HTTPS. But since just a development SSL certificate is created the browser might show a warning that the connection is not secure. You could either allow connections to this host anyways, or create your own self-signed certificate: https://www.tonyerwin.com/2014/09/generating-self-signed-ssl-certificates.html#MacKeyChainAccess.

If you decide to add a security exception to your localhost, make sure you open a second browser tab and point it to https://localhost:3000. Once the security exception is added to your browser, reload the original url of your development server and open the development console. Your should see a screen similar to the one below:

![screenshot-03](https://i.imgur.com/OfIkQiN.png)

As you may have observed, our report is designed with a 3 column-layout that renders, from left to right, the raw data that is fetched from the workspace through a graphQL query, the transformed data, and the table view. Additionally, and below the three columns, you'll notice an extra blue box that will show the average completion ratio for our dataset.

### Querying the workspace data

The first step in our report is to fetch a list of applications in our workspace. As a side note, we are assuming that you are familiar with the [leanix-reporting](https://leanix.github.io/leanix-reporting/) API and know how to query your workspace data from a custom report. In case you need further information on this topic, we recommend that you have a brief look at this nice [tutorial](https://dev.leanix.net/docs/custom-report-querying-data) on the data querying topic.
Adjust the method <code>fetchGraphQLData</code> of your <code>src/App.vue</code> file according the example below:

```html
<script setup>
import { ref } from 'vue'

const response = ref(null)
const rows = ref([])
const columns = ref([])
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
  // to be implemented, does nothing for now...
}

const computeTableColumns = () => {
  // to be implemented
}

// we call our initializeReport method here...
initializeReport()
// and we call our fetchGraphQLData method as well...
fetchGraphQLData()
</script>
```

Bear in mind that the <code>fetchGraphQLData</code> method is triggered only once, when the report loads. Notice also that we store the results of the graphQL query in the <code>response</code> state variable, defined also earlier, and that this variable is shown in our report's leftmost column.

![screenshot-04](https://i.imgur.com/4AJhEzo.png)

If you take a closer look at the structure of the response received, you'll notice that it consists of a Javascript object containing a single attribute named *"allFactSheets"*. Looking further into it, we can see that it contains a single array sub-field named *"edges"* composed by multiple, single-attribute, *"node"* objects.

At a first sight, we can immediately feel that this response data structure does not map directly into an array of rows suitable to be displayed as a table. Therefore some data transformation is in order and, for that, we'll implement it in our <code>mapResponseToRows</code> method.

## Mapping the graphQL query response into rows
In our table we are interested in showing, for each application, the *name* and *completion* ratio as percentage.
Therefore we need to map the contents of our <code>response</code> variable into an array of objects - the *table rows*, each representing an application, and composed by the two aforementioned attributes - *name* and *completion*. For that, we take our state variable <code>response</code> that contains the results of the graphQL query made earlier, and destructure it by extracting the array *"allFactSheets.edges"*. Furthermore, we'll apply the [map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) operator to this *"allFactSheets.edges"* array in order to extract the *"node"* object from each *"edge"*.  We store the result of this operation in the <code>rows</code> state variable, defined earlier, so that it can be shown in our report's center green column.

```html
<!-- src/App.vue -->
<script setup>
const mapResponseToRows = () => {
  if (response.value === null) return
  // destructure the this.response state variable, extracting the allFactSheets.edges array
  rows.value = response.value.allFactSheets.edges // <- this is an Array
    // and map each edge into its node attribute
    .map(edge => edge.node) // <- this is the Array map operator applied to it
}
</script>
```

For additional intuition on the results of this operation, take a closer comparison look between the contents of the red and green columns of your report.

![screenshot-05](https://i.imgur.com/sj96fwE.png)

As we can see from the picture above, we notice that each *"node"* of our transformed array contains, in fact, the *"name"* and "completion" attributes. However the *"completion"* attribute is still an object and not a percentage representation of its value. So, in order to make it right, we need to slightly change our mapping operation according the example below:

```html
<!-- src/App.vue -->
<script setup>
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
}
</script>
```

If you take a look at the green column you'll realize that each array items contains now the application's *"name"*, *"completion"* percentage and the *"completionValue"* attributes. Altough we need only the *"name"* and *"completion"* attributes for our table, we'll use the *"completionValue"* attribute later for computing a statistic.

![screenshot-06](https://i.imgur.com/cjZu7JP.png)

Now that we have our table rows properly mapped, there is still one pending task before they can be rendered as a table. If you take a look into your <code>template</code> section of our <code>src/App.vue</code> file that we defined earlier, and locate the ```<table>``` tag on it, you'll notice that it is using the state variable <code>columns</code> for mapping labels to columns - in the header, and rows to columns - in the body.

The <code>columns</code> state variable was defined earlier as an empty array, and now we'll use it to store our table's column keys and labels. We'll use the [leanix-reporting](https://leanix.github.io/leanix-reporting/) [lx.translateField](https://leanix.github.io/leanix-reporting/classes/lxr.lxcustomreportlib.html#translatefield) method for mapping the application's attribute keys (*name* and *completion*) into a proper translated label to be shown in our table's header. For that, implement the <code>computeTableColumns</code> method as indicated below, not forgetting to call it at the end of the <code>mapResponseToRows</code>mapResponseToRows method.

```html
<!-- src/App.vue -->
<script setup>
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
  computeTableColumns() // <-- call the computeTableColumns method here!
}

const computeTableColumns = () => {
  const  columnKeys = ['name', 'completion']
  columns.value = columnKeys
    .map(key  => ({ key, label:  lx.translateField('Application', key) }))
}
</script>
```

You should be seeing now, in the third column of your report, the rendered table!

![screenshot-07](https://i.imgur.com/BICSI3L.png)

Now that we have finished our data mapping exercise, we'll proceed  to explore the filtering feature of Javascript arrays.

## Filtering data
In this simple exercise, we'll demonstrate how to use the [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) method of Javascript [Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array) for creating a version of our dataset according a certain filtering criteria.
Just as a side note, the LeanIX GraphQL API already provides a set of very powerful filtering features, which you should use whenever possible in your query. However, there are some scenarios in which the filtering criteria can only be defined after the data is queried and analyzed, either derived from a computed statistic or some other  complex factor. In those cases a client-side filtering implementation is required, and the [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) method will fit perfectly to those cases.
In our example, let's assume a simple scenario where we want to show in our table only the applications that have a completion ratio below 10%. This filter can be quickly implemented with a single line of code using the [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) method chained after the mapping operation of our <code>mapResponseToRows</code> method defined earlier.

```html
<!-- src/App.vue -->
<script setup>
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
}
</script>
```

If you have a look at the green and yellow columns, you'll notice that only those applications with a completion ratio below 10% are being shown.

![screenshot-08](https://i.imgur.com/thXYNBY.png)

Now that we have covered the filtering exercise, we'll proceed to the last part of our tutorial in which we'll compute a statistic of our dataset using the [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) method of Javascript [Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array).

## Compute average completion ratio
Having covered the [map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) and [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) operators of Javascript [Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array), we look now into the [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) operator. Reduce is a very powerful method that can be used to transform an array into either a single object with multiple attributes, a single number or another array.
We'll use this operator for computing the sum of the completion values of all our applications, which we'll after divide by the total number of applications in order to compute the average completion ratio of all our applications. Traditionally, this also can be done by traversing our *rows* array with a *for loop*, however the [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) operator provides us with a more compact and elegant way of doing it.
In your report, change the <code>mapResponseToRows</code> method according the example below. Take note that the computed average completion ratio of our dataset is stored in the <code>avgCompletion</code> state variable that is shown in the blue box in our report.

```html
<!-- src/App.vue -->
<script setup>
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
</script>
```

Looking into our report we can confirm that the average completion ratio is indeed being shown in the blue box of our report.

![screenshot-09](https://i.imgur.com/lRsfqbm.png)

## Next steps
The [map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map), [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) and [reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) Javascript [Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/) operators are very powerful tools that every serious Javascript developer should master and guard closely in their toolbox. They provide a very compact way of writing data processing blocks that allow to shape datasets into a specific structure, filter them according a certain criteria, and compute statistics. During this tutorial we have briefly illustrated their application to an implementation of a LeanIX custom report. However, we do strongly recommend that if you are not already familiar and at ease with those particular methods, or if you are new to Javascript programming, to read further about them as they will be certainly be of great help in your future custom-report implementations.

Good work and congratulations for completing this tutorial!