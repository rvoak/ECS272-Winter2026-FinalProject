
<script setup lang="ts">
import * as d3 from 'd3'
import { debounce } from 'lodash'
import { ref, computed, watch, watchEffect, onMounted, onBeforeUnmount } from 'vue'
import type { FeatureCollection, Geometry, GeoJsonProperties } from 'geojson'

import { ComponentSize, Margin } from '../types'

const size = ref<ComponentSize>({ width: 0, height: 0 })
const margin: Margin = { left: 24, right: 24, top: 12, bottom: 52 }

const mapContainer = ref<HTMLElement | null>(null)
const mapStage = ref<HTMLElement | null>(null)
const tooltip = ref<HTMLElement | null>(null)
const geoData = ref<FeatureCollection<Geometry, GeoJsonProperties> | null>(null)
type FactorEntry = { label: string; value: number }
type YearEntry = { score: number; rank: number; factors: FactorEntry[] }
const happinessByYear = ref<Map<number, Map<string, YearEntry>>>(new Map())
const years = ref<number[]>([])
const selectedYear = ref<number>(2024)
const isPlaying = ref(false)
let playTimer: number | null = null
const mode = ref<'score' | 'delta'>('score')
const deltaYearA = ref<number | null>(null)
const deltaYearB = ref<number | null>(null)
let resizeObserver: ResizeObserver | null = null

const props = defineProps<{
    selectedKeys: string[]
}>()

const emit = defineEmits<{
    (event: 'country-selected', payload: {
        key: string
        name: string
        year: number | null
        score: number | null
        rank: number | null
        factors: FactorEntry[]
        hasData: boolean
    }): void
}>()

type FactorColumn = { label: string; column: string } | { label: string; columns: string[] }

const factorColumns: FactorColumn[] = [
    { label: 'Log GDP per capita', column: 'Explained by: Log GDP per capita' },
    { label: 'Social support', column: 'Explained by: Social support' },
    { label: 'Healthy life expectancy', column: 'Explained by: Healthy life expectancy' },
    { label: 'Freedom to make life choices', column: 'Explained by: Freedom to make life choices' },
    {
        label: 'Generosity + corruption',
        columns: ['Explained by: Generosity', 'Explained by: Perceptions of corruption']
    },
    { label: 'Dystopia + residual', column: 'Dystopia + residual' }
]

const canRender = computed(() => {
    if (geoData.value === null || size.value.width <= 0 || size.value.height <= 0) return false
    if (mode.value === 'score') {
        const yearData = happinessByYear.value.get(selectedYear.value)
        return !!yearData && yearData.size > 0
    }
    if (deltaYearA.value === null || deltaYearB.value === null) return false
    const fromData = happinessByYear.value.get(deltaYearA.value)
    const toData = happinessByYear.value.get(deltaYearB.value)
    return !!fromData && !!toData && fromData.size > 0 && toData.size > 0
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
        const factors: FactorEntry[] = factorColumns
            .map((factor) => {
                if ('columns' in factor) {
                    const values = factor.columns.map((column) => Number(row[column]))
                    if (values.some((value) => Number.isNaN(value))) return null
                    const sum = values.reduce((acc, value) => acc + value, 0)
                    return { label: factor.label, value: sum }
                }
                const value = Number(row[factor.column])
                return Number.isNaN(value) ? null : { label: factor.label, value }
            })
            .filter((value): value is FactorEntry => value !== null)
        yearSet.add(year)
        if (!dataMap.has(year)) {
            dataMap.set(year, new Map())
        }
        dataMap.get(year)?.set(canonicalizeCountryName(country), { score, rank, factors })
    })
    happinessByYear.value = dataMap
    const yearList = Array.from(yearSet).sort((a, b) => b - a)
    years.value = yearList
    if (!yearSet.has(selectedYear.value) && yearList.length > 0) {
        selectedYear.value = yearList[0]
    }
    if (yearList.length > 0) {
        const minYear = yearList[yearList.length - 1]
        const maxYear = yearList[0]
        if (deltaYearA.value === null || !yearSet.has(deltaYearA.value)) {
            deltaYearA.value = minYear
        }
        if (deltaYearB.value === null || !yearSet.has(deltaYearB.value)) {
            deltaYearB.value = maxYear
        }
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
    const yearData =
        mode.value === 'score'
            ? happinessByYear.value.get(selectedYear.value) ?? new Map()
            : new Map()

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
    const colorScale = d3
        .scaleDiverging(divergingPalette)
        .domain(mode.value === 'score' ? [10, 5, 0] : [1, 0, -1])
        .clamp(true)

    svg.append('rect').attr('width', size.value.width).attr('height', size.value.height).attr('fill', '#f7f7f2')
    svg.append('defs')

    const yearBadge = svg.append('g').attr('transform', `translate(${margin.left + 6}, ${margin.top + 6})`)
    const yearLabel =
        mode.value === 'score'
            ? String(selectedYear.value)
            : `${deltaYearA.value ?? ''}–${deltaYearB.value ?? ''}`
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
    const maxRank =
        mode.value === 'score' ? d3.max(Array.from(yearData.values()), (entry) => entry.rank) ?? 0 : 0
    const deltaMax = (() => {
        if (mode.value !== 'delta' || deltaYearA.value === null || deltaYearB.value === null) return 1
        const fromData = happinessByYear.value.get(deltaYearA.value)
        const toData = happinessByYear.value.get(deltaYearB.value)
        if (!fromData || !toData) return 1
        const diffs: number[] = []
        fromData.forEach((entry, key) => {
            const other = toData.get(key)
            if (!other) return
            diffs.push(other.score - entry.score)
        })
        const maxAbs = d3.max(diffs.map((value) => Math.abs(value))) ?? 1
        return maxAbs === 0 ? 1 : maxAbs
    })()
    if (mode.value === 'delta') {
        colorScale.domain([deltaMax, 0, -deltaMax])
    }

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
            const key = canonicalizeCountryName(name)
            if (mode.value === 'score') {
                const entry = yearData.get(key)
                return entry === undefined ? '#d9d9d9' : (colorScale(entry.score) as string)
            }
            if (deltaYearA.value === null || deltaYearB.value === null) return '#d9d9d9'
            const fromData = happinessByYear.value.get(deltaYearA.value)
            const toData = happinessByYear.value.get(deltaYearB.value)
            const entryA = fromData?.get(key)
            const entryB = toData?.get(key)
            if (!entryA || !entryB) return '#d9d9d9'
            const deltaScore = entryB.score - entryA.score
            return colorScale(deltaScore) as string
        })
        .attr('stroke', '#ffffff')
        .attr('stroke-width', 0.5)
        .on('click', (event, feature) => {
            const name = (feature.properties?.name ??
                feature.properties?.ADMIN ??
                feature.properties?.admin ??
                '') as string
            const key = canonicalizeCountryName(name)
            if (mode.value === 'score') {
                const entry = yearData.get(key)
                emit('country-selected', {
                    key,
                    name,
                    year: selectedYear.value,
                    score: entry?.score ?? null,
                    rank: entry?.rank ?? null,
                    factors: entry?.factors ?? [],
                    hasData: !!entry
                })
                return
            }
            const yearB = deltaYearB.value
            if (yearB === null) {
                emit('country-selected', {
                    key,
                    name,
                    year: null,
                    score: null,
                    rank: null,
                    factors: [],
                    hasData: false
                })
                return
            }
            const toData = happinessByYear.value.get(yearB)
            const entryB = toData?.get(key)
            emit('country-selected', {
                key,
                name,
                year: yearB,
                score: entryB?.score ?? null,
                rank: entryB?.rank ?? null,
                factors: entryB?.factors ?? [],
                hasData: !!entryB
            })
        })
        .on('mousemove', (event, feature) => {
            if (!tooltip.value || !mapStage.value) return
            const name = (feature.properties?.name ??
                feature.properties?.ADMIN ??
                feature.properties?.admin ??
                '') as string
            const safeName = escapeHtml(name)
            const key = canonicalizeCountryName(name)
            if (mode.value === 'score') {
                const entry = yearData.get(key)
                tooltip.value.innerHTML = entry
                    ? `<strong>${safeName}</strong><br>Score: ${entry.score.toFixed(2)}<br>Rank: ${entry.rank}`
                    : `<strong>${safeName}</strong>: no data`
            } else {
                if (deltaYearA.value === null || deltaYearB.value === null) {
                    tooltip.value.innerHTML = `<strong>${safeName}</strong>: no data`
                } else {
                    const fromData = happinessByYear.value.get(deltaYearA.value)
                    const toData = happinessByYear.value.get(deltaYearB.value)
                    const entryA = fromData?.get(key)
                    const entryB = toData?.get(key)
                    if (!entryA || !entryB) {
                        tooltip.value.innerHTML = `<strong>${safeName}</strong>: no data`
                    } else {
                        const deltaScore = entryB.score - entryA.score
                        const deltaRank = entryB.rank - entryA.rank
                        const scoreLabel = deltaScore >= 0 ? `+${deltaScore.toFixed(2)}` : deltaScore.toFixed(2)
                        const rankLabel = deltaRank >= 0 ? `+${deltaRank}` : `${deltaRank}`
                        tooltip.value.innerHTML = `<strong>${safeName}</strong><br>` +
                            `Year A (${deltaYearA.value}): ${entryA.score.toFixed(2)} / Rank ${entryA.rank}<br>` +
                            `Year B (${deltaYearB.value}): ${entryB.score.toFixed(2)} / Rank ${entryB.rank}<br>` +
                            `Delta Score: ${scoreLabel}<br>Delta Rank: ${rankLabel}`
                    }
                }
            }
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

    const highlightFeatures =
        mode.value === 'score'
            ? geoData.value.features.filter((feature) => {
                  const name = (feature.properties?.name ??
                      feature.properties?.ADMIN ??
                      feature.properties?.admin ??
                      '') as string
                  const entry = yearData.get(canonicalizeCountryName(name))
                  return entry?.rank === 1 || entry?.rank === maxRank
              })
            : []

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

    const selectedFeatures = geoData.value.features.filter((feature) => {
        const name = (feature.properties?.name ??
            feature.properties?.ADMIN ??
            feature.properties?.admin ??
            '') as string
        const key = canonicalizeCountryName(name)
        return props.selectedKeys.includes(key)
    })

    const selectedLayer = svg.append('g')
    selectedLayer
        .selectAll('path')
        .data(selectedFeatures)
        .join('path')
        .attr('d', (feature) => path(feature as d3.GeoPermissibleObjects) ?? '')
        .attr('fill', 'none')
        .attr('stroke', '#0b0b0b')
        .attr('stroke-width', 2)
        .attr('vector-effect', 'non-scaling-stroke')
        .attr('stroke-linejoin', 'round')
        .attr('stroke-linecap', 'round')

    const legendWidth = Math.max(180, size.value.width * 0.5)
    const legendHeight = 10
    const legendX = (size.value.width - legendWidth) / 2
    const legendY = size.value.height - margin.bottom + 24

    const legendDefs = svg.append('defs')
    const gradientId = 'legend-gradient'
    const gradient = legendDefs
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

    const legendScale = d3
        .scaleLinear()
        .domain(mode.value === 'score' ? [0, 10] : [-deltaMax, deltaMax])
        .range([0, legendWidth])
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
        .text(mode.value === 'score' ? 'Happiness Score' : 'Score Delta (Year B - Year A)')
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

watch(mode, (next) => {
    if (next === 'delta') {
        stopPlayback()
    }
})

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
    if (mapStage.value) {
        resizeObserver = new ResizeObserver(() => onResize())
        resizeObserver.observe(mapStage.value)
    }
})

onBeforeUnmount(() => {
    stopPlayback()
    window.removeEventListener('resize', debouncedOnResize)
    if (resizeObserver) {
        resizeObserver.disconnect()
        resizeObserver = null
    }
})
</script>

<!-- "ref" registers a reference to the HTML element so that we can access it via the reference in Vue.  -->
<!-- We use flex (d-flex) to arrange the layout-->
<template>
    <div class="chart-container d-flex" ref="mapContainer">
        <div class="map-controls">
            <label class="select-control">
                <span>Mode</span>
                <select v-model="mode">
                    <option value="score">Score</option>
                    <option value="delta">Delta</option>
                </select>
            </label>
            <label v-if="mode === 'score'" class="select-control">
                <span>Year</span>
                <select v-model.number="selectedYear">
                    <option v-for="year in years" :key="year" :value="year">
                        {{ year }}
                    </option>
                </select>
            </label>
            <label v-else class="select-control">
                <span>Year A</span>
                <select v-model.number="deltaYearA">
                    <option v-for="year in years" :key="`a-${year}`" :value="year">
                        {{ year }}
                    </option>
                </select>
            </label>
            <label v-if="mode === 'delta'" class="select-control">
                <span>Year B</span>
                <select v-model.number="deltaYearB">
                    <option v-for="year in years" :key="`b-${year}`" :value="year">
                        {{ year }}
                    </option>
                </select>
            </label>
            <button v-if="mode === 'score'" class="play-button" type="button" @click="togglePlayback">
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
