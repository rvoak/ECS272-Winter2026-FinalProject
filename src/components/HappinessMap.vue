
<script setup lang="ts">
import * as d3 from 'd3'
import { debounce } from 'lodash'
import { ref, computed, watchEffect, onMounted, onBeforeUnmount } from 'vue'
import type { FeatureCollection, Geometry, GeoJsonProperties } from 'geojson'

import { ComponentSize, Margin } from '../types'

const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 24, right: 24, top: 12, bottom: 52 }

const mapContainer = ref<HTMLElement | null>(null)
const mapStage = ref<HTMLElement | null>(null)
const tooltip = ref<HTMLElement | null>(null)
const geoData = ref<FeatureCollection<Geometry, GeoJsonProperties> | null>(null)
type YearEntry = { score: number; rank: number }
const happinessByYear = ref<Map<number, Map<string, YearEntry>>>(new Map())
const years = ref<number[]>([])
const selectedYear = ref<number>(2024)
const isPlaying = ref(false)
let playTimer: number | null = null


const canRender = computed(() => {
    const yearData = happinessByYear.value.get(selectedYear.value)
    return geoData.value !== null && !!yearData && yearData.size > 0 && size.value.width > 0 && size.value.height > 0
})

const normalizeCountryName = (value: string) =>
    value
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, ' ')
        .trim()

const countryAliases = new Map<string, string>([
    ['united states', 'united states of america'],
    ['usa', 'united states of america'],
    ['us', 'united states of america'],
    ['russia', 'russian federation'],
    ['czechia', 'czech republic'],
    ['south korea', 'korea republic of'],
    ['republic of korea', 'korea republic of'],
    ['north korea', 'korea democratic people s republic of'],
    ['laos', 'lao people s democratic republic'],
    ['syria', 'syrian arab republic'],
    ['iran', 'iran islamic republic of'],
    ['moldova', 'moldova republic of'],
    ['tanzania', 'tanzania united republic of'],
    ['united republic of tanzania', 'tanzania united republic of'],
    ['vietnam', 'viet nam'],
    ['bolivia', 'bolivia plurinational state of'],
    ['venezuela', 'venezuela bolivarian republic of'],
    ['brunei', 'brunei darussalam'],
    ['democratic republic of congo', 'congo the democratic republic of the'],
    ['democratic republic of the congo', 'congo the democratic republic of the'],
    ['dr congo', 'congo the democratic republic of the'],
    ['congo drc', 'congo the democratic republic of the'],
    ['congo kinshasa', 'congo the democratic republic of the'],
    ['republic of the congo', 'congo'],
    ['republic of congo', 'congo'],
    ['congo brazzaville', 'congo'],
    ['cote d ivoire', 'cote d ivoire'],
    ['ivory coast', 'cote d ivoire'],
    ['south sudan', 'south sudan'],
    ['sudan', 'sudan'],
    ['republic of the sudan', 'sudan'],
    ['the sudan', 'sudan'],
    ['angola', 'angola'],
    ['guinea', 'guinea'],
    ['turkey', 'turkiye'],
    ['turkiye', 'turkiye'],
    ['england', 'united kingdom'],
    ['macedonia', 'north macedonia'],
    ['republic of serbia', 'serbia'],
    ['equatorial guinea', 'guinea'],
    ['taiwan', 'taiwan province of china']
])

const canonicalizeCountryName = (value: string) => {
    const normalized = normalizeCountryName(value)
    return countryAliases.get(normalized) ?? normalized
}

const escapeHtml = (value: string) =>
    value.replace(/[&<>"']/g, (char) => {
        switch (char) {
            case '&':
                return '&amp;'
            case '<':
                return '&lt;'
            case '>':
                return '&gt;'
            case '"':
                return '&quot;'
            case "'":
                return '&#39;'
            default:
                return char
        }
    })

async function loadData() {
    const [world, happiness] = await Promise.all([
        d3.json('https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson'),
        d3.csv('../../data/Happiness_Data_All.csv')
    ])

    if (world) {
        geoData.value = world as FeatureCollection<Geometry, GeoJsonProperties>
    }

    const dataMap = new Map<number, Map<string, YearEntry>>()
    const yearSet = new Set<number>()
    happiness.forEach((row) => {
        const year = Number(row['Year'])
        const country = row['Country name']
        const score = Number(row['Life evaluation (3-year average)'])
        const rank = Number(row['Rank'])
        if (!country || Number.isNaN(score) || Number.isNaN(rank)) return
        yearSet.add(year)
        if (!dataMap.has(year)) {
            dataMap.set(year, new Map())
        }
        dataMap.get(year)?.set(canonicalizeCountryName(country), { score, rank })
    })
    happinessByYear.value = dataMap
    const yearList = Array.from(yearSet).sort((a, b) => b - a)
    years.value = yearList
    if (!yearSet.has(selectedYear.value) && yearList.length > 0) {
        selectedYear.value = yearList[0]
    }
}

function onResize() {
    const target = mapStage.value ?? mapContainer.value
    if (!target) return
    size.value = { width: target.clientWidth, height: target.clientHeight }
}

function renderMap() {
    if (!geoData.value) return

    const svg = d3.select('#map-svg')
    svg.selectAll('*').remove()
    const yearData = happinessByYear.value.get(selectedYear.value) ?? new Map()
    const maxRank = d3.max(Array.from(yearData.values()), (entry) => entry.rank) ?? 0

    const projection = d3
        .geoNaturalEarth1()
        .fitExtent(
            [
                [margin.left, margin.top],
                [size.value.width - margin.right, size.value.height - margin.bottom]
            ],
            geoData.value
        )

    const path = d3.geoPath(projection)
    const divergingPalette = d3.interpolateRgbBasis([
        '#0f3b74',
        '#4a83b3',
        '#a2c8e3',
        '#e9edf2',
        '#f0ad6e',
        '#c84d04',
        '#742100'
    ])
    const colorScale = d3.scaleDiverging(divergingPalette).domain([10, 5, 0]).clamp(true)

    svg.append('rect').attr('width', size.value.width).attr('height', size.value.height).attr('fill', '#f7f7f2')

    const yearBadge = svg.append('g').attr('transform', `translate(${margin.left + 6}, ${margin.top + 6})`)
    const yearLabel = String(selectedYear.value)
    const badgePaddingX = 12
    const badgePaddingY = 8
    const badgeText = yearBadge
        .append('text')
        .attr('x', badgePaddingX)
        .attr('y', badgePaddingY + 16)
        .style('font-size', '1.25rem')
        .style('font-weight', '600')
        .style('fill', '#1f2f4a')
        .text(yearLabel)
    const badgeBox = (badgeText.node() as SVGTextElement).getBBox()
    yearBadge
        .insert('rect', 'text')
        .attr('width', badgeBox.width + badgePaddingX * 2)
        .attr('height', badgeBox.height + badgePaddingY * 2)
        .attr('rx', 10)
        .attr('ry', 10)
        .attr('fill', 'rgba(255, 255, 255, 0.88)')
        .attr('stroke', '#c9d3e1')
        .attr('stroke-width', 0.9)

    const mapLayer = svg.append('g')
    mapLayer
        .selectAll('path')
        .data(geoData.value.features)
        .join('path')
        .attr('d', (feature) => path(feature as d3.GeoPermissibleObjects) ?? '')
        .attr('fill', (feature) => {
            const name = (feature.properties?.name ??
                feature.properties?.ADMIN ??
                feature.properties?.admin ??
                '') as string
            const entry = yearData.get(canonicalizeCountryName(name))
            return entry === undefined ? '#d9d9d9' : (colorScale(entry.score) as string)
        })
        .attr('stroke', '#ffffff')
        .attr('stroke-width', 0.5)
        .on('mousemove', (event, feature) => {
            if (!tooltip.value || !mapStage.value) return
            const name = (feature.properties?.name ??
                feature.properties?.ADMIN ??
                feature.properties?.admin ??
                '') as string
            const entry = yearData.get(canonicalizeCountryName(name))
            const safeName = escapeHtml(name)
            tooltip.value.innerHTML = entry
                ? `<strong>${safeName}</strong><br>Score: ${entry.score.toFixed(2)}<br>Rank: ${entry.rank}`
                : `<strong>${safeName}</strong>: no data`
            const bounds = mapStage.value.getBoundingClientRect()
            const x = event.clientX - bounds.left + 12
            const y = event.clientY - bounds.top + 12
            tooltip.value.style.left = `${x}px`
            tooltip.value.style.top = `${y}px`
            tooltip.value.style.opacity = '1'
        })
        .on('mouseleave', () => {
            if (!tooltip.value) return
            tooltip.value.style.opacity = '0'
        })

    const highlightFeatures = geoData.value.features.filter((feature) => {
        const name = (feature.properties?.name ??
            feature.properties?.ADMIN ??
            feature.properties?.admin ??
            '') as string
        const entry = yearData.get(canonicalizeCountryName(name))
        return entry?.rank === 1 || entry?.rank === maxRank
    })

    const highlightLayer = svg.append('g')
    highlightLayer
        .selectAll('path')
        .data(highlightFeatures)
        .join('path')
        .attr('d', (feature) => path(feature as d3.GeoPermissibleObjects) ?? '')
        .attr('fill', 'none')
        .attr('stroke', (feature) => {
            const name = (feature.properties?.name ??
                feature.properties?.ADMIN ??
                feature.properties?.admin ??
                '') as string
            const entry = yearData.get(canonicalizeCountryName(name))
            return entry?.rank === 1 ? '#0b2f6b' : '#8a2a06'
        })
        .attr('stroke-width', 2.5)
        .attr('stroke-linejoin', 'round')
        .attr('stroke-linecap', 'round')

    const legendWidth = Math.max(180, size.value.width * 0.5)
    const legendHeight = 10
    const legendX = (size.value.width - legendWidth) / 2
    const legendY = size.value.height - margin.bottom + 24

    const defs = svg.append('defs')
    const gradientId = 'legend-gradient'
    const gradient = defs
        .append('linearGradient')
        .attr('id', gradientId)
        .attr('x1', '0%')
        .attr('x2', '100%')
        .attr('y1', '0%')
        .attr('y2', '0%')

    const legendStops = d3.range(0, 1.0001, 0.1)
    legendStops.forEach((t) => {
        gradient.append('stop').attr('offset', `${t * 100}%`).attr('stop-color', divergingPalette(1 - t))
    })

    svg.append('g')
        .attr('transform', `translate(${legendX}, ${legendY})`)
        .append('rect')
        .attr('width', legendWidth)
        .attr('height', legendHeight)
        .attr('fill', `url(#${gradientId})`)
        .attr('stroke', '#bdbdbd')
        .attr('stroke-width', 0.5)

    const legendScale = d3.scaleLinear().domain([0, 10]).range([0, legendWidth])
    const legendAxis = d3.axisBottom(legendScale).ticks(5).tickSize(4)

    svg.append('g')
        .attr('transform', `translate(${legendX}, ${legendY + legendHeight})`)
        .call(legendAxis)
        .call((g) => g.select('.domain').attr('stroke', '#6b6b6b'))
        .call((g) => g.selectAll('text').attr('fill', '#4a4a4a').style('font-size', '0.75rem'))

    svg.append('text')
        .attr('x', size.value.width / 2)
        .attr('y', legendY - 8)
        .attr('text-anchor', 'middle')
        .style('font-size', '0.8rem')
        .style('fill', '#3a3a3a')
        .text('Happiness Score')
}

function stopPlayback() {
    if (playTimer !== null) {
        window.clearInterval(playTimer)
        playTimer = null
    }
    isPlaying.value = false
}

function startPlayback() {
    const yearList = [...years.value].sort((a, b) => a - b)
    if (yearList.length === 0) return

    let index = 0
    isPlaying.value = true

    playTimer = window.setInterval(() => {
        selectedYear.value = yearList[index]
        index += 1
        if (index >= yearList.length) {
            stopPlayback()
        }
    }, 900)
}

function togglePlayback() {
    if (isPlaying.value) {
        stopPlayback()
    } else {
        startPlayback()
    }
}

watchEffect(() => {
    if (canRender.value) {
        renderMap()
    }
})

const debouncedOnResize = debounce(onResize, 100)

onMounted(() => {
    window.addEventListener('resize', debouncedOnResize)
    onResize()
    void loadData()
})

onBeforeUnmount(() => {
    stopPlayback()
    window.removeEventListener('resize', debouncedOnResize)
})
</script>

<!-- "ref" registers a reference to the HTML element so that we can access it via the reference in Vue.  -->
<!-- We use flex (d-flex) to arrange the layout-->
<template>
    <div class="chart-container d-flex" ref="mapContainer">
        <div class="map-controls">
            <label class="select-control">
                <span>Year</span>
                <select v-model.number="selectedYear">
                    <option v-for="year in years" :key="year" :value="year">
                        {{ year }}
                    </option>
                </select>
            </label>
            <button class="play-button" type="button" @click="togglePlayback">
                {{ isPlaying ? 'PAUSE TRANSITION' : 'PLAY TRANSITION' }}
            </button>
        </div>
        <div class="map-stage" ref="mapStage">
            <svg id="map-svg" width="100%" height="100%"></svg>
            <div ref="tooltip" class="map-tooltip" aria-hidden="true"></div>
        </div>
    </div>
</template>

<style scoped>
.chart-container {
    height: 100%;
    position: relative;
    overflow: hidden;
    flex-direction: column;
    gap: 8px;
}

.map-controls {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    gap: 16px;
    padding: 8px 12px 0;
    color: #2f2f2f;
}

.play-button {
    margin-left: auto;
    border: 1px solid #b7c3d6;
    border-radius: 8px;
    padding: 8px 14px;
    font-size: 0.95rem;
    font-weight: 600;
    background: #eef3f9;
    color: #1f2f4a;
    cursor: pointer;
    transition: background 150ms ease, border-color 150ms ease;
}

.play-button:hover {
    background: #e3ecf7;
    border-color: #9db0cb;
}

.play-button:active {
    background: #d7e3f3;
}

.select-control {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.95rem;
}

.select-control select {
    border: 1px solid #c7c7c7;
    border-radius: 6px;
    padding: 8px 12px;
    font-size: 1rem;
    background: #ffffff;
    color: #1f1f1f;
}

.map-stage {
    position: relative;
    flex: 1;
    min-height: 0;
}

.map-tooltip {
    position: absolute;
    pointer-events: none;
    opacity: 0;
    background: rgba(30, 30, 30, 0.9);
    color: #ffffff;
    padding: 6px 8px;
    border-radius: 6px;
    font-size: 0.75rem;
    transition: opacity 120ms ease;
    white-space: pre-line;
}
</style>
