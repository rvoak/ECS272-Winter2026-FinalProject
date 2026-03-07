<script setup lang="ts">
import * as d3 from 'd3'
import { computed, onMounted, ref, watch } from 'vue'

type MetricMode = 'score' | 'rank'

interface CountryDelta {
    key: string
    name: string
    deltaScore: number
    deltaRank: number
    startScore: number
    endScore: number
    startRank: number
    endRank: number
}

interface SeriesPoint {
    year: number
    score: number
    rank: number
    factors: { label: string; value: number }[]
}

const rows = ref<CountryDelta[]>([])
const yearRange = ref<{ start: number; end: number } | null>(null)
const availableYears = ref<number[]>([])
const startYear = ref<number | null>(null)
const endYear = ref<number | null>(null)
const mode = ref<MetricMode>('rank')
const listRef = ref<HTMLElement | null>(null)
const tooltip = ref({ visible: false, x: 0, y: 0, text: '' })
const seriesByCountry = ref<Map<string, SeriesPoint[]>>(new Map())
const countryOptions = ref<{ name: string; series: SeriesPoint[] }[]>([])

const emit = defineEmits<{
    (event: 'country-selected', payload: {
        name: string
        series: SeriesPoint[]
        mode: MetricMode
        options: { name: string; series: SeriesPoint[] }[]
    }): void
}>()

const maxMagnitude = computed(() => {
    if (rows.value.length === 0) return 1
    if (mode.value === 'score') {
        return Math.max(...rows.value.map((row) => Math.abs(row.deltaScore)), 1)
    }
    return Math.max(...rows.value.map((row) => Math.abs(row.deltaRank)), 1)
})

const formatDelta = (value: number, isRank: boolean) => {
    if (isRank) {
        return value > 0 ? `+${value}` : `${value}`
    }
    return value > 0 ? `+${value.toFixed(2)}` : value.toFixed(2)
}

const showTooltip = (event: MouseEvent, row: CountryDelta) => {
    if (!listRef.value || !yearRange.value) return
    const bounds = listRef.value.getBoundingClientRect()
    const text =
        mode.value === 'score'
            ? `${yearRange.value.start}:${row.startScore.toFixed(2)} → ${yearRange.value.end}:${row.endScore.toFixed(2)}`
            : `${yearRange.value.start}:${row.startRank} → ${yearRange.value.end}:${row.endRank}`
    tooltip.value = {
        visible: true,
        x: event.clientX - bounds.left + 10,
        y: event.clientY - bounds.top + 10,
        text
    }
}

const moveTooltip = (event: MouseEvent) => {
    if (!listRef.value) return
    const bounds = listRef.value.getBoundingClientRect()
    tooltip.value = {
        ...tooltip.value,
        x: event.clientX - bounds.left + 10,
        y: event.clientY - bounds.top + 10
    }
}

const hideTooltip = () => {
    tooltip.value = { ...tooltip.value, visible: false }
}

const topGainers = computed(() => {
    const data = [...rows.value]
    if (mode.value === 'score') {
        return data
            .filter((row) => row.deltaScore > 0)
            .sort((a, b) => b.deltaScore - a.deltaScore)
            .slice(0, 10)
    }
    return data
        .filter((row) => row.deltaRank > 0)
        .sort((a, b) => b.deltaRank - a.deltaRank)
        .slice(0, 10)
})

const topDecliners = computed(() => {
    const data = [...rows.value]
    if (mode.value === 'score') {
        return data
            .filter((row) => row.deltaScore < 0)
            .sort((a, b) => a.deltaScore - b.deltaScore)
            .slice(0, 10)
    }
    return data
        .filter((row) => row.deltaRank < 0)
        .sort((a, b) => a.deltaRank - b.deltaRank)
        .slice(0, 10)
})

const factorColumns = [
    { label: 'Log GDP per capita', column: 'Explained by: Log GDP per capita' },
    { label: 'Social support', column: 'Explained by: Social support' },
    { label: 'Healthy life expectancy', column: 'Explained by: Healthy life expectancy' },
    { label: 'Freedom to make life choices', column: 'Explained by: Freedom to make life choices' },
    { label: 'Generosity', column: 'Explained by: Generosity' },
    { label: 'Perceptions of corruption', column: 'Explained by: Perceptions of corruption' }
]

async function load() {
    const data = await d3.csv('../../data/Happiness_Data_All.csv')
    const byYear = new Map<number, Map<string, { score: number; rank: number }>>()
    const byCountry = new Map<string, Map<number, { score: number; rank: number; factors: { label: string; value: number }[] }>>()
    const years = new Set<number>()

    data.forEach((row) => {
        const year = Number(row['Year'])
        const country = row['Country name']
        const score = Number(row['Life evaluation (3-year average)'])
        const rank = Number(row['Rank'])
        if (!country || Number.isNaN(year) || Number.isNaN(score) || Number.isNaN(rank)) return
        years.add(year)
        if (!byYear.has(year)) byYear.set(year, new Map())
        byYear.get(year)?.set(country, { score, rank })
        const factors = factorColumns
            .map((factor) => {
                const value = Number(row[factor.column])
                return Number.isNaN(value) ? null : { label: factor.label, value }
            })
            .filter((value): value is { label: string; value: number } => value !== null)
        if (!byCountry.has(country)) byCountry.set(country, new Map())
        byCountry.get(country)?.set(year, { score, rank, factors })
    })

    const yearList = Array.from(years).sort((a, b) => a - b)
    if (yearList.length === 0) return
    availableYears.value = yearList
    const defaultStart = yearList[0]
    const defaultEnd = yearList[yearList.length - 1]
    if (startYear.value === null || !yearList.includes(startYear.value)) {
        startYear.value = defaultStart
    }
    if (endYear.value === null || !yearList.includes(endYear.value)) {
        endYear.value = defaultEnd
    }
    if (startYear.value !== null && endYear.value !== null) {
        yearRange.value = { start: startYear.value, end: endYear.value }
    }

    if (startYear.value === null || endYear.value === null) return
    const startData = byYear.get(startYear.value)
    const endData = byYear.get(endYear.value)
    if (!startData || !endData) return

    const deltas: CountryDelta[] = []
    startData.forEach((startEntry, country) => {
        const endEntry = endData.get(country)
        if (!endEntry) return
        const deltaScore = endEntry.score - startEntry.score
        const deltaRank = startEntry.rank - endEntry.rank
        deltas.push({
            key: country,
            name: country,
            deltaScore,
            deltaRank,
            startScore: startEntry.score,
            endScore: endEntry.score,
            startRank: startEntry.rank,
            endRank: endEntry.rank
        })
    })
    rows.value = deltas

    const seriesMap = new Map<string, SeriesPoint[]>()
    byCountry.forEach((yearMap, country) => {
        const points: SeriesPoint[] = []
        availableYears.value.forEach((year) => {
            const entry = yearMap.get(year)
            if (!entry) return
            points.push({ year, score: entry.score, rank: entry.rank, factors: entry.factors })
        })
        seriesMap.set(country, points)
    })
    seriesByCountry.value = seriesMap
    countryOptions.value = Array.from(seriesMap.entries())
        .map(([name, series]) => ({ name, series }))
        .sort((a, b) => a.name.localeCompare(b.name))
}

onMounted(() => {
    void load()
})

watch([startYear, endYear], () => {
    void load()
})
</script>

<template>
    <div class="movers" ref="listRef">
        <div class="movers-header">
            <div>
                <h2>Movers List</h2>
                <p v-if="yearRange">Change from {{ yearRange.start }} to {{ yearRange.end }}</p>
            </div>
            <div class="mode-toggle">
                <button :class="{ active: mode === 'score' }" type="button" @click="mode = 'score'">
                    Score Change
                </button>
                <button :class="{ active: mode === 'rank' }" type="button" @click="mode = 'rank'">
                    Rank Change
                </button>
            </div>
        </div>
        <div class="year-selectors">
            <label>
                <span>Start Year</span>
                <select v-model.number="startYear">
                    <option v-for="year in availableYears" :key="`start-${year}`" :value="year">
                        {{ year }}
                    </option>
                </select>
            </label>
            <label>
                <span>End Year</span>
                <select v-model.number="endYear">
                    <option v-for="year in availableYears" :key="`end-${year}`" :value="year">
                        {{ year }}
                    </option>
                </select>
            </label>
        </div>
        <div class="movers-columns">
            <div class="movers-section">
                <h3>Top 10 Gains</h3>
                <div
                    v-for="row in topGainers"
                    :key="`up-${row.key}`"
                    class="mover-row"
                    @click="emit('country-selected', { name: row.name, series: seriesByCountry.get(row.name) ?? [], mode: mode.value, options: countryOptions })"
                >
                    <div class="mover-label">{{ row.name }}</div>
                    <div
                        class="mover-bar-track"
                        @mouseenter="showTooltip($event, row)"
                        @mousemove="moveTooltip($event)"
                        @mouseleave="hideTooltip"
                    >
                        <div
                            class="mover-bar up"
                            :style="{
                                width: `${mode === 'score'
                                    ? (Math.abs(row.deltaScore) / maxMagnitude) * 100
                                    : (Math.abs(row.deltaRank) / maxMagnitude) * 100}%`
                            }"
                        ></div>
                        <div
                            class="mover-value"
                            :style="{
                                left: `${mode === 'score'
                                    ? (Math.abs(row.deltaScore) / maxMagnitude) * 100
                                    : (Math.abs(row.deltaRank) / maxMagnitude) * 100}%`
                            }"
                        >
                            {{ mode === 'score' ? formatDelta(row.deltaScore, false) : formatDelta(row.deltaRank, true) }}
                        </div>
                    </div>
                </div>
            </div>
            <div class="movers-section">
                <h3>Top 10 Declines</h3>
                <div
                    v-for="row in topDecliners"
                    :key="`down-${row.key}`"
                    class="mover-row"
                    @click="emit('country-selected', { name: row.name, series: seriesByCountry.get(row.name) ?? [], mode: mode.value, options: countryOptions })"
                >
                    <div class="mover-label">{{ row.name }}</div>
                    <div
                        class="mover-bar-track"
                        @mouseenter="showTooltip($event, row)"
                        @mousemove="moveTooltip($event)"
                        @mouseleave="hideTooltip"
                    >
                        <div
                            class="mover-bar down"
                            :style="{
                                width: `${mode === 'score'
                                    ? (Math.abs(row.deltaScore) / maxMagnitude) * 100
                                    : (Math.abs(row.deltaRank) / maxMagnitude) * 100}%`
                            }"
                        ></div>
                        <div
                            class="mover-value"
                            :style="{
                                left: `${mode === 'score'
                                    ? (Math.abs(row.deltaScore) / maxMagnitude) * 100
                                    : (Math.abs(row.deltaRank) / maxMagnitude) * 100}%`
                            }"
                        >
                            {{ mode === 'score' ? formatDelta(row.deltaScore, false) : formatDelta(row.deltaRank, true) }}
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div v-if="tooltip.visible" class="movers-tooltip" :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }">
            {{ tooltip.text }}
        </div>
    </div>
</template>

<style scoped>
.movers {
    padding: 8px 12px 16px;
    height: 100%;
    display: flex;
    flex-direction: column;
    gap: 10px;
    position: relative;
}

.movers-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    gap: 10px;
}

.movers-header h2 {
    margin: 0;
    font-size: 1.05rem;
    color: #1f2f4a;
}

.movers-header p {
    margin: 4px 0 0;
    font-size: 0.75rem;
    color: #5a6577;
}

.mode-toggle {
    display: flex;
    gap: 6px;
}

.mode-toggle button {
    border: 1px solid #b7c3d6;
    border-radius: 8px;
    padding: 4px 10px;
    font-size: 0.75rem;
    background: #eef3f9;
    color: #1f2f4a;
    cursor: pointer;
}

.mode-toggle button.active {
    background: #1f2f4a;
    color: #ffffff;
    border-color: #1f2f4a;
}

.year-selectors {
    display: flex;
    gap: 10px;
    align-items: center;
    font-size: 0.75rem;
    color: #2f3b4f;
}

.year-selectors label {
    display: flex;
    align-items: center;
    gap: 8px;
}

.year-selectors select {
    appearance: none;
    -webkit-appearance: none;
    -moz-appearance: none;
    border: 1px solid #c7c7c7;
    border-radius: 6px;
    padding: 4px 26px 4px 8px;
    font-size: 0.75rem;
    background: #ffffff;
    background-image: linear-gradient(45deg, transparent 50%, #2f3b4f 50%),
        linear-gradient(135deg, #2f3b4f 50%, transparent 50%),
        linear-gradient(to right, #ffffff, #ffffff);
    background-position: calc(100% - 16px) 55%, calc(100% - 10px) 55%, 100% 0;
    background-size: 5px 5px, 5px 5px, 2em 100%;
    background-repeat: no-repeat;
}

.movers-columns {
    display: flex;
    flex-direction: column;
    gap: 14px;
    overflow: auto;
    padding-right: 6px;
}

.movers-section {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.movers-section h3 {
    margin: 0;
    font-size: 0.85rem;
    color: #1f2f4a;
}

.mover-row {
    display: grid;
    grid-template-columns: 140px 1fr;
    align-items: center;
    gap: 6px;
    cursor: pointer;
}

.mover-row:hover {
    background: rgba(31, 47, 74, 0.04);
    border-radius: 6px;
}

.mover-label {
    font-size: 0.75rem;
    color: #2f3b4f;
}

.mover-bar-track {
    height: 10px;
    background: transparent;
    border-radius: 999px;
    overflow: visible;
    position: relative;
    max-width: 50%;
}

.mover-bar {
    height: 100%;
    border-radius: 999px;
}

.mover-bar.up {
    background: #22c55e;
}

.mover-bar.down {
    background: #ef4444;
}

.mover-value {
    position: absolute;
    top: -4px;
    transform: translateX(6px);
    font-size: 0.7rem;
    color: #1f2f4a;
    white-space: nowrap;
}

.movers-tooltip {
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
</style>
