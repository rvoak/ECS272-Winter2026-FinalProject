<script setup lang="ts">
import HappinessMap from './components/HappinessMap.vue'
</script>

<!-- Using Vuetify components (Vuetify should be registered in the app entry). -->
<template>
  <VContainer id="main-container" class="d-flex flex-column" fluid>
    <VRow class="pa-0 fill-height">
      <VCol class="pa-0 fill-height">
        <HappinessMap />
      </VCol>
    </VRow>
  </VContainer>
</template>

<style scoped>
#main-container{
  height: 100%;
}
</style>
