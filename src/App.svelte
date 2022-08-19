<script>
import { onMount } from "svelte";
import Aioli from "@biowasm/aioli";
import Parameter from "./Parameter.svelte";

// Bootstrap
import popper from "popper.js";
import jQuery from "jquery";
import Bootstrap from "bootstrap";
import "bootstrap/dist/css/bootstrap.min.css";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------

let CLI;

// Status
let busy = true;
let progress = {
	step: 0,			// tSNE iteration we have data for
	error: null,		// tSNE error for iteration `progress.step`
	plotted: 0,			// tSNE iteration being plotted; progress.plotted always < progress.step
};

// Labels
let clusterIDs = [];
let clusterNames = [];
let clusterIDsUnique = [];

// Plots
let dataPlots = [];					// Array of all plots, keyed on iteration number
let dataErrorPlots = [];		// Array of all error plots, keyed on iteration number

// WebWorker data 
let data = null;					// Latest data received from WebWorker (a single trace)
let dataErrors = { x: [], y: [] };	// Latest historical errors

// Config
let options = {
	step: 0.5,
	perplexity: 50,
	seed: 42,
	iterations: 300,
	frequency: 1,
	minError: 0.001,
	showError: true
};
const axisOptions = {
	showticklabels: false,
	showgrid: true,
	showline: false,
	zeroline: false
};
const plotOptions = {
	margin: { t: 0, b: 0, l: 0, r: 0 },
	hovermode: "closest",
	showlegend: true,
	xaxis: { ...axisOptions, range: [-1.05, 1.05]},
	yaxis: { ...axisOptions, range: [-1.05, 1.05]},
	xaxis2: { ...axisOptions, showgrid: false, domain: [0.75, 1], anchor: "x2" },
	yaxis2: { ...axisOptions, showgrid: false, domain: [0, 0.25], anchor: "y2" },
	// Background for inset plot
	shapes: [{
		type: "rect", xref: "x", yref: "y",
		x0: 0.5, x1: 1.05,
		y0: -0.5, y1: -1.05,
		fillcolor: "#000000", opacity: 0.05, line: { width: 0 }
	}]
};


// -----------------------------------------------------------------------------
// Reactive statements
// -----------------------------------------------------------------------------

$: process(data);


// -----------------------------------------------------------------------------
// On page load
// -----------------------------------------------------------------------------

onMount(async () => {
	// Initialize tSNE WebAssembly module
	CLI = await new Aioli("bhtsne/2016.08.22", {
		callback: d => {
			data = d;
		}
	});
	busy = false;
	console.log("Loaded");

	// Enable jQuery tooltips
	jQuery("[data-toggle='popover']").popover();
});


// -----------------------------------------------------------------------------
// Launch tSNE
// -----------------------------------------------------------------------------

async function run()
{
	// Reset
	busy = true;
	dataErrors = { x: [], y: [] };
	progress.plotted = 0;

	// Launch tSNE analysis and provide callback function that saves intermediate results
	let params = `-e ${options.step} -r ${options.frequency} -p ${options.perplexity} -n ${options.iterations} -s ${options.seed}`;
	await CLI.exec(`bhtsne -d 2 ${params} /bhtsne/pollen2014.snd`);
}


// -----------------------------------------------------------------------------
// Process data we receive from WebWorker
// -----------------------------------------------------------------------------

function process(data)
{
	if(data == null)
		return;

	// If got row names from WebWorker
	if(data.row_names != null) {
		clusterIDs = data.row_names.map(d => d.split(":")[0]);
		clusterNames = data.row_names.map(d => d.split(":")[1]);
		clusterIDsUnique = [...new Set(clusterIDs)];
		return;
	}

	// Otherwise, we received a data update
	// Do we want to replot the data? i.e. has the error changed enough?
	let skip = false;
	if(Math.abs(data.error - progress.error)/progress.error < options.minError)
		skip = true;

	// Update progress
	progress.error = data.error;
	progress.step = data.iter;
	progress.n = data.N;
	// Update error values
	dataErrors.x.push(data.iter);
	dataErrors.y.push(data.error);

	// Start plotting if we have enough data
	if(progress.plotted == 0 && progress.step > Math.min(50, Math.round(options.iterations / 2)))
		plot();

	if(skip)
		return;

	console.log(`Step ${data.iter} - error = ${data.error}`);

	// Extract X and Y coordinates (stores as [x1, y1, x2, y2, ...]).
	// Also convert the x/y coordinates from a typed array to a regular array.
	let traces = [];
	let x = data.data.filter((d, k) => { return k % 2 == 0 }),
		y = data.data.filter((d, k) => { return k % 2 == 1 });

	// Generate traces
	for(let clusterID of clusterIDsUnique)
	{
		let rowIDs = clusterIDs
			.map((v, k) => v == clusterID ? k : null )
			.filter(d => d != null);

		traces.push({
			name: clusterNames[rowIDs[0]],
			x: x.filter((v, k) => rowIDs.includes(k)),
			y: y.filter((v, k) => rowIDs.includes(k)),
			// Displays line graph by default
			mode: "markers",
			type: "scattergl",
			// Don't show coordinates since they don't mean anything
			hoverinfo: "name",
			// Avoid having an ellipsis in the hover text
			hoverlabel: { namelength: -1 },
		});
	}

	// Add error trace as inlet
	let errorTraces = [];
	if(options.showError)
	{
		errorTraces.push({
			x: [...dataErrors.x],
			y: [...dataErrors.y],
			name: "Error", xaxis: "x2", yaxis: "y2"
		});
	}

	dataPlots[progress.step] = traces;
	dataErrorPlots[progress.step] = errorTraces;
}


// -----------------------------------------------------------------------------
// Plot tSNE iterations. To make the plotting smooth, we don't plot data as we
// receive it. Although that seems to work in Firefox and Safari, it appears
// very choppy in Chrome. Separating the data analysis from visualization seems
// to address that issue and ensures smooth plotting in all browsers.
// -----------------------------------------------------------------------------

function plotNext(timeout=30)
{
	if(timeout == 0)
		plot();
	else
		setTimeout(plot, timeout);
}

function plot()
{
	console.log(`plotted=${progress.plotted}; total=${dataPlots.length}`);

	// Stop when we've plotted the last tSNE iteration
	if(progress.plotted >= options.iterations) {
		busy = false;
		return;
	}

	// Stay a few steps behind so the calculation can catch up to the plotting
	if(progress.plotted > (dataPlots.length - 10) && dataPlots.length < (options.iterations - 10)) {
		plotNext();
		return;
	}

	// If we don't have data on the next iteration (i.e. % error changed not large enough),
	// keep going without timeout (plot stays as is)
	progress.plotted++;
	if(dataPlots[progress.plotted] == null) {
		plotNext(0);
		return;
	}

	// Normalize points such that the median is at (0, 0) with range up to -1 to 1.
	// This makes the visualization more stable instead of the central cluster flailing around.
	function normalizeToCenter(points) {
		let xValues = points.map(d => Array.from(d.x)).flat();
		let yValues = points.map(d => Array.from(d.y)).flat();
		let xCenter = median(xValues);
		let yCenter = median(yValues);
		let xMaxDistanceFromCenter = Math.max(...xValues.map(x => Math.abs(x - xCenter)));
		let yMaxDistanceFromCenter = Math.max(...yValues.map(y => Math.abs(y - yCenter)));
		let normalizedPoints = points.map(group => ({
				...group,
				x: group.x.map(val => (val - xCenter) / xMaxDistanceFromCenter),
				y: group.y.map(val => (val - yCenter) / yMaxDistanceFromCenter)
		}));
		return normalizedPoints;
	}

	// Plot tSNE iteration. Note that Plotly.react doesn't re-initialize the plot each time it's called
	Plotly.react(
		document.getElementById("scatter"),
		[...normalizeToCenter(dataPlots[progress.plotted]), ...dataErrorPlots[progress.plotted]],
		plotOptions,
		{ displayModeBar: false }
	);

	// Plot next iteration after slight delay
	plotNext();
}

// -----------------------------------------------------------------------------
// Utilities
// -----------------------------------------------------------------------------
function median(values) {
	let sorted = values.sort((a, b) => a - b);
	let mid = Math.floor(sorted.length / 2);
	return values.length % 2 !== 0 ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2;
};

// -----------------------------------------------------------------------------
// HTML
// -----------------------------------------------------------------------------
</script>

<svelte:head>
	<script src="https://cdn.plot.ly/plotly-1.54.0.min.js" type="text/javascript"></script>
</svelte:head>

<style>
#scatter {
	width: 350px;
	height: 300px;
}

@media (min-width: 700px) {
	#scatter {
		width: 600px;
		height: 500px;
	}
}
</style>

<nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
	<div class="container">
	<a class="navbar-brand" href="/">tSNE Sandbox</a>
	<button class="navbar-toggler collapsed" type="button" data-toggle="collapse" data-target="#navbar" aria-controls="navbar" aria-expanded="false" aria-label="Toggle navigation">
		<span class="navbar-toggler-icon"></span>
	</button>

	<div id="navbar" class="collapse navbar-collapse">
		<ul class="navbar-nav mr-auto"></ul>
		<ul class="navbar-nav">
			<li class="nav-item dropdown">
				<button class="btn btn-link nav-link dropdown-toggle" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">Data</button>
				<div class="dropdown-menu dropdown-menu-right">
					<a target="_blank" class="dropdown-item" href="https://hemberg-lab.github.io/scRNA.seq.datasets/human/tissues/#pollen">Data Source</a>
					<a target="_blank" class="dropdown-item" href="https://www.nature.com/articles/nbt.2967">Paper describing data</a>
				</div>
			</li>

			<li class="nav-item dropdown">
				<button class="btn btn-link nav-link dropdown-toggle" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">Resources</button>
				<div class="dropdown-menu dropdown-menu-right">
					<a target="_blank" class="dropdown-item" href="https://distill.pub/2016/misread-tsne/">How to Use tSNE Effectively</a>
					<a target="_blank" class="dropdown-item" href="https://www.oreilly.com/learning/an-illustrated-introduction-to-the-t-sne-algorithm">An illustrated introduction to tSNE</a>
					<a target="_blank" class="dropdown-item" href="https://lvdmaaten.github.io/tsne/">Frequently Asked Questions about tSNE</a>
					<a target="_blank" class="dropdown-item" href="https://lvdmaaten.github.io/publications/papers/JMLR_2008.pdf">Original tSNE paper</a>
					<a target="_blank" class="dropdown-item" href="https://en.wikipedia.org/wiki/T-distributed_stochastic_neighbor_embedding">tSNE Wiki page</a>
				</div>
			</li>

			<li class="nav-item active">
				<a class="nav-link" href="https://github.com/robertaboukhalil/tsne-sandbox">Code</a>
			</li>
		</ul>
	</div>
	</div>
</nav>

<main>
	<div class="jumbotron mt-5 mt-md-2 pb-1">
		<div class="container">
			<p class="lead">tSNE is an algorithm for visualizing high-dimensional data.</p>
			<p>
				In this example, our dataset consists of <code>302</code> points that live in <code>122</code> dimensions!
				Since humans aren't that good at seeing in <code>122</code> dimensions, we use tSNE to reduce the data to 2 dimensions and plot it below.
				tSNE tries to make the 2D representation preserve the relative distance between points.
			</p>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- Params -->
			<div class="col-12 col-md-3">
				<h4 class="mb-4">Parameters</h4>
				<Parameter label="Step Size" type="text" on:launch={run} bind:value={options.step} disabled={busy} help="Parameter between <code>0</code> and <code>1</code> that determines the approximation level used by tSNE; lower numbers mean less approximations, therefore higher runtime." />
				<Parameter label="Perplexity" type="text" on:launch={run} bind:value={options.perplexity} disabled={busy} help="Parameter between <code>5</code> and <code>50</code> that is an estimate of how many neighbor each data point has." />
				<Parameter label="Iterations" type="text" on:launch={run} bind:value={options.iterations} disabled={busy} help="Stop algorithm after <code>{options.iterations}</code> iterations." />
				<Parameter label="Rnd Seed" type="text" on:launch={run} bind:value={options.seed} disabled={busy} help="Seed for random number generator for reproducibility. Set to -1 to disable." />
				<Parameter label="Frequency" type="text" on:launch={run} bind:value={options.frequency} disabled={busy} help="How often you'd like the plot to be updated. The lower the number, the more work your browser has to do." />
				<Parameter label="Min Error" type="text" on:launch={run} bind:value={options.minError} disabled={busy} help="Minimum error change for plotting." />
				<Parameter label="Plot Error" type="checkbox" bind:value={options.showError} disabled={busy} help="Enable this setting to visualize how the error changes over time." />
				<hr />

				<button type="button" class="btn btn-primary btn-lg" on:click={run} disabled={busy}>
					Launch tSNE
					{#if busy}
						<span class="spinner-grow spinner-grow-sm mb-1" role="status" aria-hidden="true"></span>
					{/if}
				</button>
			</div>

			<!-- Data Viz -->
			<div class="col-12 col-md-9 mt-4 mt-md-0">
				<h4 class="mb-4">
					tSNE Plot
					{#if progress.plotted > 0}
					<small><small>
						Step {progress.plotted} / {options.iterations}
						&mdash; error = {Math.round(10000 * dataErrors.y[progress.plotted - 2]) / 10000}
					</small></small>
					{/if}
				</h4>

				{#if busy && progress.plotted == 0}
					<div class="d-flex justify-content-center">
						<div class="text-primary spinner-border mt-0" role="status">
							<span class="sr-only">Loading...</span>
						</div>
						<p class="lead ml-2">
							{#if progress.step > 0}
								Initializing tSNE...
							{:else}
								Loading...
							{/if}
						</p>
					</div>
				{/if}
				<div id="scatter"></div>
			</div>
		</div>
	</div>
</main>
