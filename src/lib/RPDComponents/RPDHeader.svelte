<script lang="ts">
  import 'carbon-components-svelte/css/all.css';
  import { Link } from "carbon-components-svelte";
  import { ChevronRight, ChevronLeft, ArrowUp, ArrowUpRight, ArrowLeft, ArrowRight, Help, Share, Launch } from 'carbon-icons-svelte';
  import { createEventDispatcher } from 'svelte';
  import ShareDrawer from './ShareDrawer.svelte';

  const dispatch = createEventDispatcher();
  let activeView = 'year';
  let isShareDrawerOpen = false;

  function handleReadMore(event: MouseEvent) {
    event.preventDefault();
    console.log('Dispatching readMoreClick event');
    dispatch('readMoreClick');
  }

  function handleShare() {
    isShareDrawerOpen = true;
  }

  function closeShareDrawer() {
    isShareDrawerOpen = false;
  }
</script>

<div class="flex flex-col justify-start pl-12 pt-32">

  <div>
    <h1 class="capitalize font-light max-w-prose text-4xl mb-4 text-slate-600">Planting the seeds of rare disease research</h1>
    
    <p class="font-base max-w-prose">
      The FDA's Priority Review Voucher Program has planted the seeds for <span class="font-base text-orange-600 font-medium">56 new rare disease treatments</span>, with 42 first-ever therapies blooming where none existed before. 2024 has been the most fruitful year yet -- learn more and join us in asking that this garden is kept flourishing.
    
    <div class="flex gap-4 mt-12">
      <!-- Get Details Button -->
      <button 
        class="w-fit flex gap-4 pr-3 pl-4 py-3 items-start align-middle justify-start rounded-full text-xs font-semibold text-gray-100 relative overflow-hidden group" 
        on:click={handleReadMore}
      >
        <div class="absolute inset-0 bg-[#ff1515] z-0"></div>
        <div class="absolute inset-0 bg-sky-800 -translate-x-full group-hover:translate-x-0 transition-transform duration-600 ease-in-out z-0"></div>
        <span class="relative z-10 flex items-center gap-4">
          Get the Details 
          <ArrowUpRight size={16} />
        </span>
      </button>

      <!-- Share Button -->
      <button 
        class="w-fit flex gap-4 pr-3 pl-4 py-3 items-start align-middle justify-start rounded-full text-xs font-semibold text-gray-100 relative overflow-hidden group" 
        on:click={handleShare}
      >
        <div class="absolute inset-0 bg-sky-800 z-0"></div>
        <div class="absolute inset-0 bg-[#828CF0] -translate-x-full group-hover:translate-x-0 transition-transform duration-600 ease-in-out z-0"></div>
        <span class="relative z-10 flex items-center gap-4">
         Advocate for the Program
          <Share size={16} />
        </span>
      </button>

      <!-- Take Action Button -->
      <a 
        href="https://everylifefoundation.org/prv/"
        target="_blank"
        rel="noopener noreferrer"
        class="w-fit flex gap-4 pr-3 pl-4 py-3 items-start align-middle justify-start rounded-sm text-xs font-semibold text-gray-100 relative overflow-hidden group"
      >
      </a>
    </div>
  </div>
</div>

{#if isShareDrawerOpen}
  <ShareDrawer isOpen={isShareDrawerOpen} onClose={closeShareDrawer} />
{/if}

<style>
.rpd-header {
  border-bottom: 1px solid #e5e7eb;
  width: 100%;
}

.header-content {
  display: flex;
  padding: 2.25rem 0 1.25rem 0.25rem;
  align-items: left;
  justify-content: left;
}

.header-left {
  display: flex;
  align-items: left;
}

.title-section h1 {
  font-size: 1.25rem;
  font-weight: 400;
  color: #0c0c0c;
}

.title-section p {
  font-size: 0.875rem;
  color: #666;
  margin: 0;
}

.header-right {
  display: flex;
  align-items: center;
}

/* Ensure all button text is white for better contrast */
:global(.group:hover .text-orange-600) {
  color: white;
}
</style>