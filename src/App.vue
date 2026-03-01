<script setup lang="ts">
import { ref } from 'vue'
import HappinessMap from './components/HappinessMap.vue'
import FactorComparisonPanel from './components/FactorComparisonPanel.vue'

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
    <div class="dashboard-layout" :class="{ 'panel-open': selectedCountries.length > 0 }">
      <div class="map-shell">
        <HappinessMap :selectedKeys="selectedCountries.map((item) => item.key)" @country-selected="onCountrySelected" />
      </div>
      <FactorComparisonPanel
        :open="selectedCountries.length > 0"
        :data="selectedCountries"
        @close="selectedCountries = []"
      />
    </div>
  </VContainer>
</template>

<style scoped>
#main-container{
  height: 100%;
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
