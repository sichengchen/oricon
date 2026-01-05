<script>
  import { slide } from 'svelte/transition'
  import { ChevronUp, Menu } from 'lucide-svelte'

  export let topN = 12
  export let currentDate = ''
  export let isPlaying = false
  export let onTogglePlay = () => {}
  export let isMobile = false

  let expanded = false
</script>

<header
  class="grid gap-4 rounded-3xl border border-slate-900 bg-[#f8f2f1] p-3 shadow-[0_18px_36px_rgba(15,23,42,0.12)] transition-all duration-300 sm:p-4"  class:md:max-w-5xl={!expanded}
  class:mx-auto={!expanded && !isMobile}
  class:w-full={!expanded && isMobile}
>
  <div class="flex flex-nowrap items-center sm:gap-1 md:gap-4 sm:flex-wrap">
    <button
      class="flex h-10 w-10 items-center justify-center rounded-full text-slate-900 transition hover:text-slate-700"
      aria-label="Toggle header"
      aria-expanded={expanded}
      on:click={() => (expanded = !expanded)}
    >
      {#if expanded}
        <ChevronUp size={22} strokeWidth={2} aria-hidden="true" />
      {:else}
        <Menu size={22} strokeWidth={2} aria-hidden="true" />
      {/if}
    </button>

    <h1 class="font-display text-xl uppercase font-bold tracking-[0.08em] sm:text-2xl md:text-4xl">
      {#if expanded}
        Oricon Weekly Top {topN} Singles
      {:else}
        Oricon Charts
      {/if}
      {#if expanded && !isMobile}
        <sup class="align-super font-sans text-[10px] font-semibold tracking-[0.08em] text-slate-600">
          1968-2008
        </sup>
      {/if}
      {#if !expanded}
        <sup class="align-super font-sans text-[10px] font-semibold tracking-[0.08em] text-slate-600">
          {currentDate || 'Loading…'}
        </sup>
      {/if}
    </h1>

    <div class="flex-1"></div>

    {#if expanded && !isMobile}
      <div class="text-right">
        <div class="mb-1 text-sm font-medium text-slate-600">Current Week</div>
        <div
          class="font-semibold"
          class:text-lg={!expanded}
          class:text-xl={expanded}
          class:md:text-2xl={expanded}
        >
          {currentDate || 'Loading…'}
        </div>
      </div>
    {/if}

    {#if !expanded}
      <button class="btn btn-primary h-10" on:click={onTogglePlay}>
        {isPlaying ? 'Pause' : 'Resume'}
      </button>
    {/if}
  </div>

  {#if expanded}
    <div class="grid gap-4" transition:slide={{ duration: 220 }}>
      <slot />
    </div>
  {/if}
</header>
