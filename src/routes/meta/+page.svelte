<script>
import * as d3 from "d3";

import { onMount } from "svelte";

let data = [];
let commits = [];
let width = 1000, height = 600;
let margin = {top: 10, right: 10, bottom: 30, left: 20};
let xAxis, yAxis;


let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left
};
usableArea.width = usableArea.right - usableArea.left;
usableArea.height = usableArea.bottom - usableArea.top;




onMount(async () => {
    data = await d3.csv("/loc.csv", row => ({
        ...row,
        line: Number(row.line), // or just +row.line
        depth: Number(row.depth),
        length: Number(row.length),
        date: new Date(row.date + "T00:00" + row.timezone),
        datetime: new Date(row.datetime)
    }));

    commits = d3.groups(data, d => d.commit).map(([commit, lines]) => {
        let first = lines[0];
        let {author, date, time, timezone, datetime} = first;
        let ret = {
            id: commit,
            url: "https://github.com/USERNAME/REPO/commit/" + commit,
            author, date, time, timezone, datetime,
            hourFrac: datetime.getHours() + datetime.getMinutes() / 60,
            totalLines: lines.length
        };

        // Like ret.lines = lines
        // but non-enumerable so it doesn’t show up in JSON.stringify
        Object.defineProperty(ret, "lines", {
            value: lines,
            configurable: true,
            writable: true,
            enumerable: false,
        });

        return ret;
    });


});



$: minDate = d3.min(commits.map(d => d.date));
$: maxDate = d3.max(commits.map(d => d.date));
$: maxDatePlusOne = new Date(maxDate);
$: maxDatePlusOne.setDate(maxDatePlusOne.getDate() + 1);

$: xScale = d3.scaleTime()
              .domain([minDate, maxDatePlusOne])
              .range([usableArea.left, usableArea.right])
              .nice();

$: yScale = d3.scaleLinear()
              .domain([24, 0])
              .range([usableArea.bottom, usableArea.top]);

$: {
    d3.select(xAxis).call(d3.axisBottom(xScale));
    d3.select(yAxis).call(d3.axisLeft(yScale));
}


</script>

<p>Total lines of code: {data.length}</p>
<svg viewBox="0 0 {width} {height}">

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />

    <g class="dots">
    {#each commits as commit, index }
        <circle
            cx={ xScale(commit.datetime) }
            cy={ yScale(commit.hourFrac) }
            r="5"
            fill="steelblue"
        />
    {/each}
    </g> 

</svg>





<section>
    <h2>Summary</h2>
    <dl class="stats">
    <dt>Total <abbr title="Lines of code">LOC</abbr></dt>
    <dd>{data.length}</dd>
    <dt>Files</dt>
    <dd>{d3.groups(data, d => d.file).length}</dd>
    <dt>Commits</dt>
    <dd>{d3.groups(data, d => d.commit).length}</dd>
    </dl>
</section>


<style>
dl{
    display: grid;
    grid-template-columns: auto;
}
dt{
    grid-row: 1;
    font-family: inherit;
    font-weight: bold;
    color: var(--border-gray);
    text-transform: uppercase;
}
dd{
    font-family: inherit;
    font-weight: bold;
}
section{
    border-width:0.15em;
    border-style:solid;
    border-color:var(--border-gray);
    padding-left: 1em;
    padding-right: 1em;
}

svg {
        overflow: visible;
    }

</style>

