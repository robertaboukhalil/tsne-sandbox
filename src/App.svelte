<script>
import { onMount } from "svelte";
import { Aioli } from "@biowasm/aioli";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------

let tSNE = new Aioli("bhtsne/latest");
let data = null;
let clusterIDs = [];
let clusterNames = [];
let clusterIDsUnique = [];
let UI = {
	busy: true
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
		UI.busy = false;
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
	UI.busy = true;
	await tSNE.exec(`-s 42 -n 500 /bhtsne/pollen2014.snd`, d => data = d);
	UI.busy = false;
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

	// Or if received occasional data update
	} else {
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

		// Plotly.react doesn't re-initialize the plot each time it's called
		Plotly.react(document.getElementById("scatter"), traces, {
			margin: { t: 0, b: 0, l: 0, r: 0 },
			hovermode: "closest",
			xaxis: {
				showticklabels: false,
				showgrid: true,
				showline: false,
				zeroline: false
			},
			yaxis: {
				showticklabels: false,
				showgrid: true,
				showline: false,
				zeroline: false
			},
			showlegend: true
		}, {
			displayModeBar: false
		});
	}
}


// -----------------------------------------------------------------------------
// HTML
// -----------------------------------------------------------------------------
</script>

<style>
#scatter {
	width: 500px;
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

	<div class="container"> <!-- mt-4 pt-5 -->
		<div class="row">
			<!-- Params -->
			<div class="col-md-4">
				<h4 class="mb-4">Parameters</h4>

				<input type="button" class="btn btn-primary btn-lg" value="Launch tSNE" on:click={run} disabled={UI.busy}>
			</div>

			<!-- Data Viz -->
			<div class="col-md-8">
				<h4 class="mb-4">tSNE Plot</h4>
				<div id="scatter"></div>
			</div>
		</div>
	</div>
</main>
