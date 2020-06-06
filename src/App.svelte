<script>
import { onMount } from "svelte";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------



// -----------------------------------------------------------------------------
// Reactive statements
// -----------------------------------------------------------------------------



// -----------------------------------------------------------------------------
// On page load
// -----------------------------------------------------------------------------

onMount(async () => {
	// Enable jQuery tooltips
	jQuery("[data-toggle='popover']").popover();
});


// -----------------------------------------------------------------------------
// HTML
// -----------------------------------------------------------------------------
</script>

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
	<div class="jumbotron mt-4 pb-3">
		<div class="container">
			<p class="lead">Quickly generate data quality reports for FASTQ files. Your data never leaves the browser.</p>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- Params -->
			<div class="col-md-4">
				<h4 class="mb-4">Parameters</h4>

				<h6>Choose FASTQ files to analyze</h6>
			</div>

			<!-- Data Viz -->
			<div class="col-md-8">
				<h4 class="mb-4">tSNE Plot</h4>
			</div>
		</div>
	</div>
</main>
