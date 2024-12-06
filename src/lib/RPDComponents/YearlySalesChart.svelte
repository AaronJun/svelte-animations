<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  
  export let constellationData: any[];
  const dispatch = createEventDispatcher();

  let svg;
  let chartWidth: number;
  let chartHeight: number;
  const margin = { top: 40, right: 45, bottom: 40, left: 60 };
  let width: number;
  let height: number;

  // Same tooltip state as before...
  let tooltipVisible = false;
  let tooltipBorderColor: string = '';
  let tooltipX = 0;
  let tooltipY = 0;
  let tooltipContent = {
    drugName: '',
    therapeuticArea: '',
    indication: '',
    price: '',
    seller: '',
    buyer: '',
    purchaseDate: ''
  };

  $: {
    width = chartWidth - margin.left - margin.right;
    height = chartHeight - margin.top - margin.bottom;
  }

  function getColorForTherapeuticArea(ta: string): string {
    const colorMap = {
    "Gastroenterology": "#4CAE3B",
      "Neurology": "#4D4DFF",
      "Ophthalmology": "#E79028",
      "Immunology": "#EA38A5",
      "Metabolic": "#133B11",
      "Dermatology": "#559368",
      "Hematology": "#CF3630",
      "Orthopedics": "#441780",
      "Respiratory": "#CBC09F",
      "Nephrology": "#ACA3DB",
      "Oncology": "#FF84DE",
      "Hepatology": "#FF00D4",
    };
    return colorMap[ta] || "#000000";
  }

  function processTransactionData(salesData) {
    // Create array of all years from 2012 to 2024
    const allYears = Array.from({length: 13}, (_, i) => 2012 + i);
    
    // First, sort the data by year and price
    const sortedData = [...salesData].sort((a, b) => {
      if (a.purchaseYear !== b.purchaseYear) {
        return a.purchaseYear - b.purchaseYear;
      }
      // Put undisclosed prices at the top of each year's stack
      if (a.price === null && b.price !== null) return -1;
      if (a.price !== null && b.price === null) return 1;
      return (b.price || 0) - (a.price || 0);
    });

    // Group data by purchase year
    const yearGroups = d3.group(sortedData, d => d.purchaseYear);
    
    // Process each year, including years with no transactions
    return allYears.map(year => {
      const transactions = yearGroups.get(year) || [];
      let cumHeight = 0;
      
      const processedTransactions = transactions.map(t => {
        const transaction = {
          ...t,
          y0: cumHeight,
          y1: t.price === null ? cumHeight + 85 : cumHeight + t.price, // Use fixed height for undisclosed prices
          originalData: constellationData.find(d => 
            d["Drug Name"] === t.drugName && 
            d["Purchase Year"] === year.toString()
          )
        };
        cumHeight = transaction.y1;
        return transaction;
      });

      return {
        year,
        transactions: processedTransactions,
        totalValue: cumHeight || 0 // Ensure we return 0 for years with no transactions
      };
    });
  }

  function handleClick(transactionData) {
    if (transactionData.originalData) {
      dispatch('clusterElementClick', {
        entry: transactionData.originalData,
        color: getColorForTherapeuticArea(transactionData.originalData.name)
      });
    }
  }

  function createChart() {
    if (!width || !height) return;

    // Process the data - FIXED: Include all purchased entries regardless of price
    const salesData = constellationData
      .filter(d => d.Purchased === "Y") // Only filter for purchased entries
      .map(d => {
        // Parse the price, but allow for null values
        let price = null;
        if (d["Sale  Price (USD, Millions)"]) {
          const parsedPrice = parseFloat(d["Sale  Price (USD, Millions)"]);
          if (!isNaN(parsedPrice)) {
            price = parsedPrice;
          }
        }

        return {
          purchaseYear: +d["Purchase Year"],
          price: price, // This will be null for undisclosed prices
          drugName: d["Drug Name"],
          buyer: d.Purchaser,
          seller: d.Sponsor,
          therapeuticArea: d.name,
          indication: d.id,
          purchaseDate: d["Purchase Month"] && d["Purchase Date"] && d["Purchase Year"] ? 
            `${d["Purchase Month"]} ${d["Purchase Date"]}, ${d["Purchase Year"]}` : 'N/A'
        };
      });

    // Process into stacked data
    const stackedData = processTransactionData(salesData);

    // Clear previous chart
    d3.select(svg).selectAll("*").remove();

    // Create scales - now using all years
    const xScale = d3.scaleBand()
      .domain(stackedData.map(d => d.year.toString()))
      .range([0, width])
      .padding(0.1);

    const maxValue = d3.max(stackedData, d => d.totalValue);
    const yScale = d3.scaleLinear()
      .domain([0, maxValue])
      .range([height, 0])
      .nice();

    // Create SVG and append groups
    const svgElement = d3.select(svg)
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom);

    const g = svgElement.append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Add grid lines
    g.append("g")
      .attr("class", "grid")
      .call(d3.axisLeft(yScale)
        .tickSize(-width)
        .tickFormat("")
      )
      .style("stroke-dasharray", "2,3")
      .style("stroke-opacity", 0.2);

    // Create the stacked bars
    const yearGroups = g.selectAll(".year-group")
      .data(stackedData)
      .join("g")
      .attr("class", "year-group")
      .attr("transform", d => `translate(${xScale(d.year.toString())},0)`);

    yearGroups.selectAll("rect")
      .data(d => d.transactions)
      .join("rect")
      .attr("x", 0)
      .attr("y", d => yScale(d.y1))
      .attr("width", xScale.bandwidth())
      .attr("height", d => yScale(d.y0) - yScale(d.y1))
      .attr("fill", d => {
        if (d.price === null) {
          return "#f5f5f5"; // Light grey fill for undisclosed prices
        }
        return getColorForTherapeuticArea(d.therapeuticArea);
      })
      .attr("stroke", d => {
        if (d.price === null) {
          return getColorForTherapeuticArea(d.therapeuticArea);
        }
        return "#fff";
      })
      .attr("stroke-width", d => d.price === null ? 2 : 1)
      .attr("stroke-dasharray", d => d.price === null ? "4,4" : "none")
      .attr("opacity", d => d.price === null ? 0.9 : 0.8)
      .style("cursor", "pointer")
      
      .on("mouseover", (event, d) => {
        tooltipContent = {
          drugName: d.drugName,
          therapeuticArea: d.therapeuticArea,
          indication: d.indication,
          price: d.price === null ? 'Undisclosed' : `$${d.price.toLocaleString()}M`,
          seller: d.seller,
          buyer: d.buyer,
          purchaseDate: d.purchaseDate
        };
        tooltipX = event.pageX;
        tooltipY = event.pageY;
        tooltipBorderColor = getColorForTherapeuticArea(d.therapeuticArea);
        tooltipVisible = true;

        d3.select(event.target)
          .attr("opacity", d.price === null ? 0.9 : 0.8)
          .attr("stroke-width", 2)
          .attr("stroke", "#ff1515")
          .attr("stroke-dasharray", d.price === null ? "1,2.5" : "none");
      })

      .on("mouseout", (event, d) => {
        tooltipVisible = false;
        d3.select(event.target)
          .attr("opacity", d.price === null ? 0.9 : 0.8)
          .attr("stroke-width", d.price === null ? 2 : 1)
          .attr("stroke", d.price === null ? 
            getColorForTherapeuticArea(d.therapeuticArea) : "#fff")
          .attr("stroke-dasharray", d.price === null ? "2,2" : "none");
      })
      .on("click", (event, d) => {
        event.stopPropagation();
        handleClick(d);
      });

    // Add axes
    g.append("g")
      .attr("transform", `translate(0,${height})`)
      .call(d3.axisBottom(xScale))
      .style("font-size", "8px");

      g.append("g")
  .call(d3.axisLeft(yScale)
    .tickFormat(d => `${d}M`))
  .call(g => {
    g.selectAll(".tick text") // Select all tick text elements
      .style("font-size", "8px")
      .style("font-weight", "medium");
    g.selectAll(".tick line") // Optional: style the tick lines if needed
      .style("stroke", "#666")
      .style("stroke-width", "0.5");
    g.select(".domain") // Optional: style the axis line if needed
      .style("stroke", "#666")
      .style("stroke-width", "0.5");
  });

    // Add labels
    g.append("text")
      .attr("transform", "rotate(-90)")
      .attr("y", -margin.left + 10)
      .attr("x", -height / 2)
      .attr("text-anchor", "middle")
      .style("fill", "#3B665D")
      .style("font-size", "9px")
      .attr("font-weight", "medium")
      .text("Transaction Value (USD Millions)");
  }

  $: if (svg && width && height) {
    createChart();
  }

  onMount(() => {
    const resizeObserver = new ResizeObserver(entries => {
      for (const entry of entries) {
        const { width, height } = entry.contentRect;
        chartWidth = width;
        chartHeight = height;
      }
    });

    const container = svg.parentElement;
    resizeObserver.observe(container);

    return () => {
      resizeObserver.disconnect();
    };
  });
</script>

<div class="chart-container">
  <div class="chart">
    <svg bind:this={svg}></svg>
  </div>
</div>

{#if tooltipVisible}
  <div
    class="tooltip p-2 pt-1"
    style="left: {tooltipX - 290}px; top: {tooltipY - 220}px; border-top: 0.625rem solid {tooltipBorderColor};"
  >
    <div>
        <p class="text-lg font-medium">
          {tooltipContent.drugName}
        </p>
        <p class="text-xs uppercase mb-4 font-mono font-semibold text-gray-600">
          {tooltipContent.therapeuticArea}
        </p>
    </div>
    <p class="text-base font-medium">
      {tooltipContent.price}
    </p>
    <p class="text-xs font-semibold font-mono text-gray-400">Transaction Value</p>
    <div class="entry-bottom mt-4">
      <p class="text-xs font-semibold font-mono mt-2 text-gray-800">
        {tooltipContent.seller} → {tooltipContent.buyer}
      </p>
      <p class="text-xs font-semibold font-mono text-gray-400">
        {tooltipContent.purchaseDate}
      </p>
      <p class="text-xs font-semibold font-mono mt-6 text-green-600">
        Click to view more details →
      </p>
    </div>
  </div>
{/if}

<style>
  .chart-container {
    width: 97.25%;
    overflow: scroll;
    height: 650px;
    margin: 2rem 0;
  }

  .chart {
    width: 100%;
    height: calc(100% - 4rem);
    overflow: auto;
  }

  :global(.grid line) {
    stroke: #e5e7eb;
    stroke-width: .25px;
  }

  :global(.grid path) {
    stroke-width: 0;
  }

  .tooltip {
    position: fixed;
    background-color: rgba(255, 255, 255, 0.962);    
    border: .5px solid #373737;
    pointer-events: none;
    z-index: 1000;
    min-width: 300px;
    max-width: 300px;
  }
</style>