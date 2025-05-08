<script>
	// CORE IMPORTS
	import { setContext, onMount } from "svelte";
	import { getMotion } from "./utils.js";
	import { themes } from "./config.js";
	import ONSHeader from "./layout/ONSHeader.svelte";
	import Footer from "./layout/Footer.svelte";
	import Header from "./layout/Header.svelte";
	import Section from "./layout/Section.svelte";
	import Media from "./layout/Media.svelte";
	import Scroller from "./layout/Scroller.svelte";
	import Filler from "./layout/Filler.svelte";
	import Divider from "./layout/Divider.svelte";

	
	// IMPORTS
	import bbox from "@turf/bbox";
	import { getData, setColors, getTopo, getBreaks, getColor } from "./utils.js";
	import { colors, units } from "./config.js";
	import { ScatterChart, LineChart, BarChart, ColumnChart, } from "@onsvisual/svelte-charts";

	// CORE CONFIG (COLOUR THEMES)
	// Set theme globally (options are 'light', 'dark' or 'lightblue')
	let theme = "light";
	setContext("theme", theme);
	setColors(themes, theme);

	// CONFIG FOR SCROLLER COMPONENTS
	// Config
	const threshold = 0.65;
	// State
	let animation = getMotion(); // Set animation preference depending on browser preference
	let id = {}; // Object to hold visible section IDs of Scroller components
	let idPrev = {}; // Object to keep track of previous IDs, to compare for changes
	onMount(() => {
		idPrev = {...id};
	});

	// DEMO-SPECIFIC CONFIG
	// Constants
	const datasets = ["region", "district"];
	const topojson = "./data/geo_lad2021.json";
	const mapstyle = "https://bothness.github.io/ons-basemaps/data/style-omt.json";
	const mapbounds = {
		uk: [
			[-9, 49 ],
			[ 2, 61 ]
		]
	};

	// Data
	let data = {district: {}, region: {}};
	let metadata = {district: {}, region: {}};
	let geojson;

	// Element bindings
	let map = null; // Bound to mapbox 'map' instance once initialised

	// State
	let hovered; // Hovered district (chart or map)
	let selected; // Selected district (chart or map)
	$: region = selected && metadata.district.lookup ? metadata.district.lookup[selected].parent : null; // Gets region code for 'selected'
	$: chartHighlighted = metadata.district.array && region ? metadata.district.array.filter(d => d.parent == region).map(d => d.code) : []; // Array of district codes in 'region'
	let mapHighlighted = []; // Highlighted district (map only)
	let xKey = "area"; // xKey for scatter chart
	let yKey = null; // yKey for scatter chart
	let zKey = null; // zKey (color) for scatter chart
	let rKey = null; // rKey (radius) for scatter chart
	let mapKey = "density"; // Key for data to be displayed on map
	let explore = false; // Allows chart/map interactivity to be toggled on/off

	// FUNCTIONS (INCL. SCROLLER ACTIONS)

	// Functions for chart and map on:select and on:hover events
	function doSelect(e) {
		console.log(e);
		selected = e.detail.id;
		if (e.detail.feature) fitById(selected); // Fit map if select event comes from map
	}
	function doHover(e) {
		hovered = e.detail.id;
	}

	// Functions for map component
	function fitBounds(bounds) {
		if (map) {
			map.fitBounds(bounds, {animate: animation, padding: 30});
		}
	}
	function fitById(id) {
		if (geojson && id) {
			let feature = geojson.features.find(d => d.properties.AREACD == id);
			let bounds = bbox(feature.geometry);
			fitBounds(bounds);
		}
	}

	// Actions for Scroller components
	const actions = {
		map: { // Actions for <Scroller/> with id="map"
			map01: () => { // Action for <section/> with data-id="map01"
				fitBounds(mapbounds.uk);
				mapKey = "density";
				mapHighlighted = [];
				explore = false;
			},
			map02: () => {
				fitBounds(mapbounds.uk);
				mapKey = "age_med";
				mapHighlighted = [];
				explore = false;
			},
			map03: () => {
				let hl = [...data.district.indicators].sort((a, b) => b.age_med - a.age_med)[0];
				fitById(hl.code);
				mapKey = "age_med";
				mapHighlighted = [hl.code];
				explore = false;
			},
			map04: () => {
				fitBounds(mapbounds.uk);
				mapKey = "age_med";
				mapHighlighted = [];
				explore = true;
			}
		},
		chart: {
			chart01: () => {
				xKey = "area";
				yKey = null;
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart02: () => {
				xKey = "area";
				yKey = null;
				zKey = null;
				rKey = "pop";
				explore = false;
			},
			chart03: () => {
				xKey = "area";
				yKey = "density";
				zKey = null;
				rKey = "pop";
				explore = false;
			},
			chart04: () => {
				xKey = "area";
				yKey = "density";
				zKey = "parent_name";
				rKey = "pop";
				explore = false;
			},
			chart05: () => {
				xKey = "area";
				yKey = "density";
				zKey = null;
				rKey = "pop";
				explore = true;
			}
		}
	};

	// Code to run Scroller actions when new caption IDs come into view
	function runActions(codes = []) {
		codes.forEach(code => {
			if (id[code] != idPrev[code]) {
				if (actions[code][id[code]]) {
					actions[code][id[code]]();
				}
				idPrev[code] = id[code];
			}
		});
	}
	$: id && runActions(Object.keys(actions)); // Run above code when 'id' object changes

	// INITIALISATION CODE
	datasets.forEach(geo => {
		getData(`./data/data_${geo}.csv`)
		.then(arr => {
			let meta = arr.map(d => ({
				code: d.code,
				name: d.name,
				parent: d.parent ? d.parent : null
			}));
			let lookup = {};
			meta.forEach(d => {
				lookup[d.code] = d;
			});
			metadata[geo].array = meta;
			metadata[geo].lookup = lookup;

			let indicators = arr.map((d, i) => ({
				...meta[i],
				area: d.area,
				pop: d['2020'],
				density: d.density,
				age_med: d.age_med
			}));

			if (geo == "district") {
				['density', 'age_med'].forEach(key => {
					let values = indicators.map(d => d[key]).sort((a, b) => a - b);
					let breaks = getBreaks(values);
					indicators.forEach((d, i) => indicators[i][key + '_color'] = getColor(d[key], breaks, colors.seq));
				});
			}
			data[geo].indicators = indicators;

			let years = [
				2001, 2002, 2003, 2004, 2005,
				2006, 2007, 2008, 2009, 2010,
				2011, 2012, 2013, 2014, 2015,
				2016, 2017, 2018, 2019, 2020
			];

			let timeseries = [];
			arr.forEach(d => {
				years.forEach(year => {
					timeseries.push({
						code: d.code,
						name: d.name,
						value: d[year],
						year
					});
				});
			});
			data[geo].timeseries = timeseries;
		});
	});

	getTopo(topojson, 'geog')
	.then(geo => {
		geo.features.sort((a, b) => a.properties.AREANM.localeCompare(b.properties.AREANM));
		geojson = geo;
	});

	let cancerSampleCases = [
		{ cancer: 'Acute Myeloid Leukemia', cases: 200, color: '#FFA500' },
		{ cancer: 'Adrenocortical Carcinoma', cases: 92, color: '#FFA500' },
		{ cancer: 'Bladder Urothelial Carcinoma', cases: 412, color: '#FFA500' },
		{ cancer: 'Breast Ductal Carcinoma', cases: 778, color: '#FFA500' },
		{ cancer: 'Breast Lobular Carcinoma', cases: 201, color: '#FFA500' },
		{ cancer: 'Cervical Carcinoma', cases: 307, color: '#FFA500' },
		{ cancer: 'Cholangiocarcinoma', cases: 51, color: '#FFA500' },
		{ cancer: 'Colorectal Adenocarcinoma', cases: 633, color: '#FFA500' },
		{ cancer: 'Esophageal Carcinoma', cases: 185, color: '#FFA500' },
		{ cancer: 'Gastric Adenocarcinoma', cases: 443, color: '#FFA500' },
		{ cancer: 'Glioblastoma Multiforme', cases: 617, color: '#FFA500' },
		{ cancer: 'Head and Neck Squamous Cell Carcinoma', cases: 528, color: '#FFA500' },
		{ cancer: 'Hepatocellular Carcinoma', cases: 377, color: '#FFA500' },
		{ cancer: 'Chromophobe Renal Cell Carcinoma', cases: 113, color: '#FFA500' },
		{ cancer: 'Clear Cell Renal Cell Carcinoma', cases: 537, color: '#FFA500' },
		{ cancer: 'Papillary Renal Cell Carcinoma', cases: 291, color: '#FFA500' },
		{ cancer: 'Lower Grade Glioma', cases: 516, color: '#FFA500' },
		{ cancer: 'Lung Adenocarcinoma', cases: 585, color: '#0000FF' },
		{ cancer: 'Lung Squamous Cell Carcinoma', cases: 504, color: '#0000FF' },
		{ cancer: 'Mesothelioma', cases: 74, color: '#FFA500' },
		{ cancer: 'Ovarian Serous Adenocarcinoma', cases: 608, color: '#FFA500' },
		{ cancer: 'Pancreatic Ductal Adenocarcinoma', cases: 185, color: '#FFA500' },
		{ cancer: 'Paraganglioma & Pheochromocytoma', cases: 179, color: '#FFA500' }, // Neutral
		{ cancer: 'Prostate Adenocarcinoma', cases: 500, color: '#FFA500' },
		{ cancer: 'Sarcoma', cases: 261, color: '#FFA500' },
		{ cancer: 'Skin Cutaneous Melanoma', cases: 470, color: '#FFA500' },
		{ cancer: 'Testicular Germ Cell Cancer', cases: 150, color: '#FFA500' },
		{ cancer: 'Thymoma', cases: 124, color: '#FFA500' },
		{ cancer: 'Thyroid Papillary Carcinoma', cases: 507, color: '#FFA500' },
		{ cancer: 'Uterine Carcinosarcoma', cases: 57, color: '#FFA500' },
		{ cancer: 'Uterine Corpus Endometrioid Carcinoma', cases: 560, color: '#FFA500' },
		{ cancer: 'Uveal Melanoma', cases: 80, color: '#FFA500' }
		];

	let cancerTypeCounts = [
		{ label: 'LUAD', count: 566, color: '#1f77b4' },
		{ label: 'LUSC', count: 487, color: '#ff7f0e' }
		];

		let genderCounts = [
		{ label: 'Male', count: 597, color: '#2ca02c' },
		{ label: 'Female', count: 402, color: '#FFC0CB' },
		{ label: 'NA', count: 54, color: 'lightgrey' }
		];

		let ageBins = [
		{ label: '≤40', count: 7 },
		{ label: '40-49', count: 49 },
		{ label: '50-59', count: 206 },
		{ label: '60-69', count: 362 },
		{ label: '70-79', count: 299 },
		{ label: '80+', count: 48 },
		{ label: 'NA', count: 82, color: 'lightgrey' }
		];

</script>

<ONSHeader filled={true} center={false} />

<!-- PROJECT HEADER -->
<Header bgcolor="#206095" bgfixed={true} theme="dark" center={false} short={true}>
	<h1>Neural Networks for Lung Cancer Classification</h1>
	<p class="text-big" style="margin-top: 5px">
		A scrollytelling exploration of Non-Small Cell Lung Cancer (NSCLC) subtypes using neural networks and feature interpretability.
	</p>
	<p style="margin-top: 20px">
		12 May 2025
	</p>
	<p style="margin-top: 20px">
		Josh, Rodolfo, Sasha, Tirth
	</p>
</Header>

<!-- GOAL STATEMENT -->
<Filler theme="lightblue" short={true} wide={true} center={false}>
	<p class="text-big">
		<strong>Goal: Use mRNA sequencing data to classify patients into two lung cancer subtypes.</strong>
	</p>
	<p class="text-big">
		<strong>What is mRNA sequencing data?</strong><br>
		<br>mRNA-seq = value that quantifies gene activity<br>
		"Think of it as pixel intensities for genes"
	</p>
</Filler>

<!-- DATASET-->
 <Section>
	<h2>Datasets Used</h2>
	<p>
		We used the The Cancer Genome Atlas (TCGA) PanCancer dataset for genomic data.
	</p>
</Section>

<Media col='medium' caption="The infographic contains details on the TCGA project and its notable impacts on research.">
	<img src="/img/tcga-infographic-enlarge.png" alt="TCGA Infographic" />
</Media>
<Divider/>

<Section>
	<h2>Cancer Type Distribution</h2>
	<p>
	  The bar chart below shows sample counts from TCGA across various cancer types. 
	  We will be focusing on the bars in green, Lung Adenocarcinoma and Lung Squamous Cell Carcinoma.
	</p>
  </Section>

<!-- Bar Chart -->
{#if data.region.indicators}
<Media
	col="medium"
	caption="Source: NIH National Cancer Institute Center for Cancer Genomics."
>
	<div class="chart-sml">
		<BarChart
			data={cancerSampleCases}
			xKey="cases" 
			yKey="cancer"
			zKey="color"
			title="Cancer Type Sample Counts (TCGA)"
			height={1300}
			barHeight={40}
			padding={{ top: 10, bottom: 15, left: 140, right: 10 }}  
			>
			</BarChart>
	</div>
</Media>
{/if}

<Divider/>

<!-- Create Plots for Patient Demograhpics here-->
<Section>
	<h2>Dataset Demographics</h2>
	<p>
		We are focusing on the LUAD and LUSC subsets.
		This section shows the distribution of cancer types, patient gender, and diagnosis age in our TCGA dataset.
	</p>
</Section>

<!-- Cancer Type -->
<Media col="medium" caption="Distribution of cancer types in the dataset.">
	<div class="chart-sml">
	  <BarChart
		data={cancerTypeCounts}
		xKey="count"
		yKey="label"
		zKey="color"
		barHeight={40}
		height={cancerTypeCounts.length * 40 + 60}
		padding={{ top: 10, bottom: 15, left: 100, right: 30 }}
		title="Cancer Type Distribution"
		colors={['#1f77b4', '#D2042D']}
	  />
	</div>
  </Media>
  
  <!-- Gender -->
  <Media col="medium" caption="Gender distribution of NSCLC patients.">
	<div class="chart-sml">
	  <BarChart
		data={genderCounts}
		xKey="count"
		yKey="label"
		zKey="color"
		barHeight={40}
		height={genderCounts.length * 40 + 60}
		padding={{ top: 10, bottom: 15, left: 100, right: 30 }}
		title="Gender Distribution"
		colors={['#1f77b4', '#F33A6A', '#ccc' ]}
	  />
	</div>
  </Media>
  
  <!-- Diagnosis Age -->
  <Media col="medium" caption="Patient diagnosis age distribution.">
	<div class="chart-sml">
		<ColumnChart
		  data={ageBins}
		  xKey="label"
		  yKey="count"
		  zKey="color"
		  height={300}
		  padding={{ top: 10, bottom: 40, left: 40, right: 20 }}
		  title="Diagnosis Age Distribution"
		  colors={['#1f77b4', '#ccc']}
		/>
	  </div>
  </Media>

  <Section>
	<h2>Input data</h2>
	<p>
		We used 1 for LUAD class label and 0 for LUSC. <br>
		Duplicate/NA genes were dropped, and we used StandardScaler() on the raw mRNA seq values."
	</p>
	<Media col='wide' caption="A snapshot of the data provided to our models. Rows are patients (1018) and columns are genes (20,530)." >
	<img src="/img/data-snapshot2.png" alt="TCGA Infographic" />
	</Media>
  </Section>


<Divider />
<!-- Classic ML Models-->
<Section>
	<h2>Benchmarking: Classic ML Models</h2>
	<p>
		We are using XGBoost and Random Forest Trees as benchmarks for our Neural Networks. 
		These models provide feature importances which are useful for interpretability.
	</p>
	
</Section>

<!-- Chart 1 -->
<Media col='wide' caption="Random Forest Average Accuracy across 5 folds: 0.9519.">
	<img src="/img/RFT_Feature_Importance.png" alt="TCGA Infographic" />
</Media>
<!-- Chart 2 -->
<Media col='wide' caption="XGBoost Average Accuracy across 5 folds: 0.9646.">
	<img src="/img/XGBoost_Feature_Importance.png" alt="TCGA Infographic" />
</Media>

<Section>
	<h2>ML Discussion</h2>
	<p>
		RFT and XGBoost are tree based classifiers that are robust to noise and control overfitting well. <br>
		Both achieved strong accuracies with RFT reaching 95% and XGBoost 96% accuracies. <br>
		Our ML classifiers have set a high bar!
	</p>
</Section>

<Divider />

<!-- MLP -->
<Section>
	<h2>MLP Model</h2>
	<p>
	MLP Model Summary:
	</p>
	<div style="padding-left: 20px;">
	Input: 20,530 mRNA-seq features<br>
	Hidden Layers: 1024 → 128 → 32 neurons<br>
	Regularization: BatchNorm + ReLU + 53% Dropout after each hidden layer<br>
	Optimizer: Adam (learning rate = 4e-6)<br>
	Batch Size: 2048<br>
	Training: Early stopping with 5-patience, max 500 epochs<br>
	Result: 97.06% test accuracy
	</div>
</Section>

<Section>
	<h3>MLP Training</h3>
</Section>
<!-- Chart 1 -->
<Media col='medium' caption="MLP loss plot.">
	<img src="/img/MLP_loss_plot.png" alt="TCGA Infographic" />
</Media>

<Section>
	<h3>MLP Results</h3>
</Section>``
<!-- Chart 2 -->
<Media col='medium' caption="MLP Test Accuracy: 0.9706.">
	<img src="/img/MLP_acc_plot.png" alt="TCGA Infographic" />
</Media>
<!-- Chart 3 -->
<Media col='wide' caption="Integrated Gradients for feature attribution.">
	<img src="/img/MLP_Feature_Importance.png" alt="TCGA Infographic" />
</Media>

<Section>
	<h3>MLP Discussion</h3>
	<p>
		MLP achieved the best test accuracy (0.97) surpassing both classic ML models (XGBoost, Random Forest) and TabNet.
		The MLP required heavy regularization through dropout and batch norm to control overfitting. 
		Since MLP doesn't provide interpretability without feature attribution methods we used Integrated Gradients for the gene importances. 
	</p>
</Section>

<Divider />

<!-- TabNet -->
<Section>
	<h2>TabNet</h2>
	<p>
		TabNet is a deep learning architecture designed for tabular data. 
		It uses sequential attention to select which features to focus on at each decision step, allowing the model to learn sparse, interpretable representations.
	</p>
	<Media col='wide' caption="A snapshot of the the TabNet architecture." >
		<img src="/img/tabnet-arch.png" alt="TCGA Infographic" />
	</Media>
</Section>

<Section>
	<h3>TabNet Training</h3>
</Section>
<!-- Chart 1 -->
<Media col='medium' caption="TabNet loss plot.">
	<img src="/img/TabNet_loss_plot.png" alt="TCGA Infographic" />
</Media>

<Section>
	<h3>TabNet Results</h3>
</Section>``
<!-- Chart 2 -->
<Media col='medium' caption="TabNet Test Accuracy: 0.8824.">
	<img src="/img/TabNet_acc_plot.png" alt="TCGA Infographic" />
</Media>
<!-- Chart 3 -->
<Media col='wide' caption="">
	<img src="/img/TabNet_Feature_Importance.png" alt="TCGA Infographic" />
</Media>

<Section>
	<h3>TabNet Discussion</h3>
	<p>
		Although TabNet is specifically designed for tabular data with a built-in attention mechanism, it performed worse (88.24% accuracy) compared to simpler models.
		However, TabNet strength lies in providing interpretability out of the box through its attention-based feature selection.
	</p>

</Section>
<Divider />

<!-- Biomarker Discovery-->
<Section>
	<h2>Biomarker Discovery</h2>
	<Media col='wide' caption="A Venn diagram comparing top-ranked genes across models.">
	  <img src="/img/overlaps.png" alt="Top gene overlaps between MLP, TabNet, and ML models." />
	</Media>
	<p>
	  We compared top-ranked genes identified by MLP (Integrated Gradients), TabNet (attention scores), and ML models (Random Forest + XGBoost pooled) to discover potential LUAD vs LUSC biomarkers.
	</p>
	<p>
	  Overlap analysis showed that only one gene, <em>TBCCD1</em>, was common between MLP and ML models. However, it is not a well-established marker specifically distinguishing LUAD from LUSC 😔.
	</p>
	<p>
		A silver lining is that integrating multiple model outputs may used to reveal and confirm broader biological pathways.
	</p>
  </Section>
  <Divider />

  <Section>
	<h2>Limitations and Future Improvements</h2>
	<p>
		One major limitation of our study is the use of only mRNA expression data. 
		Incorporating multi-omic data such as DNA methylation, copy number variation, and exomic mutations can provide better biological insights.
	</p>
	<p>
		TabNet’s lower accuracy (0.8824) compared to MLP and classic ML models suggests that further tuning of its hyperparameters might be necessary. 
	</p>
	<p>
	  While the ML models (Random Forest and XGBoost) successfully recognized known LUAD vs LUSC biomarkers such as <em>KRT5</em> and <em>DSC3</em>, 
	  the MLP and TabNet models primarily identified genes associated with general cancer biology or potential noise, 
	  such as <em>TBCCD1</em>. 
	  This discrepancy likely stems from the purely data-driven nature of the approach, without integrating prior biological knowledge to guide feature selection. 
	  Incorporating domain-specific information could help focus the models on more biologically meaningful signals in future work.
	</p>
	<p>
	  In future iterations, we aim to:
	</p>
	<ul>
	  <li>Use Graph Neural Networks (GNNs) to model gene-gene and gene-pathway interactions.</li>
	  <li>Evaluate model robustness on external NSCLC cohorts.</li>
	</ul>
	<p>
	  Our long-term goal is to build explainable models that are not only accurate but also biologically meaningful and clinically actionable.
	</p>
  </Section>
<Footer />

<style>
	/* Styles specific to elements within the demo */
	:global(svelte-scroller-foreground) {
		pointer-events: none !important;
	}
	:global(svelte-scroller-foreground section div) {
		pointer-events: all !important;
	}
</style>
