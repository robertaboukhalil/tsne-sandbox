<script>
import { onMount } from "svelte";
import { Aioli } from "@biowasm/aioli";
import Parameter from "./Parameter.svelte";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------

let tSNE = new Aioli("bhtsne/latest");
let busy = true;
let data = null;
let options = {
	step: 0.5,
	perplexity: 50,
	seed: 42,
	iterations: 300,
	frequency: 1,
	minError: 0.001,
	showError: false
};
let progress = {
	step: null,
	error: null,
	message: "Select parameters and click Launch to start"
};
let clusterIDs = [];
let clusterNames = [];
let clusterIDsUnique = [];
let errors = { x: [], y: [] };

// Constants
const axisOptions = {
	showticklabels: false,
	showgrid: true,
	showline: false,
	zeroline: false
};


// -----------------------------------------------------------------------------
// Reactive statements
// -----------------------------------------------------------------------------

$: plot(data);


// -----------------------------------------------------------------------------
// On page load
// -----------------------------------------------------------------------------

onMount(async () => {
	// Initialize tSNE WebAssembly module
	tSNE.init().then(d => {
		busy = false;
		console.log("Loaded");
	});

	// Enable jQuery tooltips
	jQuery("[data-toggle='popover']").popover();
});


// -----------------------------------------------------------------------------
// Launch tSNE
// -----------------------------------------------------------------------------

async function run()
{
	busy = true;
	progress.message = "Calculating...";

	// Reset previous data
	errors = { x: [], y: [] };

	// Launch tSNE analysis and provide callback function that saves intermediate results
	let params = `-e ${options.step} -r ${options.frequency} -p ${options.perplexity} -n ${options.iterations} -s ${options.seed}`;
	await tSNE.exec(`-d 2 ${params} /bhtsne/pollen2014.snd`, d => data = d);

	busy = false;
}

function plot(data)
{
	if(data == null || !tSNE.ready)
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
	if(progress.error != -1 && (
		Math.abs(data.error - progress.error)/progress.error < options.minError ||
		progress.step % 50 == 0
	))
		skip = true;

	// Update progress
	progress.message = "";
	progress.error = data.error;
	progress.step = data.iter;
	progress.n = data.N;
	// Update error values
	if(options.showError) {
		errors.x.push(data.iter);
		errors.y.push(data.error);
	}

	if(skip)
		return;

	// Extract X and Y coordinates (stores as [x1, y1, x2, y2, ...])
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
			// Don't show coordinates since they don't mean anything
			hoverinfo: "name",
			// Avoid having an ellipsis in the hover text
			hoverlabel: { namelength: -1 },
		})
	}

	// Add error trace as inlet
	let shapes = [];
	if(options.showError) {
		traces.push({
			...errors,
			name: "Error",
			xaxis: "x2",
			yaxis: "y2",
		});

		// Add border around the inlet
		shapes = [{
			type: "rect", xref: "x2", yref: "y2",
			x0: Math.min(...errors.x),
			y0: Math.min(...errors.y),
			x1: Math.max(...errors.x),
			y1: Math.max(...errors.y),
			fillcolor: "#0069d9", opacity: 0.1, line: { width: 0 }
		}];
	}

	// Plot tSNE
	// Plotly.react doesn't re-initialize the plot each time it's called
	Plotly.react(document.getElementById("scatter"), traces, {
		margin: { t: 0, b: 0, l: 0, r: 0 },
		hovermode: "closest",
		showlegend: true,
		xaxis: { ...axisOptions },
		yaxis: { ...axisOptions },
		xaxis2: { ...axisOptions, showgrid: false, domain: [0.7, 1], anchor: "x2" },
		yaxis2: { ...axisOptions, showgrid: false, domain: [0, 0.3], anchor: "y2" },
		shapes: shapes
	}, {
		displayModeBar: false
	});
}


// -----------------------------------------------------------------------------
// HTML
// -----------------------------------------------------------------------------
</script>

<style>
#scatter {
	width: 600px;
	height: 500px;
}
</style>

<nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
	<a class="navbar-brand" href="/">tSNE Sandbox</a>
	<div class="collapse navbar-collapse">
		<ul class="navbar-nav mr-auto"></ul>
		<ul class="navbar-nav">
			<li class="nav-item dropdown">
				<a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">Data</a>
				<div class="dropdown-menu dropdown-menu-right">
					<a target="_blank" class="dropdown-item" href="https://hemberg-lab.github.io/scRNA.seq.datasets/human/tissues/#pollen">Data Source</a>
					<a target="_blank" class="dropdown-item" href="https://www.nature.com/articles/nbt.2967">Paper describing data</a>
				</div>
			</li>

			<li class="nav-item dropdown">
				<a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">tSNE Resources</a>
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
</nav>

<main role="main">
	<div class="jumbotron mt-2 pb-1">
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
			<div class="col-md-3">
				<h4 class="mb-4">Parameters</h4>
				<Parameter label="Step Size" type="text" bind:value={options.step} disabled={busy} help="Parameter between <code>0</code> and <code>1</code> that determines the approximation level used by tSNE; lower numbers mean less approximations, therefore higher runtime." />
				<Parameter label="Perplexity" type="text" bind:value={options.perplexity} disabled={busy} help="Parameter between <code>5</code> and <code>50</code> that is an estimate of how many neighbor each data point has." />
				<Parameter label="Iterations" type="text" bind:value={options.iterations} disabled={busy} help="Stop algorithm after <code>{options.iterations}</code> iterations." />
				<Parameter label="Rnd Seed" type="text" bind:value={options.seed} disabled={busy} help="Seed for random number generator for reproducibility. Set to -1 to disable." />
				<Parameter label="Frequency" type="text" bind:value={options.frequency} disabled={busy} help="How often you'd like the plot to be updated. The lower the number, the more work your browser has to do." />
				<Parameter label="Min Error" type="text" bind:value={options.minError} disabled={busy} help="Minimum error change for plotting." />
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
			<div class="col-md-9">
				<h4 class="mb-4">
					tSNE Plot
					{#if progress.step != null}
					<small><small>
						Step {progress.step} / {options.iterations}
						&mdash; error = {Math.round(100000 * progress.error) / 100000}
					</small></small>
					{/if}
				</h4>
				<div id="scatter">
					{progress.message}
				</div>
			</div>
		</div>
	</div>
</main>
