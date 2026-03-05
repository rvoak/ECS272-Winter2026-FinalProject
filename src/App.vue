<script setup lang="ts">
import { ref } from 'vue'
import HappinessMap from './components/HappinessMap.vue'
import FactorComparisonPanel from './components/FactorComparisonPanel.vue'
import MoversList from './components/MoversList.vue'
import MoversDetailPanel from './components/MoversDetailPanel.vue'

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

const selectedCountries = ref<PanelData[]>([])
const activeView = ref<'map' | 'movers'>('map')
const moversSelection = ref<{
  name: string | null
  series: { year: number; score: number; rank: number; factors: { label: string; value: number }[] }[]
  mode: 'score' | 'rank'
  options: { name: string; series: { year: number; score: number; rank: number; factors: { label: string; value: number }[] }[] }[]
}>({
  name: null,
  series: [],
  mode: 'score',
  options: []
})

const onCountrySelected = (payload: PanelData) => {
  const existingIndex = selectedCountries.value.findIndex((item) => item.key === payload.key)
  if (existingIndex >= 0) {
    selectedCountries.value.splice(existingIndex, 1)
    return
  }
  if (selectedCountries.value.length >= 5) {
    selectedCountries.value.shift()
  }
  selectedCountries.value.push(payload)
}
</script>

<!-- Using Vuetify components (Vuetify should be registered in the app entry). -->
<template>
  <VContainer id="main-container" class="d-flex flex-column" fluid>
    <div class="top-bar">
      <button class="view-toggle" type="button" @click="activeView = activeView === 'map' ? 'movers' : 'map'">
        {{ activeView === 'map' ? 'SHOW MOVERS LIST' : 'SHOW MAP' }}
      </button>
    </div>
    <div v-if="activeView === 'map'" class="dashboard-layout" :class="{ 'panel-open': selectedCountries.length > 0 }">
      <div class="map-shell">
        <HappinessMap :selectedKeys="selectedCountries.map((item) => item.key)" @country-selected="onCountrySelected" />
      </div>
      <FactorComparisonPanel
        :open="selectedCountries.length > 0"
        :data="selectedCountries"
        @close="selectedCountries = []"
      />
    </div>
    <div v-else class="movers-shell">
      <div class="movers-layout" :class="{ 'panel-open': !!moversSelection.name }">
        <div class="movers-list-shell">
          <MoversList @country-selected="(payload) => (moversSelection = payload)" />
        </div>
        <MoversDetailPanel
          :open="!!moversSelection.name"
          :name="moversSelection.name"
          :series="moversSelection.series"
          :mode="moversSelection.mode"
          :options="moversSelection.options"
          @close="moversSelection = { name: null, series: [], mode: 'score', options: [] }"
        />
      </div>
    </div>
  </VContainer>
</template>

<style scoped>
#main-container{
  height: 100%;
}

.top-bar {
  display: flex;
  justify-content: flex-start;
  padding: 8px 12px 0;
}

.view-toggle {
  border: 1px solid #b7c3d6;
  border-radius: 8px;
  padding: 8px 12px;
  font-size: 0.85rem;
  font-weight: 600;
  background: #eef3f9;
  color: #1f2f4a;
  cursor: pointer;
}

.view-toggle:hover {
  background: #e3ecf7;
}

.movers-shell {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

.movers-layout {
  display: flex;
  height: 100%;
}

.movers-list-shell {
  flex: 1 1 auto;
  min-width: 0;
}

.dashboard-layout {
  display: flex;
  height: 100%;
  transition: all 260ms ease;
}

.map-shell {
  flex: 1 1 auto;
  min-width: 0;
  transition: flex 260ms ease;
}

.dashboard-layout.panel-open .map-shell {
  flex: 1 1 auto;
}
</style>
