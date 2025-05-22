<script>
import * as d3 from "d3";
import Bar from '$lib/Bar.svelte';
import Scrolly from "svelte-scrolly";
import FileLines from '$lib/FileLines.svelte';

import { onMount } from "svelte";
    // import FileLines from "../../lib/FileLines.svelte";

let data = [];
let commits = [];
let width = 1000, height = 600;
let margin = {top: 10, right: 10, bottom: 30, left: 20};
let xAxis, yAxis;
let yAxisGridlines;
let hoveredIndex = -1;
let cursor = {x: 0, y: 0};
let clickedCommits = [];
let commitProgress = 100;





$: hoveredCommit = filteredCommits[hoveredIndex] ?? hoveredCommit ?? {};



let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left
};
usableArea.width = usableArea.right - usableArea.left;
usableArea.height = usableArea.bottom - usableArea.top;




onMount(async () => {
    data = await d3.csv("./loc.csv", row => ({
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
        
    commits = d3.sort(commits, d => -d.totalLines);
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

$: filteredData = data.filter(d => d.datetime <= commitMaxTime);



$: allTypes = Array.from(new Set(data.map(d => d.type)));
$: selectedLines = (clickedCommits.length > 0 ? clickedCommits : filteredCommits).flatMap(d => d.lines);
$: selectedCounts = d3.rollup(
    selectedLines,
    v => v.length,
    d => d.type
);
$: languageBreakdown = allTypes.map(type => [type, selectedCounts.get(type) || 0]);


$: minDate = d3.min(commits.map(d => d.date));
$: maxDate = d3.max(commits.map(d => d.date));
$: maxDatePlusOne = new Date(maxDate);
$: maxDatePlusOne.setDate(maxDatePlusOne.getDate() + 1);

$: filteredCommits = commits.filter(
    commit => commit.datetime <= commitMaxTime);

$: filteredMinDate = d3.min(filteredCommits.map(d => d.date));
$: filteredMaxDate = d3.max(filteredCommits.map(d => d.date));
$: filteredMaxDatePlusOne = new Date(filteredMaxDate);
$: filteredMaxDatePlusOne.setDate(filteredMaxDatePlusOne.getDate() + 1);



$: xScale = d3.scaleTime()
              .domain([filteredMinDate, filteredMaxDatePlusOne])
              .range([usableArea.left, usableArea.right])
              .nice();

$: yScale = d3.scaleLinear()
              .domain([24, 0])
              .range([usableArea.bottom, usableArea.top]);

$: {
    d3.select(xAxis).call(d3.axisBottom(xScale));
    d3.select(yAxis).call(d3.axisLeft(yScale).tickFormat(d => String(d % 24).padStart(2, "0") + ":00"));

}

$: {
    d3.select(yAxisGridlines).call(
        d3.axisLeft(yScale)
          .tickFormat("")
          .tickSize(-usableArea.width)
    );
}


$: timeScale = d3.scaleTime().domain([minDate,maxDate]).range([0,100]);
$: commitMaxTime = timeScale.invert(commitProgress);


$: rScale = d3.scaleSqrt()
                .domain(d3.extent(commits.map(d=>d.totalLines)))
                .range([2, 30]);



function dotInteraction (index, evt) {
    if (evt.type === "mouseenter") {
        hoveredIndex = index;
        cursor = {x: evt.x, y: evt.y};
    }
    else if (evt.type === "mouseleave") {
        hoveredIndex = -1
    }

    else if (evt.type === "click") {
        let commit = filteredCommits[index]
        if (!clickedCommits.includes(commit)) {
            // Add the commit to the clickedCommits array
            clickedCommits = [...clickedCommits, commit];
        }
        else {
                // Remove the commit from the array
                clickedCommits = clickedCommits.filter(c => c !== commit);
        }
}

}


</script>

<p>Total lines of code: {data.length}</p>

<dl class="info tooltip" hidden={hoveredIndex === -1} style="top: {cursor.y}px; left: {cursor.x}px">

    <dt>Commit</dt>
    <dd><a href="{ hoveredCommit.url }" target="_blank">{ hoveredCommit.id }</a></dd>

    <dt>Date</dt>
    <dd>{ hoveredCommit.datetime?.toLocaleString("en", {dateStyle: "full"}) }</dd>

    <dt>Author</dt>
    <dd>{ hoveredCommit.author }</dd>

    <dt>Time</dt>
    <dd>{ hoveredCommit.time }</dd>

    <!-- Add: Time, author, lines edited -->
</dl>

<div class="slider-container">
    <div class="slider">
        <label for="slider">Show commits until:</label>
        <input type="range" id="slider" name="slider" min=0 max=100 bind:value={commitProgress}/>
    </div>
    <time class="time-label">{commitMaxTime.toLocaleString()}</time>
</div>


<Scrolly bind:progress={ commitProgress }>
	{#each commits as commit, index }
	<p>
		On {commit.datetime.toLocaleString("en", {dateStyle: "full", timeStyle: "short"})},
		{index === 0 
			? "I set forth on my very first commit, beginning a magical journey of code. You can view it "
			: "I added another commit. See it "}
		<a href="{commit.url}" target="_blank">
			{index === 0 ? "here" : "here"}
		</a>.
		This update transformed {commit.totalLines} lines across { d3.rollups(commit.lines, D => D.length, d => d.file).length } files.
		With every commit, our project grows.
	</p>
{/each}

	<svelte:fragment slot="viz">
		<svg viewBox="0 0 {width} {height}">

        <FileLines lines={filteredData} width={width}/>
        <!-- // <FileLines lines={filteredLines} width={width} colorScale={colorScale} />; -->

        <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
        <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
        <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />


        <g class="dots">
            {#each filteredCommits as commit, index (commit.id) }

            <circle
            class:selected={ clickedCommits.includes(commit) }
            on:mouseenter={evt => dotInteraction(index, evt)}
            on:mouseleave={evt => dotInteraction(index, evt)}
            on:click={ evt => dotInteraction(index, evt) }
            cx={ xScale(commit.datetime) }
            cy={ yScale(commit.hourFrac) }
            r={ rScale(commit.totalLines) }
            fill="steelblue"
            fill-opacity="0.5"
        />

    {/each}
    </g> 
</svg>


<Bar data={languageBreakdown} width={width} />


	</svelte:fragment>
</Scrolly>




<!-- <section>
    <h2>Summary</h2>
    <dl class="stats">
    <dt>Total <abbr title="Lines of code">LOC</abbr></dt>
    <dd>{filteredLines.length}</dd>
    <dt>Files</dt>
    <dd>{d3.groups(filteredLines, d => d.file).length}</dd>
    <dt>Commits</dt>
    <dd>{d3.groups(filteredLines, d => d.commit).length}</dd>
    </dl>
</section> -->



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

    
    .info dt{
        grid-column:1;
        grid-row:auto;
    }
    
    .info dd{
        grid-column:2;
        grid-row:auto;
        font-weight: 400;
    }
    
    .tooltip{
        position: fixed;
        top: 1em;
        left: 1em;
    }
    
    .gridlines {
        stroke-opacity: .2;
    }
    
    .info{
        display: grid;
        margin:0;
        grid-template-columns: 2;
        background-color: oklch(100% 0% 0 / 80%);
        box-shadow: 1px 1px 3px 3px gray;
        border-radius: 5px;
        backdrop-filter: blur(10px);
        padding:10px;
    
        /* ... other styles ... */
        transition-duration: 500ms;
        transition-property: opacity, visibility;
    
        &[hidden]:not(:hover, :focus-within) {
            opacity: 0;
            visibility: hidden;
        }
    }
    circle {
        transition: 200ms;
        transform-origin: center;
        transform-box: fill-box;
    
    
    
        &:hover {
            transform: scale(1.5);
        }

        @starting-style {
            r: 0;
        }

    }

    .slider-container{
	display:grid;
    }
    .slider{
        display: flex;
    }
    #slider{
        flex:1;
    }
    .time-label{
        font-size: 0.75em;
        text-align: right;
    }


</style>

