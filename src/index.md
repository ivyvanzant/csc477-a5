---
toc: false
---

<h1>Global Tech Layoffs Over Time</h1>
<p class="subtitle">An interactive exploration of layoffs across tech companies worldwide from 2020–2025</p>

```js
import * as d3 from "npm:d3";
const raw = await FileAttachment("data/layoffs.csv").csv({ typed: true });
const data = raw.filter(d => d.Date_layoffs && d.Laid_Off > 0);
```

---

<h2 class="section-title">Layoffs Over Time</h2>
<p class="section-sub">Brush the timeline to explore changes in layoffs across different time periods</p>

```js
const brushRange = Mutable([null, null]);
const setBrush = (range) => { brushRange.value = range; };
```

```js
const parseDate = d3.timeParse("%Y-%m-%d");
const formatMonth = d3.timeFormat("%Y-%m");

const byMonth = d3.rollups(
  data,
  v => d3.sum(v, d => d.Laid_Off),
  d => formatMonth(typeof d.Date_layoffs === "string" ? parseDate(d.Date_layoffs) : d.Date_layoffs)
)
.map(([month, total]) => ({ date: new Date(month + "-01"), total }))
.sort((a, b) => a.date - b.date);
```

```js
const margin = { top: 20, right: 30, bottom: 35, left: 70 };
const W = width - margin.left - margin.right;
const H = 280;

const svg = d3.create("svg").attr("width", width).attr("height", H + margin.top + margin.bottom);

const container = htl.html`
  <style>
    #tooltip-timeline {
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
  <div id="tooltip-timeline"></div>
  ${svg.node()}
`;

const tooltip = d3.select(container).select("#tooltip-timeline");

const g = svg.append("g").attr("transform", `translate(${margin.left},${margin.top})`);

const x = d3.scaleTime().domain(d3.extent(byMonth, d => d.date)).range([0, W]);
const y = d3.scaleLinear().domain([0, d3.max(byMonth, d => d.total)]).nice().range([H, 0]);

const area = d3.area().x(d => x(d.date)).y0(H).y1(d => y(d.total)).curve(d3.curveMonotoneX);
const line = d3.line().x(d => x(d.date)).y(d => y(d.total)).curve(d3.curveMonotoneX);

g.append("g")
  .attr("transform", `translate(0,${H})`)
  .call(d3.axisBottom(x).ticks(d3.timeMonth.every(3)).tickFormat(d3.timeFormat("%b %Y")))
  .selectAll("text").style("text-anchor", "middle");

g.append("g").call(d3.axisLeft(y).ticks(6).tickFormat(d3.format(",")));

g.append("text").attr("x", -H / 2).attr("y", -50).attr("transform", "rotate(-90)")
  .attr("text-anchor", "middle").style("font-size", "12px").text("Employees Laid Off");

g.append("path").datum(byMonth).attr("fill", "steelblue").attr("fill-opacity", 0.3).attr("d", area);
g.append("path").datum(byMonth).attr("fill", "none").attr("stroke", "steelblue").attr("stroke-width", 2).attr("d", line);

// Vertical line for tooltip crosshair
const hoverLine = g.append("line")
  .attr("class", "hover-line")
  .attr("y1", 0).attr("y2", H)
  .attr("stroke", "#999").attr("stroke-width", 1)
  .attr("stroke-dasharray", "4")
  .style("visibility", "hidden");

const bisect = d3.bisector(d => d.date).left;

const brush = d3.brushX()
  .extent([[0, 0], [W, H]])
  .on("brush end", ({ selection }) => {
    setBrush(selection ? selection.map(x.invert) : [null, null]);
  });

const brushG = g.append("g").attr("class", "brush").call(brush);

// Attach tooltip to the brush overlay so it works alongside brushing
brushG.select(".overlay")
  .on("mousemove", function(event) {
    const [mx] = d3.pointer(event);
    const date = x.invert(mx);
    const i = bisect(byMonth, date);
    const d = byMonth[Math.min(i, byMonth.length - 1)];
    if (!d) return;
    hoverLine.attr("x1", x(d.date)).attr("x2", x(d.date)).style("visibility", "visible");
    tooltip.style("visibility", "visible")
      .style("left", (event.pageX + 12) + "px")
      .style("top", (event.pageY - 28) + "px")
      .html(`<strong>${d3.timeFormat("%B %Y")(d.date)}</strong><br/>Laid off: ${d3.format(",")(d.total)}`);
  })
  .on("mouseout", function() {
    hoverLine.style("visibility", "hidden");
    tooltip.style("visibility", "hidden");
  });
display(container);
```

```js
const [start, end] = brushRange;
const brushFiltered = (start && end)
  ? data.filter(d => {
      const t = typeof d.Date_layoffs === "string" ? new Date(d.Date_layoffs) : d.Date_layoffs;
      return t >= start && t <= end;
    })
  : data;

const totalLaidOff = d3.sum(brushFiltered, d => d.Laid_Off);
const numCompanies = new Set(brushFiltered.map(d => d.Company)).size;
const peakCompany = d3.rollups(brushFiltered, v => d3.sum(v, d => d.Laid_Off), d => d.Company)
  .sort((a, b) => b[1] - a[1])[0];

const formatDate = d3.timeFormat("%b %Y");
const dateRange = (start && end) ? `${formatDate(start)} - ${formatDate(end)}` : "All Time";

const statsDiv = html`<div class="stats-row">
  <div class="stat-card"><div class="stat-value">${dateRange}</div><div class="stat-label">Selected Range</div></div>
  <div class="stat-card"><div class="stat-value">${d3.format(",")(totalLaidOff)}</div><div class="stat-label">Total Amount Laid Off</div></div>
  <div class="stat-card"><div class="stat-value">${numCompanies}</div><div class="stat-label">Companies Affected</div></div>
  <div class="stat-card"><div class="stat-value">${peakCompany ? peakCompany[0] : "-"}</div><div class="stat-label">Peak Company</div></div>
</div>`;
display(statsDiv);
```

---

<h2 class="section-title">Design Rationale</h2>

<br/>

**Design Goal**

I chose to create an interactive visualization exploring tech layoffs globally from 2020-2025. This is the topic for my group project, so I chose to use assignment 5 as a chance to play around with the dataset and find ways to visualize the data in a useful way. Over the last several years, layoffs in the tech industry became a major topic across news outlets and social media, especially following major worldwide events like the COVID-19 pandemic and the subsequent economic slowdown. As well, with the release of generative AI replacing jobs and creating new ones, I wanted to design a visualization that helps users explore how have layoffs changed over time and identify periods where layoffs were especially severe.

Rather than attempting to visualize every aspect of the dataset, I focused specifically on temporal patterns and interactive exploration. The central question this visualization aims to answer is: **How have tech industry layoffs changed over time, and when were they most severe?** The goal is to allow users to investigate how layoffs fluctuated month-to-month and examine summary statistics for specific time periods.

**Visualization and Encoding Choices**

The primary visualization is an area and line chart showing the total number of employees laid off over time. I chose a time-series visualization because it is one of the clearest ways to display trends, spikes, and long-term patterns in sequential data. The filled area helps emphasize the magnitude of layoffs during major events, while the line overlay improves readability and makes peaks easier to follow visually.

The x-axis represents time from 2020–2025 in three month increments, while the y-axis represents the total number of employees laid off during each month.

Below the chart, I included summary statistic cards that dynamically update based on user interaction. These cards display the selected date range, total layoffs, number of companies affected, and the company responsible for the largest layoffs within the selected period. These metrics provide quick context without requiring users to manually interpret every point on the graph.

**Interaction Design**

The primary interaction technique used in this project is brushing. Users can drag across the timeline to select a custom date range, allowing them to focus on specific periods of interest. Once a range is selected, the summary statistics automatically update to reflect only the filtered data.

I chose brushing because it creates a direct and intuitive way to explore different data points quickly. Instead of forcing users to rely on dropdown menus or buttons, brushing allows continuous exploration and supports rapid comparisons between different periods.

I also implemented tooltips that appear when hovering over differnt points in the graph. These provide details-on-demand by displaying the exact month and layoff total without adding noise to the chart with many annotations. It also lets the user know the exact month they are hovering over when beginning or ending the brush.

**Alternative Designs Considered**

I considered using a bar chart for monthly layoffs, but the continuous nature of the line and area chart better communicated overall trends and major spikes over time. In future iterations, I would like to add a geographic component, such as a choropleth map or a company-level bar chart, to better answer where layoffs were concentrated around the world.

**References and Data Sources**

- Dataset: [Layoffs.fyi](https://layoffs.fyi/) via [Kaggle](https://www.kaggle.com/datasets/ulrikeherold/tech-layoffs-2020-2024)
- [D3.js documentation](https://d3js.org/) - brushing (`d3.brushX`), bisector (`d3.bisector`), and data aggregation (`d3.rollups`)
- [Observable Framework documentation](https://observablehq.com/framework/) - `Mutable`, `FileAttachment`
- [View source on GitHub](https://github.com/ivyvanzant/csc477-a5)

---

<style>
h1 { font-size: 2rem; margin-bottom: 0.25rem; pointer-events: none; }
h1 a, h1 a:hover { color: inherit; text-decoration: none; pointer-events: none; }
.subtitle { font-size: 1.1rem; color: #666; margin-top: 0; margin-bottom: 1.5rem; }
.section-title { pointer-events: none; }
.section-title a, .section-title a:hover { color: inherit; text-decoration: none; pointer-events: none; }
.section-sub { color: #555; margin-top: -0.5rem; margin-bottom: 1rem; }
hr { margin: 2rem 0; }
.brush .selection { fill: steelblue; fill-opacity: 0.2; stroke: steelblue; }
.stats-row { display: flex; gap: 1.5rem; margin: 1rem 0 2rem; }
.stat-card { background: #f5f5f5; border-radius: 8px; padding: 1rem 1.5rem; min-width: 140px; text-align: center; }
.stat-value { font-size: 1.8rem; font-weight: 700; color: steelblue; }
.stat-label { font-size: 0.85rem; color: #666; margin-top: 0.25rem; }
</style>
