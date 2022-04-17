<script>
  import { onDestroy } from 'svelte';
  import InputDataTable from "./InputDataTable.svelte";
  import Table from "./Table.svelte";
  import Tabs from "./Tabs.svelte";
  import Plot from "./Plot.svelte";
  import { data, dataText, resetOptions, parameters, parameterTypes, parameterOptions,
           numParetoPoints, fullyDefined, nonTargetOutputs, xAxisOutput,
           yAxisOutput} from './stores.js';

  import * as XLSX from 'xlsx';

  // start webworker for python calculations
  const pyodideWorker = new Worker('webworker.js');
  
  onDestroy(() => pyodideWorker.terminate());

  let selectedTab = 0;
  let plotData = null;
  let paretoData = null;
  let plotPromise = null;

  function handleFile(event) {
    let e = event.detail;
    console.log(e.type);
    let files, f;
    if (e.type === "drop") {
      e.stopPropagation();
      e.preventDefault();
      files = e.dataTransfer.files;
      f = files[0];
    } else { 
      files = e.target.files;
      f = files[0];
    }
    let reader = new FileReader();
    reader.onload = function(e) {
      let data = new Uint8Array(e.target.result);
      let workbook = XLSX.read(data, {type: 'array'});

      let first_sheet_name = workbook.SheetNames[0];

      let worksheet = workbook.Sheets[first_sheet_name];

      let new_data = XLSX.utils.sheet_to_json(worksheet, {header: 1});

      $parameters = new_data[0];
      $dataText = new_data.slice(1);

      // reset the options the user has chosen when loading a new file
      resetOptions();

    } 
    reader.readAsArrayBuffer(f);
  };

  function getParetoData(){
    return new Promise((resolve, reject) => {

      function handleWorkerMessage(e){
        if (e.data === "pyodide_not_available") {
          // pyodide didn't load properly
          reject('Pyodide not available for calculations. Try refreshing page or using a different browser.');
        } else {
          resolve(e.data)
        }
      }
      pyodideWorker.onmessage = handleWorkerMessage;
      pyodideWorker.postMessage([$data, $parameters, $parameterTypes, $parameterOptions, $numParetoPoints]);

    });
  }

  function handleGenerateParetoData () {
    plotPromise = getParetoData()
      .then(data => {
        plotData = data.plot;
        paretoData = data.data;
      });
  }

  function handleSaveParetoData(e){
    if(paretoData){
      let workbook = XLSX.utils.book_new();
      let sheet = XLSX.utils.aoa_to_sheet([paretoData.headers, ...paretoData.data]);
      XLSX.utils.book_append_sheet(workbook, sheet, 'Pareto Data');
      XLSX.writeFile(workbook, 'pareto_data.csv');
    }
  }

  $: if(!$fullyDefined) {
    plotData = null;
    paretoData = null;
    plotPromise = null;
  }

  $: $parameterOptions.forEach((option, i) => {
      option.x_axis = (i === $xAxisOutput) ? true : false;
      option.y_axis = (i === $yAxisOutput) ? true : false;
    });

</script>

<style>
  .hidden {
    display: none;
  }
</style>

<Tabs tabs={['Upload DICOM', 'DICOM Tags', 'DICOM Images', 'Instructions', 'About']} bind:selectedTab>
  <div class:hidden={selectedTab !== 0}>
    <InputDataTable on:loadfile={handleFile}/>
  </div>
  <div class:hidden={selectedTab !== 1}>
    {#if $parameterOptions.length > 0}
      {#if $nonTargetOutputs.length >= 3}
        <label>
          x-axis output:
          <select bind:value={$xAxisOutput}>
            {#each $nonTargetOutputs as output}
              <option value={output.index} selected={$xAxisOutput === output.index}>{output.text}</option>
            {/each}
          </select>
        </label>
        <label> 
          y-axis output:
          <select bind:value={$yAxisOutput}>
            {#each $nonTargetOutputs as output}
              <option value={output.index} selected={$yAxisOutput === output.index}>{output.text}</option>
            {/each}
          </select>
        </label>
      {:else}
        Select at least one input and at least 2 non-target outputs.
      {/if}
    {:else}
      No data loaded.
    {/if}
    {#if $fullyDefined}
      <label>
        Number of Pareto points: <input type=number bind:value={$numParetoPoints} min=3 max=100>
      </label>
      <button on:click={handleGenerateParetoData}>Generate Pareto Data</button>
      {#if plotPromise}
        {#await plotPromise}
          <span>Updating plot...</span>
        {:catch error}
          <span>{error}</span>
        {/await}
      {/if}
      {#if plotData}
        <Plot plotData={plotData}></Plot>
      {/if}
    {/if}
  </div>
  <div class:hidden={selectedTab !== 2}>
      {#if paretoData}
        <button on:click={handleSaveParetoData}>Export as csv...</button>
        <Table headers={paretoData.headers} data={paretoData.data}></Table>
      {:else}
        Generate Pareto plot to obtain Pareto data.
      {/if}
  </div>
  <div class:hidden={selectedTab !== 3}>
    <h3>Instructions</h3>
    <p>Upload your DICOM file. This should allow the user to view/edit both the tags and any image_array data contained. The DICOM data can be saved back to your machine at any point.
    </p>
    <h3>Data Saftey</h3>
    <p>Even so this site is being accessed in the browser all python code is being run locally on your machine. This means the DICOM data is not sent back to any external server and remains locally.
    However, the Author does not except any liability of data saftey or implications of modifying DICOM data when using this site and this responsbility remains soley with the user.
    </p>
  </div>
  <div class:hidden={selectedTab !== 4}>
    <p>This tool was developed by <a href="https://github.com/SamPIngram">Sam Ingram</a> - Clinical Scientist at The Christie NHS Foundation Trust. Report issues or bugs to the
      author at <a href="mailto:samuel.ingram1@nhs.net">samuel.ingram1@nhs.net</a>. </p>
    <p>DICOM reading is done using <a href="https://pyodide.org/en/stable/">Pyodide</a>
      to compile a version of <a href="https://pydicom.github.io/">Pydicom</a>.
      <a href="https://svelte.dev/">Svelte</a> is used to implement the user interface, 
      <a href="https://plotly.com/javascript/">plotly</a> is used for plotting, and 
      <a href="https://github.com/SheetJS/sheetjs">sheetjs</a> is used for spreadsheet 
      file I/O.
    </p>
    <p>This is an open source project and the source code is available on this project's 
      <a href="https://github.com/SamPIngram/ReadDicomAnywhere">GitHub Page</a>.
    </p>
  </div>
</Tabs>