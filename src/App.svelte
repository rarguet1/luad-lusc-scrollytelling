<script>
	// CORE IMPORTS
	import { setContext, onMount } from "svelte";
	import { getMotion } from "./utils.js";
	import { themes } from "./config.js";
	import ONSHeader from "./layout/ONSHeader.svelte";
	import ONSFooter from "./layout/ONSFooter.svelte";
	import Header from "./layout/Header.svelte";
	import Section from "./layout/Section.svelte";
	import Media from "./layout/Media.svelte";
	import Scroller from "./layout/Scroller.svelte";
	import Filler from "./layout/Filler.svelte";
	import Divider from "./layout/Divider.svelte";
	import Toggle from "./ui/Toggle.svelte";
	import Arrow from "./ui/Arrow.svelte";
	
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
		{ cancer: 'Adrenocortical Carcinoma', cases: 92, color: '#FFD700' },
		{ cancer: 'Bladder Urothelial Carcinoma', cases: 412, color: '#000080' },
		{ cancer: 'Breast Ductal Carcinoma', cases: 778, color: '#FFC0CB' },
		{ cancer: 'Breast Lobular Carcinoma', cases: 201, color: '#FFC0CB' },
		{ cancer: 'Cervical Carcinoma', cases: 307, color: '#008080' },
		{ cancer: 'Cholangiocarcinoma', cases: 51, color: '#4CBB17' },
		{ cancer: 'Colorectal Adenocarcinoma', cases: 633, color: '#00008B' },
		{ cancer: 'Esophageal Carcinoma', cases: 185, color: '#CCCCFF' },
		{ cancer: 'Gastric Adenocarcinoma', cases: 443, color: '#CCCCFF' },
		{ cancer: 'Glioblastoma Multiforme', cases: 617, color: '#808080' },
		{ cancer: 'Head and Neck Squamous Cell Carcinoma', cases: 528, color: '#800020' },
		{ cancer: 'Hepatocellular Carcinoma', cases: 377, color: '#50C878' },
		{ cancer: 'Chromophobe Renal Cell Carcinoma', cases: 113, color: '#FFA500' },
		{ cancer: 'Clear Cell Renal Cell Carcinoma', cases: 537, color: '#FFA500' },
		{ cancer: 'Papillary Renal Cell Carcinoma', cases: 291, color: '#FFA500' },
		{ cancer: 'Lower Grade Glioma', cases: 516, color: '#808080' },
		{ cancer: 'Lung Adenocarcinoma', cases: 585, color: '#EAE0C8' },
		{ cancer: 'Lung Squamous Cell Carcinoma', cases: 504, color: '#EAE0C8' },
		{ cancer: 'Mesothelioma', cases: 74, color: '#0000FF' },
		{ cancer: 'Ovarian Serous Adenocarcinoma', cases: 608, color: '#008080' },
		{ cancer: 'Pancreatic Ductal Adenocarcinoma', cases: 185, color: '#800080' },
		{ cancer: 'Paraganglioma & Pheochromocytoma', cases: 179, color: '#AAAAAA' }, // Neutral
		{ cancer: 'Prostate Adenocarcinoma', cases: 500, color: '#ADD8E6' },
		{ cancer: 'Sarcoma', cases: 261, color: '#FFFF00' },
		{ cancer: 'Skin Cutaneous Melanoma', cases: 470, color: '#000000' },
		{ cancer: 'Testicular Germ Cell Cancer', cases: 150, color: '#800080' },
		{ cancer: 'Thymoma', cases: 124, color: '#008080' },
		{ cancer: 'Thyroid Papillary Carcinoma', cases: 507, color: '#008080' },
		{ cancer: 'Uterine Carcinosarcoma', cases: 57, color: '#FFE5B4' },
		{ cancer: 'Uterine Corpus Endometrioid Carcinoma', cases: 560, color: '#FFE5B4' },
		{ cancer: 'Uveal Melanoma', cases: 80, color: '#000000' }
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
	<h1>Neural Networks for NSCLC Classification</h1>
	<p class="text-big" style="margin-top: 5px">
		A scrollytelling exploration of Non-Small Cell Lung Cancer (NSCLC) subtypes using neural networks and feature interpretability.
	</p>
	<p style="margin-top: 20px">
		18 April 2025
	</p>
	<p style="margin-top: 20px">
		Josh, Rodolfo, Sasha, Tirth
	</p>
	<!-- <p>
		<Toggle label="Animation {animation ? 'on' : 'off'}" mono={true} bind:checked={animation}/>
	</p> -->
	<!-- <div style="margin-top: 90px;">
		<Arrow color="white" {animation}>Scroll to begin</Arrow>
	</div> -->
</Header>

<!-- GOAL STATEMENT -->
<Filler theme="lightblue" short={true} wide={true} center={false}>
	<p class="text-big">
		<strong>Goal: Use mRNA sequencing data to classify patients into LUAD or LUSC subtypes.</strong>
	</p>
	<p class="text-big">
		<strong>What is MRNA sequencing data?</strong><br>
		<br>mRNA-seq = value that quantifies gene activity<br>
		"Think of it as pixel intensities for genes"
	</p>
</Filler>

<!-- First story section, Why Lung Cancer
<Section>
	<h2>Why Lung Cancer?</h2>
	<p>
		Lung cancer is the leading cause of cancer-related deaths worldwide. 
		Non-Small Cell Lung Cancer (NSCLC), which includes LUAD (adenocarcinoma) and LUSC (squamous cell carcinoma), accounts for 85% of cases.
	</p>
	<p>
		Precise classification of these subtypes is essential to guide targeted treatment and improve patient outcomes. 
		In this project, we use neural networks to distinguish between LUAD and LUSC based on genomic data.
	</p>
	<blockquote class="text-indent">
		"Lung cancer is the leading cause of cancer-related deaths worldwide, accounting for the highest mortality rates among both men and women."&mdash;World Health Organization
	</blockquote>
</Section> -->


<!-- Second story section, Dataset used-->
 <Section>
	<h2>Datasets Used</h2>
	<p>
		We used the The Cancer Genome Atlas (TCGA) PanCancer dataset.
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
	  Each bar’s color corresponds to the <strong>awareness ribbon</strong> color associated with that cancer type.
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
			colors={[
				'#FFA500',  // Acute Myeloid Leukemia – Orange
				'#FFD700',  // Adrenocortical Carcinoma – Gold
				'#000080',  // Bladder Urothelial Carcinoma – Navy
				'#FFC0CB',  // Breast Ductal Carcinoma – Pink
				'#FFC0CB',  // Breast Lobular Carcinoma – Pink
				'#008080',  // Cervical Carcinoma – Teal
				'#4CBB17',  // Cholangiocarcinoma – Kelly Green
				'#00008B',  // Colorectal Adenocarcinoma – Dark Blue
				'#CCCCFF',  // Esophageal Carcinoma – Periwinkle
				'#CCCCFF',  // Gastric Adenocarcinoma – Periwinkle
				'#808080',  // Glioblastoma Multiforme – Gray
				'#800020',  // Head and Neck Squamous Cell Carcinoma – Burgundy
				'#50C878',  // Hepatocellular Carcinoma – Emerald Green
				'#FFA500',  // Chromophobe Renal Cell Carcinoma – Orange
				'#FFA500',  // Clear Cell Renal Cell Carcinoma – Orange
				'#FFA500',  // Papillary Renal Cell Carcinoma – Orange
				'#808080',  // Lower Grade Glioma – Gray
				'#EDEDED',  // Lung Adenocarcinoma – Soft White
				'#EDEDED',  // Lung Squamous Cell Carcinoma – Soft White
				'#0000FF',  // Mesothelioma – Blue
				'#008080',  // Ovarian Serous Adenocarcinoma – Teal
				'#800080',  // Pancreatic Ductal Adenocarcinoma – Purple
				'#AAAAAA',  // Paraganglioma & Pheochromocytoma – Neutral
				'#ADD8E6',  // Prostate Adenocarcinoma – Light Blue
				'#FFFF00',  // Sarcoma – Yellow
				'#000000',  // Skin Cutaneous Melanoma – Black
				'#800080',  // Testicular Germ Cell Cancer – Purple
				'#008080',  // Thymoma – Teal
				'#008080',  // Thyroid Papillary Carcinoma – Teal
				'#FFE5B4',  // Uterine Carcinosarcoma – Peach
				'#FFE5B4',  // Uterine Corpus Endometrioid Carcinoma – Peach
				'#000000'   // Uveal Melanoma – Black
			  ]}			  
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
		We are focusing on the Lung Adenocarcinoma and Lung Squamous Cell Carcinoma subsets.
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

<!-- <Section>
	<h2>Goal:</h2>
	<p>
		Use MRNA sequencing data (values that quantify the activity of a gene) to classify patients into LUAD or LUSC subtypes.
	</p>
</Section> -->

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

<Divider />

<!-- MLP -->
<Section>
	<h2>MLP Model</h2>
	<p>
		A 4-layer MLP with batch normalization and dropout (0.53) after each hidden layer, trained using Adam optimizer to classify LUAD vs LUSC from 20,530 mRNA features.
	</p>
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
		MLP doesn't provide interpretability without feature attribution methods. 
		In biological settings, it's necessary to know exactly what features led to a classification.
	</p>
</Section>

<Divider />

<!-- TabNet -->
<Section>
	<h2>TabNet</h2>
	<p>
		Google model designed for tabular data, uses built-in attention mechanism. Insert quick explanation of TabNet. 
	</p>
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
		It was surprising that TabNet, despite its built-in attention mechanism for tabular data, performed worse (0.8824 accuracy) than simpler models.
		However, TabNet strength lies in providing interpretability out of the box through feature importances.
	</p>

</Section>
<Divider />

<!-- Biomarker Discovery-->
 <Section>
	<h2>Biomarker Discovery</h2>
	<p>
		IG feature from MLP. TabNet attention results. Compare with literature.
	</p>
 </Section>
 <Divider />

<ONSFooter />

<style>
	/* Styles specific to elements within the demo */
	:global(svelte-scroller-foreground) {
		pointer-events: none !important;
	}
	:global(svelte-scroller-foreground section div) {
		pointer-events: all !important;
	}
	select {
		max-width: 350px;
	}
	.chart {
		margin-top: 45px;
		width: calc(100% - 5px);
	}
	.chart-full {
		margin: 0 20px;
	}
	.chart-sml {
		font-size: 0.85em;
	}
	/* The properties below make the media DIVs grey, for visual purposes in demo */
	.media {
		background-color: #f0f0f0;
		display: -webkit-box;
		display: -ms-flexbox;
		display: flex;
		-webkit-box-orient: vertical;
		-webkit-box-direction: normal;
		-ms-flex-flow: column;
		flex-flow: column;
		-webkit-box-pack: center;
		-ms-flex-pack: center;
		justify-content: center;
		text-align: center;
		color: #aaa;
	}
</style>
