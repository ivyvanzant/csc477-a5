<!-- Backup of Chart 2 (Map) and Chart 3 (Bar Chart) removed from index.md for A5 -->

---

<h2 class="section-title">Layoffs by State</h2>
<p class="section-sub">Select a year to update the map.</p>

```js
const years = ["All", "2020", "2021", "2022", "2023", "2024", "2025"];
const selectedYear = Inputs.select(years, { label: "Year", value: "All" });
const year = Generators.input(selectedYear);
display(selectedYear);
```

```js
const stateNameToAbbr = {
  "Alabama":"AL","Alaska":"AK","Arizona":"AZ","Arkansas":"AR","California":"CA",
  "Colorado":"CO","Connecticut":"CT","Delaware":"DE","Florida":"FL","Georgia":"GA",
  "Hawaii":"HI","Idaho":"ID","Illinois":"IL","Indiana":"IN","Iowa":"IA",
  "Kansas":"KS","Kentucky":"KY","Louisiana":"LA","Maine":"ME","Maryland":"MD",
  "Massachusetts":"MA","Michigan":"MI","Minnesota":"MN","Mississippi":"MS",
  "Missouri":"MO","Montana":"MT","Nebraska":"NE","Nevada":"NV","New Hampshire":"NH",
  "New Jersey":"NJ","New Mexico":"NM","New York":"NY","North Carolina":"NC",
  "North Dakota":"ND","Ohio":"OH","Oklahoma":"OK","Oregon":"OR","Pennsylvania":"PA",
  "Rhode Island":"RI","South Carolina":"SC","South Dakota":"SD","Tennessee":"TN",
  "Texas":"TX","Utah":"UT","Vermont":"VT","Virginia":"VA","Washington":"WA",
  "West Virginia":"WV","Wisconsin":"WI","Wyoming":"WY","District of Columbia":"DC"
};

const yearFiltered = year === "All" ? data : data.filter(d => String(d.Year) === year);

const byState = d3.rollups(
  yearFiltered,
  v => ({ total: d3.sum(v, d => d.Laid_Off), count: v.length }),
  d => d.USState
);
const stateMap = new Map(byState.map(([state, val]) => [state, val]));
```

```js
const us = await fetch("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json").then(r => r.json());
const { feature } = await import("npm:topojson-client");
const states = feature(us, us.objects.states);
```

```js
const mapH = 480;
const maxVal = d3.max(byState, ([, v]) => v.total) || 1;
const color = d3.scaleSequential(d3.interpolateBlues).domain([0, maxVal]);
const projection = d3.geoAlbersUsa().fitSize([width, mapH], states);
const path = d3.geoPath().projection(projection);

const mapSvg = d3.create("svg").attr("width", width).attr("height", mapH);

const mapContainer = htl.html`
  <style>
    #tooltip-map {
      font: 10pt sans-serif;
      background-color: white;
      border: 1pt solid grey;
      padding: 5px;
      box-shadow: 3px 3px 3px darkgrey;
      max-width: 40ch;
      z-index: 1;
      visibility: hidden;
      position: absolute;
      pointer-events: none;
    }
  </style>
  <div id="tooltip-map"></div>
  ${mapSvg.node()}
`;

const mapTooltip = d3.select(mapContainer).select("#tooltip-map");

function attachMapTooltip(selection, tooltip, htmlFn) {
  selection
    .on("mouseover", function(event, d) {
      d3.select(this).attr("stroke", "#333").attr("stroke-width", 1.5);
      tooltip.style("visibility", "visible").html(htmlFn(d));
    })
    .on("mousemove", function(event) {
      tooltip.style("left", (event.pageX + 12) + "px").style("top", (event.pageY - 28) + "px");
    })
    .on("mouseout", function() {
      d3.select(this).attr("stroke", "#fff").attr("stroke-width", 0.5);
      tooltip.style("visibility", "hidden");
    });
}

mapSvg.selectAll("path.state")
  .data(states.features)
  .join(
    enter => enter.append("path")
      .attr("class", "state")
      .attr("d", path)
      .attr("fill", d => {
        const abbr = stateNameToAbbr[d.properties.name];
        const val = stateMap.get(d.properties.name) || stateMap.get(abbr);
        return val ? color(val.total) : "#eee";
      })
      .attr("stroke", "#fff")
      .attr("stroke-width", 0.5)
      .call(sel => attachMapTooltip(sel, mapTooltip, d => {
        const abbr = stateNameToAbbr[d.properties.name];
        const val = stateMap.get(d.properties.name) || stateMap.get(abbr);
        return `<strong>${d.properties.name}</strong><br/>Laid off: ${val ? d3.format(",")(val.total) : 0}<br/>Events: ${val ? val.count : 0}`;
      })),
    update => update
      .attr("fill", d => {
        const abbr = stateNameToAbbr[d.properties.name];
        const val = stateMap.get(d.properties.name) || stateMap.get(abbr);
        return val ? color(val.total) : "#eee";
      }),
    exit => exit.remove()
  );

// legend
const legendW = 200, legendH = 12;
const legendX = width - legendW - 20;
const defs = mapSvg.append("defs");
const grad = defs.append("linearGradient").attr("id", "legend-grad");
grad.selectAll("stop").data(d3.range(0, 1.01, 0.1)).join("stop")
  .attr("offset", d => d).attr("stop-color", d => color(d * maxVal));
const lg = mapSvg.append("g").attr("transform", `translate(${legendX}, ${mapH - 40})`);
lg.append("rect").attr("width", legendW).attr("height", legendH).style("fill", "url(#legend-grad)");
lg.append("text").attr("y", legendH + 14).style("font-size", "11px").text("0");
lg.append("text").attr("x", legendW).attr("y", legendH + 14).style("font-size", "11px")
  .attr("text-anchor", "end").text(d3.format(",")(maxVal) + " laid off");

display(mapContainer);
```

---

<h2 class="section-title">Top Companies by State</h2>
<p class="section-sub">Select a state to see which companies laid off the most workers there.</p>

```js
const validStates = new Set(["Alabama","Alaska","Arizona","Arkansas","California","Colorado","Connecticut","Delaware","Florida","Georgia","Hawaii","Idaho","Illinois","Indiana","Iowa","Kansas","Kentucky","Louisiana","Maine","Maryland","Massachusetts","Michigan","Minnesota","Mississippi","Missouri","Montana","Nebraska","Nevada","New Hampshire","New Jersey","New Mexico","New York","North Carolina","North Dakota","Ohio","Oklahoma","Oregon","Pennsylvania","Rhode Island","South Carolina","South Dakota","Tennessee","Texas","Utah","Vermont","Virginia","Washington","West Virginia","Wisconsin","Wyoming","District of Columbia"]);
const stateList = ["All States", ...Array.from(new Set(data.map(d => d.USState).filter(s => s && validStates.has(s)))).sort()];
const stateSelector = Inputs.select(stateList, { label: "State", value: "All States" });
const pickedState = Generators.input(stateSelector);
display(stateSelector);
```

```js
const companySource = pickedState === "All States" ? data : data.filter(d => d.USState === pickedState);

const companyData = d3.rollups(
  companySource,
  v => d3.sum(v, d => d.Laid_Off),
  d => d.Company
)
.map(([company, total]) => ({ company, total }))
.sort((a, b) => b.total - a.total)
.slice(0, 15);
```

```js
const barMargin = { top: 10, right: 30, bottom: 30, left: 160 };
const barW = width - barMargin.left - barMargin.right;
const barH = companyData.length * 28;

const barSvg = d3.create("svg").attr("width", width).attr("height", barH + barMargin.top + barMargin.bottom);

const barContainer = htl.html`
  <style>
    #tooltip-bar {
      font: 10pt sans-serif;
      background-color: white;
      border: 1pt solid grey;
      padding: 5px;
      box-shadow: 3px 3px 3px darkgrey;
      max-width: 40ch;
      z-index: 1;
      visibility: hidden;
      position: absolute;
      pointer-events: none;
    }
  </style>
  <div id="tooltip-bar"></div>
  ${barSvg.node()}
`;

const barTooltip = d3.select(barContainer).select("#tooltip-bar");

function attachBarTooltip(selection, tooltip, htmlFn) {
  selection
    .on("mouseover", function(event, d) {
      d3.select(this).attr("fill", "#2a5f8f");
      tooltip.style("visibility", "visible").html(htmlFn(d));
    })
    .on("mousemove", function(event) {
      tooltip.style("left", (event.pageX + 12) + "px").style("top", (event.pageY - 28) + "px");
    })
    .on("mouseout", function() {
      d3.select(this).attr("fill", "steelblue");
      tooltip.style("visibility", "hidden");
    });
}

const bg = barSvg.append("g").attr("transform", `translate(${barMargin.left},${barMargin.top})`);

const xBar = d3.scaleLinear().domain([0, d3.max(companyData, d => d.total) || 1]).range([0, barW]);
const yBar = d3.scaleBand().domain(companyData.map(d => d.company)).range([0, barH]).padding(0.2);

bg.append("g").call(d3.axisLeft(yBar).tickSize(0)).select(".domain").remove();
bg.append("g").attr("transform", `translate(0,${barH})`).call(d3.axisBottom(xBar).ticks(5).tickFormat(d3.format(",")));

bg.selectAll("rect.bar")
  .data(companyData)
  .join(
    enter => enter.append("rect")
      .attr("class", "bar")
      .attr("y", d => yBar(d.company))
      .attr("width", d => xBar(d.total))
      .attr("height", yBar.bandwidth())
      .attr("fill", "steelblue")
      .attr("rx", 3)
      .call(sel => attachBarTooltip(sel, barTooltip, d =>
        `<strong>${d.company}</strong><br/>Laid off: ${d3.format(",")(d.total)}`
      )),
    update => update
      .attr("y", d => yBar(d.company))
      .attr("width", d => xBar(d.total))
      .attr("height", yBar.bandwidth()),
    exit => exit.remove()
  );

display(barContainer);
```
