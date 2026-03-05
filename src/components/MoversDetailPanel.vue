<script setup lang="ts">
import * as d3 from 'd3'
import { computed, onBeforeUnmount, onMounted, ref, watchEffect } from 'vue'

type MetricMode = 'score' | 'rank'

interface SeriesPoint {
    year: number
    score: number
    rank: number
    factors: { label: string; value: number }[]
}

const props = defineProps<{
    open: boolean
    name: string | null
    mode: MetricMode
    series: SeriesPoint[]
    options: { name: string; series: SeriesPoint[] }[]
}>()

const emit = defineEmits<{
    (event: 'close'): void
}>()

const containerRef = ref<HTMLElement | null>(null)
const factorRef = ref<HTMLElement | null>(null)
const panelRef = ref<HTMLElement | null>(null)
const size = ref({ width: 0, height: 0 })
let resizeObserver: ResizeObserver | null = null
const tooltip = ref({ visible: false, x: 0, y: 0, text: '' })

const yLabel = computed(() => (props.mode === 'score' ? 'Happiness Score' : 'Rank'))
const selectedName = ref<string | null>(props.name)
const selectedSeries = ref<SeriesPoint[]>(props.series)

const onResize = () => {
    if (!containerRef.value && !factorRef.value) return
    size.value = { width: containerRef.value?.clientWidth ?? 0, height: containerRef.value?.clientHeight ?? 0 }
}

const getChartSize = (el: HTMLElement | null) => {
    if (!el) return { width: 0, height: 0 }
    return { width: el.clientWidth, height: el.clientHeight }
}

const renderScoreChart = () => {
    const { width: fullWidth, height: fullHeight } = getChartSize(containerRef.value)
    if (!containerRef.value || fullWidth <= 0 || fullHeight <= 0) return
    const svg = d3.select(containerRef.value).select('svg')
    svg.selectAll('*').remove()

    if (!selectedName.value || selectedSeries.value.length === 0) return

    const margin = { top: 30, right: 24, bottom: 36, left: 48 }
    const width = fullWidth - margin.left - margin.right
    const height = fullHeight - margin.top - margin.bottom
    if (width <= 0 || height <= 0) return

    const g = svg
        .attr('width', fullWidth)
        .attr('height', fullHeight)
        .append('g')
        .attr('transform', `translate(${margin.left}, ${margin.top})`)

    const years = selectedSeries.value.map((d) => d.year)
    const xScale = d3.scalePoint<number>().domain(years).range([0, width]).padding(0.4)

    const values = selectedSeries.value.map((d) => (props.mode === 'score' ? d.score : d.rank))
    const extent = d3.extent(values) as [number, number]
    const yDomain = props.mode === 'score' ? extent : [extent[1], extent[0]]
    const yScale = d3.scaleLinear().domain(yDomain).range([height, 0]).nice()

    const line = d3
        .line<SeriesPoint>()
        .x((d) => xScale(d.year) ?? 0)
        .y((d) => yScale(props.mode === 'score' ? d.score : d.rank))

    g.append('g')
        .attr('transform', `translate(0, ${height})`)
        .call(d3.axisBottom(xScale).tickSize(4))
        .call((axis) => axis.selectAll('text').style('font-size', '0.7rem'))
    g.append('g').call(d3.axisLeft(yScale).ticks(5))

    g.append('path')
        .datum(selectedSeries.value)
        .attr('fill', 'none')
        .attr('stroke', '#1f2f4a')
        .attr('stroke-width', 2)
        .attr('d', line)

    g.selectAll('circle')
        .data(selectedSeries.value)
        .join('circle')
        .attr('cx', (d) => xScale(d.year) ?? 0)
        .attr('cy', (d) => yScale(props.mode === 'score' ? d.score : d.rank))
        .attr('r', 3)
        .attr('fill', '#3b82f6')
        .on('mouseenter', (event, d) => {
            const value = props.mode === 'score' ? d.score.toFixed(2) : d.rank.toString()
            showTooltip(event as MouseEvent, `${d.year}: ${value}`)
        })
        .on('mousemove', (event) => moveTooltip(event as MouseEvent))
        .on('mouseleave', hideTooltip)

    g.append('text')
        .attr('x', width / 2)
        .attr('y', -12)
        .attr('text-anchor', 'middle')
        .style('font-size', '0.9rem')
        .style('fill', '#1f2f4a')
        .text(selectedName.value)

    g.append('text')
        .attr('x', -margin.left + 4)
        .attr('y', -10)
        .style('font-size', '0.75rem')
        .style('fill', '#5a6577')
        .text(yLabel.value)
}

const factorLabels = [
    'Log GDP per capita',
    'Social support',
    'Healthy life expectancy',
    'Freedom to make life choices',
    'Generosity',
    'Perceptions of corruption'
]

const factorColors = ['#2b59c3', '#3b82f6', '#22c55e', '#eab308', '#f97316', '#dc2626']

const renderFactorChart = () => {
    const { width: fullWidth, height: fullHeight } = getChartSize(factorRef.value)
    if (!factorRef.value || fullWidth <= 0 || fullHeight <= 0) return
    const svg = d3.select(factorRef.value).select('svg')
    svg.selectAll('*').remove()

    if (!selectedName.value || selectedSeries.value.length === 0) return

    const margin = { top: 24, right: 24, bottom: 46, left: 48 }
    const width = fullWidth - margin.left - margin.right
    const height = fullHeight - margin.top - margin.bottom
    if (width <= 0 || height <= 0) return

    const g = svg
        .attr('width', fullWidth)
        .attr('height', fullHeight)
        .append('g')
        .attr('transform', `translate(${margin.left}, ${margin.top})`)

    const years = selectedSeries.value.map((d) => d.year)
    const xScale = d3.scalePoint<number>().domain(years).range([0, width]).padding(0.4)

    const allValues = selectedSeries.value.flatMap((d) => d.factors.map((f) => f.value))
    const extent = d3.extent(allValues) as [number, number]
    const yScale = d3.scaleLinear().domain(extent).range([height, 0]).nice()

    g.append('g').attr('transform', `translate(0, ${height})`).call(d3.axisBottom(xScale).tickSize(4))
    g.append('g').call(d3.axisLeft(yScale).ticks(4))

    factorLabels.forEach((label, index) => {
        const line = d3
            .line<SeriesPoint>()
            .x((d) => xScale(d.year) ?? 0)
            .y((d) => {
                const factor = d.factors[index]
                return yScale(factor ? factor.value : 0)
            })
        g.append('path')
            .datum(selectedSeries.value)
            .attr('fill', 'none')
            .attr('stroke', factorColors[index])
            .attr('stroke-width', 1.5)
            .attr('d', line)

        g.selectAll(`circle.factor-${index}`)
            .data(selectedSeries.value)
            .join('circle')
            .attr('class', `factor-${index}`)
            .attr('cx', (d) => xScale(d.year) ?? 0)
            .attr('cy', (d) => {
                const factor = d.factors[index]
                return yScale(factor ? factor.value : 0)
            })
            .attr('r', 2.5)
            .attr('fill', factorColors[index])
            .on('mouseenter', (event, d) => {
                const factor = d.factors[index]
                const value = factor ? factor.value.toFixed(3) : '0.000'
                showTooltip(event as MouseEvent, `${label} • ${d.year}: ${value}`)
            })
            .on('mousemove', (event) => moveTooltip(event as MouseEvent))
            .on('mouseleave', hideTooltip)
    })

    const legend = g.append('g').attr('transform', `translate(0, ${height + 18})`)
    const legendItemWidth = Math.max(120, width / 3)
    factorLabels.forEach((label, index) => {
        const col = index % 3
        const row = Math.floor(index / 3)
        const x = col * legendItemWidth
        const y = row * 14
        const item = legend.append('g').attr('transform', `translate(${x}, ${y})`)
        item.append('line').attr('x1', 0).attr('y1', 6).attr('x2', 16).attr('y2', 6).attr('stroke', factorColors[index]).attr('stroke-width', 2)
        item.append('text').attr('x', 22).attr('y', 9).style('font-size', '0.7rem').style('fill', '#4a5568').text(label)
    })
}

watchEffect(() => {
    if (props.open) {
        requestAnimationFrame(() => {
            onResize()
            renderScoreChart()
            renderFactorChart()
        })
    }
})

watchEffect(() => {
    selectedName.value = props.name
    selectedSeries.value = props.series
    requestAnimationFrame(() => {
        onResize()
        renderScoreChart()
        renderFactorChart()
    })
})

const onSelectionChange = () => {
    if (!selectedName.value) return
    const found = props.options.find((option) => option.name === selectedName.value)
    if (found) {
        selectedSeries.value = found.series
        renderScoreChart()
        renderFactorChart()
    }
}

const showTooltip = (event: MouseEvent, text: string) => {
    if (!panelRef.value) return
    const bounds = panelRef.value.getBoundingClientRect()
    tooltip.value = {
        visible: true,
        x: event.clientX - bounds.left + 10,
        y: event.clientY - bounds.top + 10,
        text
    }
}

const moveTooltip = (event: MouseEvent) => {
    if (!panelRef.value) return
    const bounds = panelRef.value.getBoundingClientRect()
    tooltip.value = {
        ...tooltip.value,
        x: event.clientX - bounds.left + 10,
        y: event.clientY - bounds.top + 10
    }
}

const hideTooltip = () => {
    tooltip.value = { ...tooltip.value, visible: false }
}

onMounted(() => {
    onResize()
    if (containerRef.value) {
        resizeObserver = new ResizeObserver(() => onResize())
        resizeObserver.observe(containerRef.value)
    }
})

onBeforeUnmount(() => {
    if (resizeObserver) {
        resizeObserver.disconnect()
        resizeObserver = null
    }
})
</script>

<template>
    <aside class="panel" :class="{ open }" ref="panelRef">
        <div class="panel-header">
            <div class="panel-title">
                <span>Country Trend:</span>
                <select v-model="selectedName" @change="onSelectionChange">
                    <option v-for="option in options" :key="option.name" :value="option.name">
                        {{ option.name }}
                    </option>
                </select>
            </div>
            <button class="close-button" type="button" @click="emit('close')">Close</button>
        </div>
        <div class="panel-body">
            <div v-if="!name" class="panel-empty">Select a country bar to view trend.</div>
            <div v-else class="chart-shell score-chart" ref="containerRef">
                <svg></svg>
            </div>
            <div v-if="name" class="chart-shell factor-chart" ref="factorRef">
                <svg></svg>
            </div>
        </div>
        <div v-if="tooltip.visible" class="panel-tooltip" :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }">
            {{ tooltip.text }}
        </div>
    </aside>
</template>

<style scoped>
.panel {
    width: 0;
    flex: 0 0 0;
    max-width: 640px;
    height: 100%;
    overflow: hidden;
    background: #fbfbfc;
    border-left: 1px solid #e0e4ea;
    transition: width 260ms ease, transform 260ms ease;
    transform: translateX(20px);
    display: flex;
    flex-direction: column;
}

.panel.open {
    width: 640px;
    flex: 0 0 640px;
    transform: translateX(0);
}

.panel-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 16px 16px 8px;
    border-bottom: 1px solid #e6e9ef;
}

.panel-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.95rem;
    color: #1f2f4a;
    font-weight: 600;
}

.panel-title select {
    appearance: none;
    -webkit-appearance: none;
    -moz-appearance: none;
    border: 1px solid #c7c7c7;
    border-radius: 6px;
    padding: 6px 28px 6px 10px;
    font-size: 0.85rem;
    background: #ffffff;
    background-image: linear-gradient(45deg, transparent 50%, #2f3b4f 50%),
        linear-gradient(135deg, #2f3b4f 50%, transparent 50%),
        linear-gradient(to right, #ffffff, #ffffff);
    background-position: calc(100% - 16px) 55%, calc(100% - 10px) 55%, 100% 0;
    background-size: 6px 6px, 6px 6px, 2.2em 100%;
    background-repeat: no-repeat;
}

.close-button {
    border: 1px solid #c9d3e1;
    border-radius: 6px;
    padding: 4px 10px;
    font-size: 0.8rem;
    background: #ffffff;
    color: #1f2f4a;
    cursor: pointer;
}

.close-button:hover {
    background: #f0f4fa;
}

.panel-body {
    padding: 12px 16px 16px;
    height: calc(100% - 52px);
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.chart-shell {
    width: 100%;
    height: 100%;
    min-height: 0;
    flex: 1;
}

.score-chart {
    flex: 0.85;
}

.factor-chart {
    flex: 1.15;
}

.chart-shell svg {
    width: 100%;
    height: 100%;
    display: block;
}

.panel-tooltip {
    position: absolute;
    background: rgba(20, 20, 24, 0.92);
    color: #ffffff;
    padding: 6px 8px;
    border-radius: 6px;
    font-size: 0.75rem;
    pointer-events: none;
    z-index: 5;
    white-space: nowrap;
}

.panel-empty {
    font-size: 0.9rem;
    color: #5a6577;
}
</style>
