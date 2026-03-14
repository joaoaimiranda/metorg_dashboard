---
toc: true
theme: "dashboard"
---

<div class="hero">
  <h1>Decoding Metastasis Organotropism</h1>
  <!-- <h2>Welcome to your new app! Edit&nbsp;<code style="font-size: 90%;">src/index.md</code> to change this page.</h2>
  <a href="https://observablehq.com/framework/getting-started">Get started<span style="display: inline-block; margin-left: 0.25rem;">↗︎</span></a> -->
</div>

## Metastasis-associated Intercellular Interactions 

```js
const mints = aq.fromCSV(await FileAttachment("data/mints_all.csv").text());
const weights = aq.fromCSV(await FileAttachment("data/weights.csv").text());
const frequencies = aq.fromCSV(await FileAttachment('data/frequencies.csv').text());
const proteins = aq.fromCSV(await FileAttachment('data/proteins.csv').text());
const opentargets = aq.fromCSV(
  await FileAttachment('data/open_targets.csv').text()
);
const rankedmints = aq.fromArrow(
  await FileAttachment("data/ranked_mints.arrow").arrow()
);
```

```js
const genes = mints
  .select('cancer', 'metastasis')
  .fold(aq.all()) // stacks the two columns into one
  .dedupe('value')
  .array('value')
  .sort();
```

```js
const unique_mints = mints
  .select(aq.not('fdr', 'metdataset'))
  .groupby('mint_id')
  .derive({ 'rho': d => aq.op.median(d.rho) }, )
  .dedupe()
  .ungroup()
```

```js
// Functions
function getPlotData(selectedMint, frequencies, weights) {
  if (!selectedMint) return null;
  
  const mint_id = selectedMint.mint_id.toFixed(0);
  return frequencies
    .join(weights, 'pair_id', [aq.all(), mint_id])
    .rename({[mint_id]: 'weight'});
}

function getAvailableDatasets(selectedMint, mintsData) {
  if (!selectedMint) return [];
  
  const filtered = mintsData
    .filter(aq.escape(d => d.mint_id === selectedMint.mint_id));
  
  return Array.from(new Set(filtered.array('metdataset'))).sort();
}

function displayPlot(plotData, selectedDataset) {
  if (!plotData || !selectedDataset) {
    return htl.html`<div>Please select an interaction on the table</div>`;
  }
  
  const filteredPlot = plotData
    .params({selectedDataset})
    .filter(d => aq.op.includes(d.metdataset, selectedDataset));

  if (filteredPlot.numRows() === 0) {
    return htl.html`<div>No data available for selected dataset</div>`;
  }

  return Plot.plot({
    marks: [
      Plot.dot(filteredPlot, {
        x: "freq",
        y: 'weight',
        stroke: "red",
      }),
      Plot.linearRegressionY(filteredPlot, {
        x: "freq",
        y: 'weight',
        stroke: "red",
        strokeWidth: 2,
      }),
    ],
    x: {label: "Frequency", type: "log"},
    y: {label: "Weights", labelOffset: 30},
  });
}

function getStats(selectedMint, selectedDataset, mintsData) {
  if (!selectedMint || !selectedDataset) {
    return { rho: null, fdr: null };
  }
  
  const filtered = mintsData
    .filter(aq.escape(
      d => d.mint_id === selectedMint.mint_id &&
      d.metdataset === selectedDataset
    ));
  
  if (filtered.numRows() === 0) {
    return { rho: null, fdr: null };
  }
  
  const rho = filtered.get('rho', 0).toFixed(3);
  const fdr = filtered.get('fdr', 0).toExponential(2);
  
  return { rho, fdr };
}
```

```js
const gene = Inputs.search(unique_mints, {
  label: "Gene",
  placeholder: "Enter gene symbol...",
  column: ["cancer", "metastasis"],
  datalist: genes,
  autocapitalize: "characters",
});

const viewofgene = view(gene);
```

```js
const minttable = Inputs.table(viewofgene, {
  columns: [
    "cancer",
    "metastasis",
    "rho",
    "signal",
    "tau",
  ],
  header: {
    "cancer": "cancer tissue gene",
    "metastasis": "metastasis tissue gene",
    "rho": "median spearman rho",
  },
  layout: "auto",
  select: true,
  multiple: false,
  value: viewofgene[142]
});

const viewoftable = view(minttable);
```

```js
const radio_input = Inputs.radio(
  availableDatasets,
  {
    label: "Frequency Dataset:",
    value: availableDatasets.length > 0 ? availableDatasets[0] : null,
    disabled: availableDatasets.length === 0
  }
);
```

```js
const viewofradio_input = view(radio_input);
```

```js
const plot = getPlotData(viewoftable, frequencies, weights);
```

```js
const availableDatasets = getAvailableDatasets(viewoftable, mints);
```

```js
const stats = getStats(viewoftable, viewofradio_input, mints);
```

<div class="card">
  <div style="padding: 1em">${ resize(width => gene) }
  </div>${ resize(width => minttable) }
</div>

<div class="grid grid-cols-3">
  <div class="card grid-rowspan-2 grid-colspan-2">
    <div style="padding: 1em">${ resize(width => radio_input) }
    </div>${ resize(width => displayPlot(plot, viewofradio_input)) }
  </div>
  <div class="card grid-rowspan-1 grid-colspan-1">
    <h2>spearman rho</h2>
    <span class="big">${stats.rho}</span>
  </div>
  <div class="card grid-rowspan-1 grid-colspan-1">
    <h2>FDR</h2>
    <span class="big">${stats.fdr}</span>
  </div>
</div>

## Tissue-specific Analysis

```js
const rankedmintssearch = Inputs.search(rankedmints, {
  placeholder: "Search term...",
  column: ["cancer", "metastasis"],
  datalist: genes,
});

const viewofrankedmintssearch = view(rankedmintssearch);
```

```js
const rankedmintstable = Inputs.table(viewofrankedmintssearch, {
  columns: [
    "cancer",
    "metastasis",
    "tissue",
    "tissuetype",
    "rho",
    "rank",
  ],
  header: {
    "cancer": "cancer tissue gene",
    "metastasis": "metastasis tissue gene",
    "tissue": "Tissue",
    "tissuetype": "Site",
    "rho": "Median Spearman Rho",
    "rank": "Rank"
  },
  layout: "auto",
  select: false,
  multiple: true,
});
```

<div class="card">
  <div style="padding: 1em">${ resize(width => rankedmintssearch) }
  </div>${ resize(width => rankedmintstable) }
</div>

## Top 10% MINT Proteins 

```js
const topmintproteins = Inputs.search(proteins, {
  placeholder: "Search gene and tissue...",
  column: ["gene_id", "tissue"],
  datalist: proteins.dedupe('gene_id').array('gene_id'),
});
const viewoftopmintproteins = view(topmintproteins);
```

```js
const proteintable = Inputs.table(viewoftopmintproteins, {
  columns: [
    "gene_id",
    "tissue",
    "tissuetype",
    "rho",
  ],
  header: {
    "gene_id": "Gene ID",
    "tissue": "Tissue",
    "tissuetype": "Site",
    "rho": "Max Spearman Rho",
  },
  layout: "auto",
  select: true,
  multiple: false,
});

```

<div class="card">
  <div style="padding: 1em">${ resize(width => topmintproteins) }
  </div>${ resize(width => proteintable) }
</div>

## Open Targets Candidates

```js
const opentargetssearch = Inputs.search(opentargets, {
  placeholder: "Search gene...",
  column: "gene_id",
  datalist: proteins.array('gene_id'),
  autocapitalize: "characters"
});
const viewofopentargetssearch = view(opentargetssearch);
```

```js
const opentargetstable = Inputs.table(viewofopentargetssearch, {
  header: {
    "gene_id": "Gene ID",
    "cancer_drug": "Cancer Drug",
    "noncancer_drug": "Non-Cancer Drug",
    "repurposing_candidate": "Repurposing Candidate",
    "n_mints": "Number of Interactions"
  },
  layout: "auto",
  select: true,
  multiple: false,
});
```

<div class="card">
  <div style="padding: 1em">${ resize(width => opentargetssearch) }
  </div>${ resize(width => opentargetstable) }
</div>

<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 1rem 0;
  padding: 1rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 900;
  line-height: 1;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>
