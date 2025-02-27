<!-- VoucherBeeswarmPlot.svelte -->
<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  import { Medication, Money, CalendarHeatMap } from 'carbon-icons-svelte';

  export let data: any[];
  export let onPointClick: (detail: any) => void;
  export let highlightedTransaction: { seller: string, buyer: string } | null = null;
  
  const dispatch = createEventDispatcher<{
    transactionHover: { seller: string, buyer: string };
    transactionLeave: void;
  }>();
  
  let svg;
  let container;
  let circles;
  let tooltipVisible = false;
  let tooltipX = 0;
  let tooltipY = 0;
  let tooltipContent = {
    buyer: '',
    seller: '',
    price: '',
    candidate: '',
    date: '',
    therapeuticArea: '',
    isUndisclosed: false
  };
  
  const margin = { top: 20, right: 20, bottom: 40, left: 60 };
  let width: number;
  let height: number;

  const therapeuticAreaColorScale = d3.scaleOrdinal()
    .domain([
        'Neurology', 'Oncology', 'Metabolic', 'Ophthalmology',
        'Cardiovascular', 'Pulmonology', 'Hematology',
        'Endocrinology', 'Genetic', 'Immunology',
        'Gastroenterology', 'Hepatology', 'Dermatology',
        'Neonatology', 'Urology'
    ])
    .range([
        '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4',
        '#FFEEAD', '#D4A5A5', '#9DE0AD',
        '#FF9F1C', '#2EC4B6', '#E71D36',
        '#FDFFB6', '#CBE896', '#FFA07A',
        '#98D8C8', '#B8B8D1'
    ]);

  function getPrice(d: any): number {
    const price = d["Sale Price (USD Millions)"];
    return price === "Undisclosed" ? 0 : parseFloat(price);
  }

  function isUndisclosed(d: any): boolean {
    return d["Sale Price (USD Millions)"] === "Undisclosed";
  }

  function handlePointClick(event: MouseEvent, d: any) {
    event.stopPropagation();
    tooltipVisible = false;
    
    setTimeout(() => {
      onPointClick({
        drugName: d.Candidate,
        year: d["RPDD Year"],
        Company: d.Company,
        therapeuticArea: d.TherapeuticArea1,
        entries: data.filter(entry => entry.TherapeuticArea1 === d.TherapeuticArea1),
        color: therapeuticAreaColorScale(d.TherapeuticArea1),
        currentStage: d["Current Development Stage"] || "TBD",
        indication: d.Indication || "",
        rpddAwardDate: d["RPDD Year"],
        voucherAwardDate: d["PRV Issue Year"] || "",
        treatmentClass: d.Class1 || "TBD",
        mechanismOfAction: d.MOA || "TBD",
        companyUrl: d["Link to CrunchBase"] || ""
      });
    }, 0);
  }

  $: if (highlightedTransaction && circles) {
    highlightPoints(highlightedTransaction);
  } else if (circles) {
    resetHighlight();
  }

  function highlightPoints(transaction: { seller: string, buyer: string }) {
    circles
      .style("opacity", d => 
        (d.Company === transaction.seller && d.Purchaser === transaction.buyer) ? 1 : 0.2
      )
      .attr("r", d => 
        (d.Company === transaction.seller && d.Purchaser === transaction.buyer) ? 6 : 5
      );
  }

  function resetHighlight() {
    if (!circles) return;
    circles
      .style("opacity", 0.8)
      .attr("r", 5);
  }
  
  function updateDimensions() {
    if (!container) return;
    
    const containerRect = container.getBoundingClientRect();
    width = containerRect.width - margin.left - margin.right;
    height = containerRect.height - margin.top - margin.bottom;
    
    if (svg) {
      svg.attr("width", width + margin.left + margin.right)
         .attr("height", height + margin.top + margin.bottom);
      
      createVisualization();
    }
  }
  
  function createVisualization() {
    // Filter and sort the data
    const purchasedVouchers = data
      .filter(d => d.Purchased === "Y")
      .sort((a, b) => d3.ascending(getPrice(a), getPrice(b)));
    
    const prices = purchasedVouchers.map(d => getPrice(d));
    
    // Clear previous content
    svg.selectAll("*").remove();
    
    const g = svg.append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Create scales
    const y = d3.scaleLinear()
      .domain([0, d3.max(prices)])
      .range([height, 0])
      .nice();

    const x = d3.scaleLinear()
      .domain([-1, 1])
      .range([0, width]);

    // Add y-axis with proper formatting
    const yAxis = d3.axisLeft(y)
      .tickFormat(d => `$${d}M`)
      .ticks(5);

    g.append("g")
      .attr("class", "y-axis")
      .call(yAxis);

    // Create simulation
    const simulation = d3.forceSimulation(purchasedVouchers)
      .force("y", d3.forceY(d => y(getPrice(d))).strength(1))
      .force("x", d3.forceX(width/2).strength(0.1))
      .force("collide", d3.forceCollide(6))
      .alphaDecay(0.1)
      .alpha(0.5)
      .stop();

    // Run simulation
    for (let i = 0; i < 300; ++i) simulation.tick();

    // Create circles with special styling for undisclosed prices
    circles = g.selectAll("circle")
      .data(purchasedVouchers)
      .join("circle")
      .attr("cx", d => Math.max(0, Math.min(width, d.x)))
      .attr("cy", d => Math.max(0, Math.min(height, d.y)))
      .attr("r", 5)
      .attr("fill", d => therapeuticAreaColorScale(d.TherapeuticArea1))
      .style("opacity", 0.8)
      .attr("cursor", "pointer")
      .style("stroke", "#565656")
      .style("stroke-width", 1)
      .style("stroke-dasharray", d => isUndisclosed(d) ? "2,2" : "none");

    // Add event listeners with debouncing
    let hoverTimeout: number;
    circles
      .on("mouseenter", (event, d) => {
        clearTimeout(hoverTimeout);
        hoverTimeout = setTimeout(() => {
          dispatch('transactionHover', {
            seller: d.Company,
            buyer: d.Purchaser
          });

          tooltipContent = {
            candidate: d.Candidate || 'N/A',
            seller: d.Company || 'N/A',
            buyer: d.Purchaser || 'N/A',
            price: isUndisclosed(d) ? 'Undisclosed' : `$${d["Sale Price (USD Millions)"]}M`,
            date: `${d["Purchase Month"]} ${d["Purchase Date"]}, ${d["Purchase Year"]}`,
            therapeuticArea: d.TherapeuticArea1,
            isUndisclosed: isUndisclosed(d)
          };

          const rect = container.getBoundingClientRect();
          tooltipX = event.clientX - rect.left;
          tooltipY = event.clientY - rect.top;
          tooltipVisible = true;
        }, 50);
      })
      .on("mouseleave", () => {
        clearTimeout(hoverTimeout);
        dispatch('transactionLeave');
        tooltipVisible = false;
      })
      .on("click", handlePointClick);
  }

  onMount(() => {
    svg = d3.select("#beeswarm-plot")
      .append("svg")
      .attr("class", "overflow-visible");
      
    updateDimensions();
    
    const resizeObserver = new ResizeObserver(() => {
      requestAnimationFrame(updateDimensions);
    });
    resizeObserver.observe(container);
    
    return () => {
      resizeObserver.disconnect();
    };
  });
</script>

<div 
  class="beeswarm-wrapper h-full"
  bind:this={container}
>
  <div id="beeswarm-plot" class="h-full relative"></div>

  {#if tooltipVisible}
    <div
      class="absolute z-10 bg-white p-4 rounded shadow-lg text-sm border border-slate-200 min-w-fit"
      style="left: {tooltipX}px; top: {tooltipY}px; transform: translate(-50%, -100%)"
    >
      <div class="font-semibold text-base text-slate-800 mb-4">{tooltipContent.seller} → {tooltipContent.buyer}</div>
      
      <div class="flex gap-4 text-slate-600 items-baseline">
        <Medication size="14" class="text-slate-600" />
        <p class="font-semibold text-sm">
          {tooltipContent.candidate}
        </p>
      </div>

      <div class="flex gap-4 items-baseline">
        <Money size="14" class="text-gray-800" />
        <p class="text-sm {tooltipContent.isUndisclosed ? 'italic' : ''}">
          {tooltipContent.price}        
        </p>
      </div>
    
      <div class="flex gap-4 items-baseline">
        <CalendarHeatMap size="14" class="text-gray-800" />
        <p class="text-sm">
          {tooltipContent.date}
        </p>
      </div>
    </div>
  {/if}
</div>

<style>
  #beeswarm-plot {
    width: 100%;
    min-height: 100%;
  }

  :global(.y-axis path) {
    stroke: #e2e8f0;
  }

  :global(.y-axis line) {
    stroke: #e2e8f0;
  }

  :global(.y-axis text) {
    fill: #4a5568;
    font-size: 10px;
  }
</style>