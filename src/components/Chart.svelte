<script>
  export let currentItems = []
  export let chartWidth = 1000
  export let chartHeight = 600
  export let innerWidth = 760
  export let margin = { top: 90, right: 120, bottom: 60, left: 220 }
  export let bandHeight = 34
  export let barHeight = 26
  export let labelOffset = 28
  export let isMobile = false
  export let barWidthScale = 1

  const getBarWidth = () => innerWidth * barWidthScale
</script>

<section class="card">
  <svg viewBox={`0 0 ${chartWidth} ${chartHeight}`} preserveAspectRatio="xMidYMid meet" class="w-full h-auto">
    <g transform={`translate(${margin.left}, ${margin.top})`}>
      {#each currentItems as item (item.key)}
        <g
          style={`transform: translateY(${item.index * bandHeight}px); opacity: ${item.opacity ?? 1};`}
        >
          <rect
            x={labelOffset}
            y="0"
            width={getBarWidth()}
            height={barHeight}
            rx="8"
            fill={item.color}
          />
          {#if isMobile}
            <text
              x={labelOffset + 12}
              y={barHeight / 2 - 4}
            >
              <tspan class="bar-text-rank" x={labelOffset + 10} dy="0">
                #{item.rank}
              </tspan>
              <tspan class="bar-text-title" x={labelOffset + 40} dy="0">
                {item.title}
              </tspan>
            </text>
            <text
              class="bar-text-artist"
              x={labelOffset + 40}
              y={barHeight / 2 + 16}
            >
              {item.artist}
            </text>
          {:else}
            <text
              class="bar-text-title"
              x={labelOffset + 12}
              y={barHeight / 2 + 2}
              dy="0.35em"
            >
              <tspan class="bar-text-rank">#{item.rank}</tspan>
              <tspan dx="8">
              {item.title}
              </tspan>
              <tspan class="bar-text-artist" dx="10">
                {item.artist}
              </tspan>
            </text>
          {/if}
        </g>
      {/each}
    </g>
  </svg>
</section>
