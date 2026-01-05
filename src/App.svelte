<script>
  import { onDestroy, onMount } from 'svelte'
  import * as d3 from 'd3'
  import Chart from './components/Chart.svelte'
  import Controls from './components/Controls.svelte'
  import Header from './components/Header.svelte'

  const dataUrl = '/data/oricon_singles.csv'

  const desktopMargin = { top: 16, right: 0, bottom: 16, left: 0 }
  const mobileMargin = { top: 12, right: 0, bottom: 12, left: 0 }
  const desktopBarHeight = 26
  const mobileBarHeight = 48
  const desktopBarGap = 8
  const mobileBarGap = 8
  const desktopChartWidth = 1000
  const mobileChartWidth = 360
  const desktopLabelOffset = 0
  const mobileLabelOffset = 0
  const desktopBarWidthScale = 1
  const mobileBarWidthScale = 1
  const baseSpeed = 900

  let dates = []
  let dataByDate = new Map()

  let currentIndex = 0
  let isPlaying = true
  let speed = baseSpeed
  let tickHandle = null
  let isMobile = false
  let chartContainerWidth = 0
  let mediaQuery = null
  let handleMediaChange = null
  let loadError = ''

  const parseNumber = (value) => {
    if (!value) return null
    const cleaned = String(value).replace(/[^\d]/g, '')
    return cleaned ? Number(cleaned) : null
  }

  const loadData = async () => {
    const rows = await d3.csv(dataUrl, (row) => ({
      chartDate: row.chart_date,
      rank: Number(row.rank),
      title: row.title,
      artist: row.artist,
      weeklySales: parseNumber(row.weekly_sales),
      totalSales: parseNumber(row.total_sales)
    }))

    const grouped = d3.group(rows, (d) => d.chartDate)
    dates = Array.from(grouped.keys()).sort()
    dataByDate = new Map(dates.map((date) => [date, grouped.get(date) || []]))
  }

  const stopPlayback = () => {
    isPlaying = false
    if (tickHandle) {
      clearTimeout(tickHandle)
      tickHandle = null
    }
  }

  const tick = () => {
    if (!isPlaying) return
    currentIndex = (currentIndex + 1) % dates.length
    tickHandle = setTimeout(tick, speed)
  }

  const startPlayback = () => {
    if (!dates.length) return
    if (isPlaying) return
    isPlaying = true
    tickHandle = setTimeout(tick, speed)
  }

  const togglePlayback = () => {
    if (isPlaying) {
      stopPlayback()
    } else {
      startPlayback()
    }
  }

  const moveBy = (delta) => {
    if (!dates.length) return
    stopPlayback()
    currentIndex = (currentIndex + delta + dates.length) % dates.length
  }

  const handleScrub = () => {
    stopPlayback()
  }

  onMount(() => {
    mediaQuery = window.matchMedia('(max-width: 640px)')
    handleMediaChange = () => {
      isMobile = mediaQuery.matches
    }
    handleMediaChange()
    if (mediaQuery.addEventListener) {
      mediaQuery.addEventListener('change', handleMediaChange)
    } else {
      mediaQuery.addListener(handleMediaChange)
    }

    const init = async () => {
      try {
        loadError = ''
        await loadData()
        if (dates.length && isPlaying) {
          tickHandle = setTimeout(tick, speed)
        }
      } catch (error) {
        loadError = error instanceof Error ? error.message : 'Failed to load data.'
        stopPlayback()
      }
    }
    init()
  })

  onDestroy(() => {
    stopPlayback()
    if (mediaQuery && handleMediaChange) {
      if (mediaQuery.removeEventListener) {
        mediaQuery.removeEventListener('change', handleMediaChange)
      } else {
        mediaQuery.removeListener(handleMediaChange)
      }
    }
  })

  $: currentDate = dates[currentIndex] || ''
  $: sortedItems = (dataByDate.get(currentDate) || [])
    .slice()
    .sort((a, b) => a.rank - b.rank)
  $: topItems = sortedItems
  $: topN = topItems.length
  $: maxRank = topItems.length || 1
  $: colorScale = d3
    .scaleLinear()
    .domain([1, Math.max(1, maxRank)])
    .range(['#d96a74', '#f2a3a5'])
  $: currentItems = topItems.map((entry, index) => ({
    ...entry,
    value: 1,
    index,
    key: `${entry.title}__${entry.artist}`,
    color: colorScale(entry.rank)
  }))
  $: margin = isMobile ? mobileMargin : desktopMargin
  $: barHeight = isMobile ? mobileBarHeight : desktopBarHeight
  $: barGap = isMobile ? mobileBarGap : desktopBarGap
  $: labelOffset = isMobile ? mobileLabelOffset : desktopLabelOffset
  $: barWidthScale = isMobile ? mobileBarWidthScale : desktopBarWidthScale
  $: chartWidth = isMobile
    ? Math.max(320, chartContainerWidth || mobileChartWidth)
    : desktopChartWidth
  $: innerWidth = chartWidth - margin.left - margin.right
  $: bandHeight = barHeight + barGap
  $: innerHeight = topItems.length * bandHeight
  $: chartHeight = margin.top + innerHeight + margin.bottom
  $: isDisabled = !dates.length || !!loadError
</script>

<main class="mx-auto grid min-h-screen max-w-5xl gap-6 px-4 py-6 md:px-10 md:py-10">
  <Header {topN} {currentDate} {isPlaying} onTogglePlay={togglePlayback} {isMobile}>
    <Controls
      bind:speed
      bind:currentIndex
      {dates}
      {baseSpeed}
      {isPlaying}
      disabled={isDisabled}
      onPrev={() => moveBy(-1)}
      onToggle={togglePlayback}
      onNext={() => moveBy(1)}
      {isMobile}
      on:scrub={handleScrub}
    />
  </Header>

  {#if loadError}
    <div class="card text-center text-sm text-slate-900">
      <div class="font-semibold">Failed to load chart data.</div>
      <div class="mt-1 text-xs text-slate-600">{loadError}</div>
    </div>
  {:else}
    <div bind:clientWidth={chartContainerWidth}>
      <Chart
        {currentItems}
        {chartWidth}
        {chartHeight}
        {innerWidth}
        {margin}
        {bandHeight}
        {barHeight}
        {labelOffset}
        {isMobile}
        {barWidthScale}
      />
    </div>
  {/if}

  <footer
    class="text-center text-xs"
  >
    Website by <a class="underline" href="https://scchan.moe" target="_blank" rel="noopener noreferrer">scchan.moe</a> | <a href="https://github.com/sichengchen/oricon" class="underline" target="_blank" rel="noopener noreferrer">View on Github</a>
  </footer>
</main>
