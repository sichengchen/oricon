<script>
  import { createEventDispatcher } from 'svelte'

  export let isPlaying = false
  export let disabled = false
  export let speed = 900
  export let baseSpeed = 900
  export let onPrev = () => {}
  export let onToggle = () => {}
  export let onNext = () => {}
  export let isMobile = false
  export let dates = []
  export let currentIndex = 0

  const dispatch = createEventDispatcher()

  let speedOptions = []

  $: speedOptions = [
    { label: '1x', value: baseSpeed },
    { label: '2x', value: Math.round(baseSpeed / 2) },
    { label: '4x', value: Math.round(baseSpeed / 4) },
    { label: '8x', value: Math.round(baseSpeed / 8) },
    { label: '0.5x', value: Math.round(baseSpeed * 2) }
  ]

  const getSpeedIndex = (value) => speedOptions.findIndex((option) => option.value === value)

  const cycleSpeed = () => {
    const speedIndex = getSpeedIndex(speed)
    const nextIndex = speedIndex === -1 ? 0 : (speedIndex + 1) % speedOptions.length
    speed = speedOptions[nextIndex].value
  }

  $: speedLabel = (speedOptions.find((option) => option.value === speed) || speedOptions[0]).label
  $: maxIndex = Math.max(0, dates.length - 1)
  $: currentLabel = dates[currentIndex] || 'Loading…'

  const handleInput = (event) => {
    const value = Number(event.target.value)
    currentIndex = value
    dispatch('scrub', { value })
  }
</script>

<section class="flex flex-wrap items-center gap-1 md:gap-4">
  <button class="btn btn-ghost" on:click={onPrev} disabled={disabled}>
    Prev
  </button>
  <button class="btn btn-primary" on:click={onToggle} disabled={disabled}>
    {isPlaying ? 'Pause' : 'Resume'}
  </button>
  <button class="btn btn-ghost" on:click={onNext} disabled={disabled}>
    Next
  </button>

  <button class="btn btn-ghost ml-auto group items-baseline" type="button" on:click={cycleSpeed} disabled={disabled}>
    {#if !isMobile}
      <span class="text-xs font-semibold uppercase mr-1 tracking-[0.08em] text-slate-600 transition-colors group-hover:text-white">
        Speed
      </span>
    {/if}
    <span class="font-semibold">{speedLabel}</span>
  </button>

</section>

<section class="flex flex-wrap items-center gap-4">
  {#if isMobile}
    <div class="pill">
      <span class="text-xs font-semibold uppercase tracking-[0.08em] text-slate-600">Week</span>
      <span class="font-semibold">{currentLabel}</span>
    </div>
  {/if}

  {#if !isMobile}
    <span class="text-xs font-semibold text-slate-600">
      {dates[0] || 'Loading…'}
    </span>
  {/if}

  <input
    class="range flex-1"
    type="range"
    min="0"
    max={maxIndex}
    step="1"
    value={currentIndex}
    disabled={disabled}
    on:input={handleInput}
  />

  {#if !isMobile}
    <span class="text-xs font-semibold text-slate-600">
      {dates[dates.length - 1] || 'Loading…'}
    </span>
  {/if}
</section>
