<template>
  <div class="app-container">
    <CesiumMap ref="cesiumMap" @vehicle-update="onVehicleUpdate" />
    <VehiclePanel :vehicles="vehicles" :selected-vehicle="selectedVehicle" @select-vehicle="onSelectVehicle" />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import CesiumMap from './components/CesiumMap.vue'
import VehiclePanel from './components/VehiclePanel.vue'

const vehicles = ref([])
const selectedVehicle = ref(null)
const cesiumMap = ref(null)

function onVehicleUpdate(data) {
  vehicles.value = data
}

function onSelectVehicle(vehicle) {
  selectedVehicle.value = vehicle
  if (cesiumMap.value) {
    cesiumMap.value.flyToVehicle(vehicle)
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
html, body, #app {
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.app-container {
  position: relative;
  width: 100%;
  height: 100%;
}
</style>
