<script setup lang="ts">
import { ref } from 'vue'
interface FactorEntry {
    label: string
    value: number
}

interface PanelData {
    key: string
    name: string
    year: number | null
    score: number | null
    rank: number | null
    factors: FactorEntry[]
    hasData: boolean
}

const panelRef = ref<HTMLElement | null>(null)
const tooltip = ref({ visible: false, x: 0, y: 0, text: '' })

const emit = defineEmits<{
    (event: 'close'): void
}>()

defineProps<{
    open: boolean
    data: PanelData[]
}>()

const legendLabels = [
    'Log GDP per capita',
    'Social support',
    'Healthy life expectancy',
    'Freedom to make life choices',
    'Generosity + corruption',
    'Dystopia + residual'
]

const segmentColors = [
    '#3aa59a',
    '#4edacb',
    '#b7e36b',
    '#c992fb',
    '#ee6aa6',
    '#d86f86'
]

const showTooltip = (event: MouseEvent, label: string, value: number) => {
    if (!panelRef.value) return
    const bounds = panelRef.value.getBoundingClientRect()
    tooltip.value = {
        visible: true,
        x: event.clientX - bounds.left + 10,
        y: event.clientY - bounds.top + 10,
        text: `${label}: ${value.toFixed(3)}`
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
</script>

<template>
    <aside
        class="panel"
        :class="{ open }"
        :style="{ width: open ? '360px' : '0px', flex: open ? '0 0 360px' : '0 0 0' }"
        ref="panelRef"
    >
        <div class="panel-header">
            <h2>Factor Comparison</h2>
            <button class="close-button" type="button" @click="emit('close')">Close</button>
        </div>
        <div v-if="data.length > 0" class="panel-body">
            <div class="panel-meta" v-if="data[0].year !== null">Year {{ data[0].year }}</div>
            <div class="bar-list">
                <div v-for="country in data" :key="country.key" class="bar-row">
                    <div class="bar-label">
                        <div class="bar-name">{{ country.name }}</div>
                        <div class="bar-sub" v-if="country.hasData">
                            Score {{ country.score?.toFixed(2) }} • Rank {{ country.rank }}
                        </div>
                        <div class="bar-sub" v-else>No data</div>
                    </div>
                    <div class="bar-track">
                        <div
                            v-if="country.hasData"
                            class="bar"
                            :style="{ width: `${((country.score ?? 0) / 10) * 100}%` }"
                        >
                            <div
                                v-for="(factor, index) in country.factors"
                                :key="factor.label"
                                class="bar-segment"
                                :style="{
                                    background: segmentColors[index],
                                    width: `${((factor.value / (country.score ?? 1)) * 100).toFixed(2)}%`
                                }"
                                @mouseenter="showTooltip($event, factor.label, factor.value)"
                                @mousemove="moveTooltip($event)"
                                @mouseleave="hideTooltip"
                            ></div>
                        </div>
                    </div>
                </div>
            </div>
            <div class="legend">
                <div class="legend-item" v-for="(label, index) in legendLabels" :key="label">
                    <span class="legend-swatch" :style="{ background: segmentColors[index] }"></span>
                    <span>{{ label }}</span>
                </div>
            </div>
        </div>
        <div v-else class="panel-empty">Select up to five countries to compare factors.</div>
        <div v-if="tooltip.visible" class="segment-tooltip" :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }">
            {{ tooltip.text }}
        </div>
    </aside>
</template>

<style scoped>
.panel {
    position: relative;
    width: 0;
    flex: 0 0 0;
    max-width: 360px;
    overflow: hidden;
    background: #fbfbfc;
    border-left: 1px solid #e0e4ea;
    transition: width 260ms ease, transform 260ms ease;
    transform: translateX(20px);
}

.segment-tooltip {
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

.panel.open {
    width: 360px;
    flex: 0 0 360px;
    transform: translateX(0);
}

.panel-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 16px 16px 8px;
    border-bottom: 1px solid #e6e9ef;
}

.panel-header h2 {
    font-size: 1rem;
    margin: 0;
    color: #1f2f4a;
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
    color: #1f2f4a;
}

.panel-meta {
    font-size: 0.85rem;
    color: #5a6577;
    margin-bottom: 12px;
}

.legend {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 6px 12px;
    margin-top: 28px;
    margin-bottom: 8px;
    font-size: 0.75rem;
    color: #344055;
}

.legend-item {
    display: flex;
    align-items: center;
    gap: 6px;
}

.legend-swatch {
    width: 12px;
    height: 12px;
    border-radius: 3px;
    border: 1px solid rgba(0, 0, 0, 0.08);
}

.bar-list {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.bar-row {
    display: grid;
    grid-template-columns: 1fr;
    gap: 6px;
}

.bar-label {
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.bar-name {
    font-weight: 600;
    font-size: 0.9rem;
}

.bar-sub {
    font-size: 0.75rem;
    color: #5a6577;
}

.bar-track {
    height: 18px;
    background: #eef2f7;
    border-radius: 8px;
    overflow: hidden;
}

.bar {
    display: flex;
    height: 100%;
}

.bar-segment {
    height: 100%;
}

.panel-empty {
    padding: 16px;
    font-size: 0.9rem;
    color: #5a6577;
}
</style>
