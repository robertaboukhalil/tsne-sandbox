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
	message: ""
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
	if(options.showError)
		traces.push({
			...errors,
			name: "Error",
			xaxis: "x2",
			yaxis: "y2",
		});

	// Plot tSNE
	// Plotly.react doesn't re-initialize the plot each time it's called
	Plotly.react(document.getElementById("scatter"), traces, {
		margin: { t: 0, b: 0, l: 0, r: 0 },
		hovermode: "closest",
		xaxis: { ...axisOptions },
		yaxis: { ...axisOptions },
		xaxis2: { ...axisOptions, showgrid: false, domain: [0.7, 1], anchor: "x2" },
		yaxis2: { ...axisOptions, showgrid: false, domain: [0, 0.3], anchor: "y2" },
		showlegend: true
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
	<div class="collapse navbar-collapse" id="navbarsExampleDefault">
		<ul class="navbar-nav mr-auto"></ul>
		<ul class="navbar-nav">
			<li class="nav-item active">
				<a class="nav-link" href="https://github.com/robertaboukhalil/tsne-sandbox">Code</a>
			</li>
		</ul>
	</div>
</nav>

<main role="main">
	<div class="jumbotron mt-2 pb-1">
		<div class="container">
			<p class="lead">Abc</p>
		</div>
	</div>

	<!-- mt-4 pt-5 -->
	<div class="container">
		<div class="row">
			<!-- Params -->
			<div class="col-md-4">
				<h4 class="mb-4">Parameters</h4>
				<Parameter label="Step Size" type="text" bind:value={options.step} disabled={busy} help="TODO" />
				<Parameter label="Perplexity" type="text" bind:value={options.perplexity} disabled={busy} help="TODO" />
				<Parameter label="Iterations" type="text" bind:value={options.iterations} disabled={busy} help="Stop algorithm after {options.iterations} iterations" />
				<Parameter label="Rnd Seed" type="text" bind:value={options.seed} disabled={busy} help="Seed for random number generator for reproducibility. Set to -1 to disable." />
				<Parameter label="Frequency" type="text" bind:value={options.frequency} disabled={busy} help="How often you'd like the plot to be updated. The lower the number, the more work your computer has to do." />
				<Parameter label="Min Error" type="text" bind:value={options.minError} disabled={busy} help="Minimum error change for plotting" />
				<Parameter label="Plot Error" type="checkbox" bind:value={options.showError} disabled={busy} help="Enable to see how the error changes over time" />
				<hr />

				<button type="button" class="btn btn-primary btn-lg" on:click={run} disabled={busy}>
					{#if busy}
						<span class="spinner-grow spinner-grow-sm mb-1" role="status" aria-hidden="true"></span>
					{/if}
					Launch tSNE
				</button>
			</div>

			<!-- Data Viz -->
			<div class="col-md-8">
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
