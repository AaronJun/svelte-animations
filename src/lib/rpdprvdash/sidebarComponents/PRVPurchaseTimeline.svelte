<!-- PRVPurchaseTimeline.svelte - Timeline focused on transaction values by year -->
<script lang="ts">
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import { getTherapeuticAreaColor } from '../utils/colorDefinitions';
    
    export let data: any[] = [];
    export let onYearSelect: (year: string) => void;
    export let selectedYear: string | null = null;
    export let transactionYearSelected: (year: string) => void = () => {};
    
    let svg: SVGElement;
    const margin = { top: 20, right: 20, bottom: 20, left: 40 };
    const width = 125;
    const height = 800;

    // Process data to filter for purchases with transaction values
    $: purchaseData = data.filter(entry => entry.Purchased === "Y" && entry["Purchase Year"]);
    
    // Process data to group by purchase year
    $: yearData = Object.entries(
        purchaseData.reduce((acc, entry) => {
            const year = entry["Purchase Year"];
            if (!year) return acc;
            
            // Parse sale price
            let salePrice = 0;
            if (entry["Sale Price (USD Millions)"] && entry["Sale Price (USD Millions)"] !== "Undisclosed") {
                salePrice = parseFloat(entry["Sale Price (USD Millions)"]);
            }
            
            if (!acc[year]) {
                acc[year] = {
                    count: 0,
                    totalValue: 0,
                    areas: {}
                };
            }
            acc[year].count += 1;
            acc[year].totalValue += salePrice;
            
            const area = entry.TherapeuticArea1;
            if (area) {
                if (!acc[year].areas[area]) {
                    acc[year].areas[area] = {
                        count: 0,
                        value: 0
                    };
                }
                acc[year].areas[area].count += 1;
                acc[year].areas[area].value += salePrice;
            }
            return acc;
        }, {} as Record<string, { 
            count: number; 
            totalValue: number;
            areas: Record<string, { count: number; value: number }> 
        }>)
    )
    .map(([year, data]) => {
        // Calculate area percentages based on value
        const areaEntries = Object.entries(data.areas)
            .map(([area, stats]) => ({
                area,
                count: stats.count,
                value: stats.value,
                percentage: data.totalValue > 0 ? stats.value / data.totalValue : stats.count / data.count
            }))
            .sort((a, b) => b.value - a.value);
            
        return {
            year,
            count: data.count,
            totalValue: data.totalValue,
            areas: areaEntries
        };
    })
    .sort((a, b) => a.year.localeCompare(b.year));

    function createGradientId(year: string): string {
        return `purchase-gradient-${year}`;
    }

    function createVisualization() {
        if (!svg || !yearData.length) return;

        const svgElement = d3.select(svg);
        svgElement.selectAll("*").remove();

        const innerWidth = width - margin.left - margin.right;
        const innerHeight = height - margin.top - margin.bottom;

        // Create y scale (reversed for top-to-bottom order)
        const yScale = d3.scalePoint()
            .domain(yearData.map(d => d.year))
            .range([margin.top, innerHeight - 60]) // Leave space at bottom for "All Years"
            .padding(0.5);

        // Scale for circle radius based on total transaction value
        const radiusScale = d3.scaleSqrt()
            .domain([0, d3.max(yearData, d => d.totalValue) || 0])
            .range([1, 20]);

        const g = svgElement.append("g")
            .attr("transform", `translate(${margin.left},0)`);

        // Create gradients
        const defs = svgElement.append("defs");
        yearData.forEach(yearEntry => {
            const gradient = defs.append("linearGradient")
                .attr("id", createGradientId(yearEntry.year))
                .attr("x1", "0%")
                .attr("y1", "0%")
                .attr("x2", "100%")
                .attr("y2", "100%");

            let currentPosition = 0;
            yearEntry.areas.forEach(area => {
                const colorCombo = getTherapeuticAreaColor(area.area);
                
                gradient.append("stop")
                    .attr("offset", `${currentPosition * 100}%`)
                    .attr("stop-color", colorCombo.fill);

                currentPosition += area.percentage;

                gradient.append("stop")
                    .attr("offset", `${currentPosition * 100}%`)
                    .attr("stop-color", colorCombo.fill);
            });
        });
        
        // Create a special gradient for "All Years"
        const allYearsGradient = defs.append("linearGradient")
            .attr("id", "purchase-gradient-all-years")
            .attr("x1", "10%")
            .attr("y1", "20%")
            .attr("x2", "100%")
            .attr("y2", "100%");
            
        // Get all unique therapeutic areas
        const allAreas = new Set();
        yearData.forEach(yearEntry => {
            yearEntry.areas.forEach(area => {
                allAreas.add(area.area);
            });
        });
        
        // Add all therapeutic areas to the gradient
        const uniqueAreas = Array.from(allAreas);
        uniqueAreas.forEach((area, index) => {
            const colorCombo = getTherapeuticAreaColor(area);
            const position = index / uniqueAreas.length;
            
            allYearsGradient.append("stop")
                .attr("offset", `${position * 100}%`)
                .attr("stop-color", colorCombo.fill);
                
            if (index < uniqueAreas.length - 1) {
                allYearsGradient.append("stop")
                    .attr("offset", `${(position + 1/uniqueAreas.length) * 100}%`)
                    .attr("stop-color", colorCombo.fill);
            }
        });

        // Create glow filter
        const filter = defs.append("filter")
            .attr("id", "purchase-glow")
            .attr("x", "-50%")
            .attr("y", "-50%")
            .attr("width", "400%")
            .attr("height", "400%");

        filter.append("feGaussianBlur")
            .attr("stdDeviation", "2")
            .attr("result", "coloredBlur");

        const feMerge = filter.append("feMerge");
        feMerge.append("feMergeNode")
            .attr("in", "coloredBlur");
        feMerge.append("feMergeNode")
            .attr("in", "SourceGraphic");

        // Add connecting line
        g.append("line")
            .attr("x1", innerWidth / 2)
            .attr("x2", innerWidth / 2)
            .attr("y1", margin.top - 10)
            .attr("y2", innerHeight - 10)
            .attr("stroke", "#666666")
            .attr("stroke-width", 0.5)
            .attr("stroke-dasharray", "3,3");

        // Create year groups
        const yearGroups = g.selectAll(".year-group")
            .data(yearData)
            .join("g")
            .attr("class", "year-group")
            .attr("transform", d => `translate(${innerWidth / 2},${yScale(d.year)})`);

        // Add highlight circles
        yearGroups.append("circle")
            .attr("class", "highlight-circle")
            .attr("r", d => radiusScale(d.totalValue) + 4)
            .attr("fill", "none")
            .attr("stroke", "#55D88E") // Using orange for purchase highlight
            .attr("stroke-width", 5)
            .attr("opacity", 0);

        // Add main circles
        yearGroups.append("circle")
            .attr("class", "year-circle")
            .attr("r", d => radiusScale(d.totalValue))
            .attr("fill", d => `url(#${createGradientId(d.year)})`)
            .attr("stroke", "#565656")
            .attr("stroke-width", 1.5)
            .style("cursor", "pointer")
            .on("click", (event, d) => {
                selectedYear = d.year;
                onYearSelect(d.year);
                transactionYearSelected(d.year);
                updateSelection();
            })
            .on("mouseenter", function(event, d) {
                if (d.year !== selectedYear) {
                    d3.select(this.parentNode)
                        .select(".highlight-circle")
                        .transition()
                        .duration(200)
                        .attr("opacity", 0.325);

                    d3.select(this)
                        .transition()
                        .duration(200)
                        .attr("stroke-width", 2)
                        .style("filter", "url(#purchase-glow)");

                    d3.select(this.parentNode)
                        .select(".year-label")
                        .transition()
                        .duration(200)
                        .attr("font-weight", "600")
                        .attr("fill", "#FF9F1C");
                }
            })
            .on("mouseleave", function(event, d) {
                if (d.year !== selectedYear) {
                    d3.select(this.parentNode)
                        .select(".highlight-circle")
                        .transition()
                        .duration(200)
                        .attr("opacity", 0);

                    d3.select(this)
                        .transition()
                        .duration(200)
                        .attr("stroke-width", 1.5)
                        .style("filter", "none");

                    d3.select(this.parentNode)
                        .select(".year-label")
                        .transition()
                        .duration(200)
                        .attr("font-weight", "400")
                        .attr("fill", "#718096");
                }
            });

        // Add year labels
        yearGroups.append("text")
            .attr("class", "year-label")
            .attr("x", -radiusScale(d3.max(yearData, d => d.count) || 0))
            .attr("y", -42) // Center vertically
            .attr("text-anchor", "middle")
            .attr("transform", "rotate(-90)")
            .attr("fill", "#718096")
            .attr("font-size", "9.75px")
            .style("dominant-baseline", "end")
            .style("font-family", "'IBM Plex Mono', monospace")
            .text(d => d.year);
            
        // Create "All Years" circle at the bottom
        const allYearsGroup = g.append("g")
            .attr("class", "all-years-group")
            .attr("transform", `translate(${innerWidth / 2}, ${innerHeight + 10})`)
            .attr("cursor", "pointer");
            
        // Create highlight circle for "All Years"
        allYearsGroup.append("circle")
            .attr("class", "highlight-circle")
            .attr("r", 25) // Larger than regular circles
            .attr("fill", "none")
            .attr("stroke", "#55D88E")
            .attr("stroke-width", 5)
            .attr("opacity", selectedYear === "All" ? 0.5 : 0);
            
        // Create main circle for "All Years"
        allYearsGroup.append("circle")
            .attr("class", "year-circle")
            .attr("r", 22) // Larger than regular circles
            .attr("fill", "url(#purchase-gradient-all-years)")
            .attr("stroke", "#BC9533")
            .attr("stroke-width", selectedYear === "All" ? 4 : 2.5)
            .style("filter", selectedYear === "All" ? "url(#purchase-glow)" : "none")
            .style("cursor", "pointer")
            .on("click", () => {
                selectedYear = "All";
                onYearSelect("All");
                transactionYearSelected("All");
                updateSelection();
            })
            .on("mouseenter", function() {
                if (selectedYear !== "All") {
                    allYearsGroup.select(".highlight-circle")
                        .transition()
                        .duration(200)
                        .attr("opacity", 0.325);
                        
                    d3.select(this)
                        .transition()
                        .duration(200)
                        .attr("stroke-width", 2)
                        .style("filter", "url(#purchase-glow)");
                        
                    allYearsGroup.select(".year-label")
                        .transition()
                        .duration(200)
                        .attr("font-weight", "600")
                        .attr("fill", "#FF9F1C");
                }
            })
            .on("mouseleave", function() {
                if (selectedYear !== "All") {
                    allYearsGroup.select(".highlight-circle")
                        .transition()
                        .duration(200)
                        .attr("opacity", 0);
                        
                    d3.select(this)
                        .transition()
                        .duration(200)
                        .attr("stroke-width", 1.5)
                        .style("filter", "none");
                        
                    allYearsGroup.select(".year-label")
                        .transition()
                        .duration(200)
                        .attr("font-weight", "400")
                        .attr("fill", "#718096");
                }
            });
            
        // Add "All Years" label
        allYearsGroup.append("text")
            .attr("class", "year-label")
            .attr("y", -42) // Center vertically
            .attr("text-anchor", "middle")
            .attr("transform", "rotate(-90)")
            .attr("fill", "#718096")
            .attr("font-size", "9.75px")
            .style("dominant-baseline", "end")
            .style("font-family", "'IBM Plex Mono', monospace")
            .text("All");

        updateSelection();
    }

    function updateSelection() {
        if (!svg) return;

        // Update "All Years" circle state
        const allYearsGroup = d3.select(svg).select(".all-years-group");
        if (!allYearsGroup.empty()) {
            allYearsGroup.select(".highlight-circle")
                .transition()
                .duration(300)
                .attr("opacity", selectedYear === "All" ? 0.5 : 0);
                
            allYearsGroup.select(".year-circle")
                .transition()
                .duration(300)
                .attr("stroke-width", selectedYear === "All" ? 3 : 1.5)
                .style("filter", selectedYear === "All" ? "url(#purchase-glow)" : "none");
                
            allYearsGroup.select(".year-label")
                .transition()
                .duration(300)
                .attr("font-weight", selectedYear === "All" ? "600" : "400")
                .attr("fill", selectedYear === "All" ? "#FF1515" : "#718096");
        }

        // Update year circles state
        d3.select(svg)
            .selectAll(".year-group")
            .each(function(d: any) {
                const group = d3.select(this);
                const isSelected = d.year === selectedYear;

                group.select(".highlight-circle")
                    .transition()
                    .duration(300)
                    .attr("opacity", isSelected ? 0.5 : 0);

                group.select(".year-circle")
                    .transition()
                    .duration(300)
                    .attr("stroke-width", isSelected ? 3 : 1.5)
                    .style("filter", isSelected ? "url(#purchase-glow)" : "none");

                group.select(".year-label")
                    .transition()
                    .duration(300)
                    .attr("font-weight", isSelected ? "600" : "400")
                    .attr("fill", isSelected ? "#FF1515" : "#718096");
            });
    }

    $: if (data.length > 0 && svg) {
        createVisualization();
    }
</script>

<div class="timeline-container">
    <div class="sidebar-header ml-2 flex gap-2 uppercase place-items-center">      
        <h4 class="text-sm capitalize font-medium text-slate-800">
            Select Year             
            </h4>
    </div>    
    <svg
        bind:this={svg}
        {width}
        {height}
        viewBox="0 0 {width} {height}"
        class="w-full h-auto"
    />
</div>


<style>
    .timeline-container {
        height: 100%;
        width: 100%;
        display: flex;
        flex-direction: column;
        position: relative;
    }
    

    .timeline-header {
        margin-bottom: 0.5rem;
        text-align: center;
        letter-spacing: 0.05em;
    }

    :global(.year-group) {
        transition: all 0.3s ease;
    }
</style>