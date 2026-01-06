<script>
  import { onDestroy, onMount } from 'svelte'
  import { linear } from 'svelte/easing'
  import { tweened } from 'svelte/motion'
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
  const baseSpeed = 1300
  const baseTransitionDuration = 420
  const minTransitionDuration = 120
  const fastSpeedThreshold = 4

  let dates = []
  let dataByDate = new Map()

  let currentIndex = 0
  let previousIndex = 0
  let isPlaying = true
  let speed = baseSpeed
  let tickHandle = null
  let isMobile = false
  let chartContainerWidth = 0
  let mediaQuery = null
  let handleMediaChange = null
  let loadError = ''
  let showIntroModal = false

  const introStorageKey = 'oricon_intro_seen'

  const transitionProgress = tweened(1, { duration: 0, easing: linear })
  let progress = 1
  const progressUnsubscribe = transitionProgress.subscribe((value) => {
    progress = value
  })

  const startTransition = () => {
    transitionProgress.set(0, { duration: 0 })
    transitionProgress.set(1, { duration: transitionDuration, easing: linear })
  }

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
    previousIndex = currentIndex
    currentIndex = (currentIndex + 1) % dates.length
    startTransition()
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
    previousIndex = currentIndex
    currentIndex = (currentIndex + delta + dates.length) % dates.length
    startTransition()
  }

  const handleScrub = () => {
    stopPlayback()
    previousIndex = currentIndex
    transitionProgress.set(1, { duration: 0 })
  }

  const dismissIntroModal = () => {
    showIntroModal = false
    try {
      localStorage.setItem(introStorageKey, 'true')
    } catch (error) {
      console.warn('Failed to persist intro modal preference.', error)
    }
  }

  const checkIntroModal = () => {
    try {
      showIntroModal = localStorage.getItem(introStorageKey) !== 'true'
    } catch (error) {
      showIntroModal = true
      console.warn('Failed to read intro modal preference.', error)
    }
  }

  const handleIntroKeydown = (event) => {
    if (!showIntroModal) return
    if (event.key === 'Escape') {
      dismissIntroModal()
    }
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
    checkIntroModal()
  })

  onDestroy(() => {
    stopPlayback()
    progressUnsubscribe()
    if (mediaQuery && handleMediaChange) {
      if (mediaQuery.removeEventListener) {
        mediaQuery.removeEventListener('change', handleMediaChange)
      } else {
        mediaQuery.removeListener(handleMediaChange)
      }
    }
  })

  $: currentDate = dates[currentIndex] || ''
  $: previousDate = dates[previousIndex] || currentDate
  $: sortedItems = (dataByDate.get(currentDate) || [])
    .slice()
    .sort((a, b) => a.rank - b.rank)
  $: previousItems = (dataByDate.get(previousDate) || [])
    .slice()
    .sort((a, b) => a.rank - b.rank)
  $: topItems = sortedItems
  $: topN = topItems.length
  $: maxRank = Math.max(topItems.length, previousItems.length) || 1
  $: colorScale = d3
    .scaleLinear()
    .domain([1, Math.max(1, maxRank)])
    .range(['#d96a74', '#f2a3a5'])
  $: previousRankByKey = new Map(
    previousItems.map((entry) => [`${entry.title}__${entry.artist}`, entry.rank])
  )
  $: currentItems = topItems.map((entry) => {
    const key = `${entry.title}__${entry.artist}`
    const previousRank = previousRankByKey.get(key)
    const startRank = previousRank ?? entry.rank
    const displayRank = startRank + (entry.rank - startRank) * progress
    return {
      ...entry,
      value: 1,
      index: displayRank - 1,
      key,
      color: colorScale(displayRank),
      opacity: previousRank == null ? progress : 1
    }
  })
  $: speedMultiplier = baseSpeed / speed
  $: transitionDuration =
    speedMultiplier >= fastSpeedThreshold
      ? Math.max(
          minTransitionDuration,
          Math.round(baseTransitionDuration / Math.sqrt(speedMultiplier))
        )
      : baseTransitionDuration
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

<svelte:window on:keydown={handleIntroKeydown} />

<main class="mx-auto grid min-h-screen max-w-5xl gap-6 px-4 py-6 md:px-10 md:py-10">
  {#if showIntroModal}
    <div
      class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/40 px-4 py-6"
      role="presentation"
    >
      <button
        class="absolute inset-0 z-0 h-full w-full bg-transparent"
        type="button"
        aria-label="Dismiss intro"
        on:click={dismissIntroModal}
      ></button>
      <div
        class="card relative z-10 w-full max-w-xl shadow-[0_24px_48px_rgba(15,23,42,0.18)]"
        role="dialog"
        aria-modal="true"
        aria-labelledby="intro-title"
        tabindex="-1"
      >
        <div class="flex items-start justify-between gap-4">
          <div>
            <div class="pill mb-3 text-xs font-semibold uppercase tracking-[0.2em] text-slate-700">
              Quick Guide
            </div>
            <h2
              id="intro-title"
              class="ml-2 font-display text-2xl font-bold uppercase text-slate-900 sm:text-3xl"
            >
              Welcome to Oricon Charts Viewer
            </h2>
            <p class="ml-2 mt-2 text-sm text-slate-700">
              First time here? Here are two quick tips to get you started.
            </p>
          </div>
        </div>

        <div class="mt-4 grid gap-3 text-sm text-slate-800">
          <div class="rounded-2xl border border-slate-900/10 bg-white/70 px-4 py-3">
            Click the menu button in the top-left to expand controls and playback settings.
          </div>
          <div class="rounded-2xl border border-slate-900/10 bg-white/70 px-4 py-3">
            Click any song to search it on Google.
          </div>
        </div>

        <div class="mt-5 flex flex-wrap items-center justify-end gap-3">
          <button class="btn btn-primary" on:click={dismissIntroModal}>
            Got it
          </button>
        </div>
      </div>
    </div>
  {/if}

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

  <footer class="text-center text-xs text-slate-800">
    <div>
      Website by
      <a class="underline" href="https://scchan.moe" target="_blank" rel="noopener noreferrer">
        scchan.moe
      </a>
      |
      <a
        href="https://github.com/sichengchen/oricon"
        class="underline"
        target="_blank"
        rel="noopener noreferrer"
      >
        View on Github
      </a>
    </div>
    <div class="mt-2 text-[10px] leading-relaxed">
      <div>
        This site is not affiliated with Oricon. It only collects and organizes
        publicly available chart data.
      </div>
      <div class="mt-1">
        本サイトはオリコンとは一切関係ありません。公開されているチャートデータを収集・整理して紹介しています。
      </div>
    </div>
  </footer>
</main>
