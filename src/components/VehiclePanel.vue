<template>
  <div class="vehicle-panel">
    <div class="panel-header">
      <div class="header-icon">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <rect x="1" y="3" width="15" height="13" rx="2"/>
          <polygon points="16 8 20 8 23 11 23 16 16 16 16 8"/>
          <circle cx="5.5" cy="18.5" r="2.5"/>
          <circle cx="18.5" cy="18.5" r="2.5"/>
        </svg>
      </div>
      <span class="header-title">车辆监控面板</span>
      <span class="vehicle-count">{{ vehicles.length }} 辆</span>
    </div>
    <div class="panel-body">
      <div
        v-for="v in vehicles"
        :key="v.id"
        class="vehicle-card"
        :class="{ active: selectedVehicle && selectedVehicle.id === v.id }"
        @click="$emit('select-vehicle', v)"
      >
        <div class="card-header">
          <div class="vehicle-icon" :style="{ backgroundColor: v.color }">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="white" stroke="none">
              <path d="M18.92 6.01C18.72 5.42 18.16 5 17.5 5h-11c-.66 0-1.21.42-1.42 1.01L3 12v8c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-1h12v1c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-8l-2.08-5.99zM6.5 16c-.83 0-1.5-.67-1.5-1.5S5.67 13 6.5 13s1.5.67 1.5 1.5S7.33 16 6.5 16zm11 0c-.83 0-1.5-.67-1.5-1.5s.67-1.5 1.5-1.5 1.5.67 1.5 1.5-.67 1.5-1.5 1.5zM5 11l1.5-4.5h11L19 11H5z"/>
            </svg>
          </div>
          <div class="vehicle-name">
            <span class="name">{{ v.name }}</span>
            <span class="type">{{ v.type }}</span>
          </div>
          <div class="status-badge" :class="v.status === '运行中' ? 'running' : 'warning'">
            {{ v.status }}
          </div>
        </div>
        <div class="card-body">
          <div class="info-row">
            <div class="info-item">
              <span class="label">速度</span>
              <span class="value">{{ v.speed }} km/h</span>
            </div>
            <div class="info-item">
              <span class="label">电量</span>
              <div class="battery-bar">
                <div
                  class="battery-fill"
                  :style="{
                    width: v.battery + '%',
                    backgroundColor: v.battery > 50 ? '#4caf50' : v.battery > 20 ? '#ff9800' : '#f44336'
                  }"
                ></div>
              </div>
              <span class="value small">{{ v.battery }}%</span>
            </div>
          </div>
          <div class="info-row">
            <div class="info-item">
              <span class="label">里程</span>
              <span class="value">{{ v.distance }} km</span>
            </div>
            <div class="info-item">
              <span class="label">航向</span>
              <span class="value">{{ v.heading }}°</span>
            </div>
          </div>
          <div class="info-row full">
            <div class="info-item">
              <span class="label">坐标</span>
              <span class="value mono">{{ v.lon }}, {{ v.lat }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  vehicles: { type: Array, default: () => [] },
  selectedVehicle: { type: Object, default: null },
})
defineEmits(['select-vehicle'])
</script>

<style scoped>
.vehicle-panel {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 340px;
  max-height: calc(100vh - 32px);
  background: rgba(13, 17, 28, 0.88);
  border: 1px solid rgba(100, 181, 246, 0.2);
  border-radius: 12px;
  backdrop-filter: blur(16px);
  color: #e0e0e0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  z-index: 100;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.panel-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 16px;
  border-bottom: 1px solid rgba(100, 181, 246, 0.15);
  background: rgba(21, 101, 192, 0.15);
}
.header-icon {
  color: #64b5f6;
  display: flex;
  align-items: center;
}
.header-title {
  font-size: 15px;
  font-weight: 600;
  color: #e3f2fd;
}
.vehicle-count {
  margin-left: auto;
  font-size: 12px;
  color: #64b5f6;
  background: rgba(100, 181, 246, 0.12);
  padding: 2px 8px;
  border-radius: 10px;
}
.panel-body {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.panel-body::-webkit-scrollbar {
  width: 4px;
}
.panel-body::-webkit-scrollbar-thumb {
  background: rgba(100, 181, 246, 0.3);
  border-radius: 2px;
}
.vehicle-card {
  background: rgba(30, 40, 60, 0.7);
  border: 1px solid rgba(100, 181, 246, 0.1);
  border-radius: 10px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.2s ease;
}
.vehicle-card:hover {
  border-color: rgba(100, 181, 246, 0.4);
  background: rgba(30, 50, 80, 0.8);
}
.vehicle-card.active {
  border-color: #64b5f6;
  background: rgba(21, 101, 192, 0.2);
  box-shadow: 0 0 12px rgba(100, 181, 246, 0.15);
}
.card-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}
.vehicle-icon {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.vehicle-name {
  flex: 1;
  display: flex;
  flex-direction: column;
}
.vehicle-name .name {
  font-size: 14px;
  font-weight: 600;
  color: #e3f2fd;
}
.vehicle-name .type {
  font-size: 11px;
  color: #90a4ae;
}
.status-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 500;
}
.status-badge.running {
  color: #4caf50;
  background: rgba(76, 175, 80, 0.15);
  border: 1px solid rgba(76, 175, 80, 0.3);
}
.status-badge.warning {
  color: #ff9800;
  background: rgba(255, 152, 0, 0.15);
  border: 1px solid rgba(255, 152, 0, 0.3);
}
.card-body {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.info-row {
  display: flex;
  gap: 12px;
}
.info-row.full {
  margin-top: 2px;
}
.info-item {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 6px;
}
.info-item .label {
  font-size: 11px;
  color: #78909c;
  white-space: nowrap;
  min-width: 28px;
}
.info-item .value {
  font-size: 12px;
  color: #cfd8dc;
  font-weight: 500;
}
.info-item .value.mono {
  font-family: 'Courier New', monospace;
  font-size: 11px;
}
.info-item .value.small {
  font-size: 11px;
  margin-left: 4px;
}
.battery-bar {
  flex: 1;
  height: 6px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 3px;
  overflow: hidden;
}
.battery-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s ease;
}
</style>
