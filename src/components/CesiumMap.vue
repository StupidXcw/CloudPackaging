<template>
  <div ref="cesiumContainer" class="cesium-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import * as Cesium from "cesium";

const emit = defineEmits(["vehicle-update"]);
const cesiumContainer = ref(null);

const CENTER_LON = 120.3726;
const CENTER_LAT = 31.5564;

const VEHICLE_COLORS = [
  Cesium.Color.fromCssColorString("#00e5ff"),
  Cesium.Color.fromCssColorString("#76ff03"),
  Cesium.Color.fromCssColorString("#ffea00"),
  Cesium.Color.fromCssColorString("#ff6d00"),
  Cesium.Color.fromCssColorString("#d500f9"),
];

let viewer = null;
let animationFrameId = null;
let vehicles = [];
let pathEntities = [];

function generateCircularPath(centerLon, centerLat, radius, numPoints) {
  const path = [];
  for (let i = 0; i <= numPoints; i++) {
    const angle = (i / numPoints) * Math.PI * 2;
    const lon = centerLon + (radius * Math.cos(angle)) / 111320;
    const lat = centerLat + (radius * Math.sin(angle)) / 110540;
    path.push({ lon, lat });
  }
  return path;
}

function generateRectPath(centerLon, centerLat, halfW, halfH, numPoints) {
  const hw = halfW / 111320;
  const hh = halfH / 110540;
  const corners = [
    { lon: centerLon - hw, lat: centerLat - hh },
    { lon: centerLon + hw, lat: centerLat - hh },
    { lon: centerLon + hw, lat: centerLat + hh },
    { lon: centerLon - hw, lat: centerLat + hh },
  ];
  const path = [];
  const perSide = Math.floor(numPoints / 4);
  for (let i = 0; i < 4; i++) {
    const from = corners[i];
    const to = corners[(i + 1) % 4];
    for (let j = 0; j < perSide; j++) {
      const t = j / perSide;
      path.push({
        lon: from.lon + (to.lon - from.lon) * t,
        lat: from.lat + (to.lat - from.lat) * t,
      });
    }
  }
  path.push(path[0]);
  return path;
}

function generateSpiralPath(centerLon, centerLat, maxRadius, turns, numPoints) {
  const path = [];
  for (let i = 0; i <= numPoints; i++) {
    const t = i / numPoints;
    const angle = turns * 2 * Math.PI * t;
    const r = maxRadius * t;
    const lon = centerLon + (r * Math.cos(angle)) / 111320;
    const lat = centerLat + (r * Math.sin(angle)) / 110540;
    path.push({ lon, lat });
  }
  return path;
}

function generateRandomPath(centerLon, centerLat, radius, numPoints) {
  const path = [];
  let lon = centerLon;
  let lat = centerLat;
  for (let i = 0; i <= numPoints; i++) {
    const angle = Math.random() * Math.PI * 2;
    const step = radius / numPoints / 111320;
    lon += Math.cos(angle) * step;
    lat += (Math.sin(angle) * step) / (111320 / 110540);
    const dlon = lon - centerLon;
    const dlat = lat - centerLat;
    const dist = Math.sqrt(dlon * dlon + dlat * dlat);
    const maxDist = radius / 111320;
    if (dist > maxDist) {
      lon = centerLon + (dlon / dist) * maxDist * 0.8;
      lat = centerLat + (dlat / dist) * maxDist * 0.8;
    }
    path.push({ lon, lat });
  }
  return path;
}

function initVehicles() {
  const configs = [
    {
      name: "AGV-001",
      pathFn: () => generateCircularPath(CENTER_LON, CENTER_LAT, 200, 60),
      speed: 15,
      type: "AGV搬运车",
    },
    {
      name: "AGV-002",
      pathFn: () =>
        generateCircularPath(CENTER_LON + 0.002, CENTER_LAT + 0.001, 150, 50),
      speed: 12,
      type: "AGV搬运车",
    },
    {
      name: "VAN-001",
      pathFn: () =>
        generateRectPath(CENTER_LON - 0.001, CENTER_LAT, 300, 200, 80),
      speed: 25,
      type: "物流货车",
    },
    {
      name: "PATROL-001",
      pathFn: () => generateSpiralPath(CENTER_LON, CENTER_LAT, 350, 3, 100),
      speed: 8,
      type: "巡检车",
    },
    {
      name: "SHUTTLE-001",
      pathFn: () =>
        generateRandomPath(CENTER_LON + 0.001, CENTER_LAT - 0.001, 400, 120),
      speed: 20,
      type: "穿梭车",
    },
  ];

  vehicles = configs.map((cfg, i) => {
    const path = cfg.pathFn();
    return {
      id: i,
      name: cfg.name,
      type: cfg.type,
      speed: cfg.speed,
      path,
      pathIndex: 0,
      progress: 0,
      currentLon: path[0].lon,
      currentLat: path[0].lat,
      heading: 0,
      battery: Math.floor(Math.random() * 40) + 60,
      status: "运行中",
      distance: 0,
      color: VEHICLE_COLORS[i % VEHICLE_COLORS.length],
    };
  });
}

function drawPaths() {
  vehicles.forEach((v) => {
    const positions = v.path.map((p) =>
      Cesium.Cartesian3.fromDegrees(p.lon, p.lat),
    );
    const pathEntity = viewer.entities.add({
      polyline: {
        positions,
        width: 2,
        material: new Cesium.PolylineGlowMaterialProperty({
          glowPower: 0.2,
          color: v.color.withAlpha(0.5),
        }),
        clampToGround: true,
      },
    });
    pathEntities.push(pathEntity);
  });
}

function createVehicleEntities() {
  vehicles.forEach((v) => {
    v.entity = viewer.entities.add({
      position: new Cesium.CallbackProperty(() => {
        return Cesium.Cartesian3.fromDegrees(v.currentLon, v.currentLat);
      }, false),
      billboard: {
        image: createVehicleIcon(v.color),
        verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
        scale: 1.0,
        disableDepthTestDistance: Number.POSITIVE_INFINITY,
      },
      label: {
        text: v.name,
        font: "12px sans-serif",
        fillColor: Cesium.Color.WHITE,
        outlineColor: Cesium.Color.BLACK,
        outlineWidth: 2,
        style: Cesium.LabelStyle.FILL_AND_OUTLINE,
        verticalOrigin: Cesium.VerticalOrigin.TOP,
        pixelOffset: new Cesium.Cartesian2(0, 15),
        disableDepthTestDistance: Number.POSITIVE_INFINITY,
      },
    });

    v.trailPositions = [];
    v.trailEntity = viewer.entities.add({
      polyline: {
        positions: new Cesium.CallbackProperty(() => {
          return v.trailPositions.slice();
        }, false),
        width: 3,
        material: new Cesium.PolylineGlowMaterialProperty({
          glowPower: 0.3,
          color: v.color.withAlpha(0.7),
        }),
        clampToGround: true,
      },
    });
  });
}

function createVehicleIcon(color) {
  const canvas = document.createElement("canvas");
  canvas.width = 48;
  canvas.height = 48;
  const ctx = canvas.getContext("2d");

  const colorStr = color.toCssColorString();

  // 外发光圈
  ctx.beginPath();
  ctx.arc(24, 24, 20, 0, Math.PI * 2);
  ctx.fillStyle = colorStr.replace(")", ", 0.3)").replace("rgb", "rgba");
  ctx.fill();

  // 内圈
  ctx.beginPath();
  ctx.arc(24, 24, 14, 0, Math.PI * 2);
  ctx.fillStyle = colorStr;
  ctx.fill();
  ctx.strokeStyle = "white";
  ctx.lineWidth = 2;
  ctx.stroke();

  // 中心点
  ctx.beginPath();
  ctx.arc(24, 24, 6, 0, Math.PI * 2);
  ctx.fillStyle = "white";
  ctx.fill();

  // 方向指示（向上箭头）
  ctx.beginPath();
  ctx.moveTo(24, 8);
  ctx.lineTo(18, 18);
  ctx.lineTo(30, 18);
  ctx.closePath();
  ctx.fillStyle = "white";
  ctx.fill();

  return canvas;
}

function updateVehiclePositions(deltaTime) {
  const updatedData = [];

  vehicles.forEach((v) => {
    v.progress += v.speed * deltaTime * 0.01;
    if (v.progress >= 1) {
      v.progress = 0;
      v.pathIndex = (v.pathIndex + 1) % (v.path.length - 1);
    }

    const currentIdx = v.pathIndex;
    const nextIdx = Math.min(currentIdx + 1, v.path.length - 1);
    const from = v.path[currentIdx];
    const to = v.path[nextIdx];

    const newLon = from.lon + (to.lon - from.lon) * v.progress;
    const newLat = from.lat + (to.lat - from.lat) * v.progress;

    const dLon = newLon - v.currentLon;
    const dLat = newLat - v.currentLat;
    v.heading = Math.atan2(dLon, dLat) * (180 / Math.PI);

    const distStep = Math.sqrt(dLon * dLon + dLat * dLat) * 111320;
    v.distance += distStep;

    v.currentLon = newLon;
    v.currentLat = newLat;

    v.battery = Math.max(10, v.battery - deltaTime * 0.002);
    if (v.battery < 20) {
      v.status = "低电量";
    } else {
      v.status = "运行中";
    }

    v.trailPositions.push(
      Cesium.Cartesian3.fromDegrees(v.currentLon, v.currentLat),
    );
    if (v.trailPositions.length > 200) {
      v.trailPositions.shift();
    }

    updatedData.push({
      id: v.id,
      name: v.name,
      type: v.type,
      speed: (v.speed + Math.random() * 2 - 1).toFixed(1),
      status: v.status,
      battery: Math.floor(v.battery),
      distance: (v.distance / 1000).toFixed(2),
      lon: v.currentLon.toFixed(6),
      lat: v.currentLat.toFixed(6),
      heading: v.heading.toFixed(1),
      color: v.color.toCssColorString(),
    });
  });

  emit("vehicle-update", updatedData);
}

function animate() {
  if (!viewer) return;
  const delta =
    viewer.clock.currentTime.secondsOfDay -
    (viewer.clock.startTime.secondsOfDay || 0);
  updateVehiclePositions(0.016);
  animationFrameId = requestAnimationFrame(animate);
}

function flyToVehicle(vehicle) {
  const v = vehicles.find((item) => item.id === vehicle.id);
  if (v) {
    viewer.camera.flyTo({
      destination: Cesium.Cartesian3.fromDegrees(
        v.currentLon,
        v.currentLat,
        500,
      ),
      duration: 1.5,
    });
  }
}

onMounted(() => {
  Cesium.Ion.defaultAccessToken =
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI3ZjQ5ZGUzNC1jNWFhLTQ0MDktYjk5OC1iN2FhYTVmYzQ0NDgiLCJpZCI6MjU5LCJpYXQiOjE3MjY0NjYxODN9.Wnfaj7o3S1vLk_YG2tPnKkFXxCPMNIHPIc5OjVElxkc";

  // 使用 ArcGIS 卫星影像作为底图（更稳定）
  const arcgisImagery = new Cesium.ArcGisMapServerImageryProvider({
    url: "https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer",
    enablePickFeatures: false,
  });

  // 使用高德地图注记
  const gaodeLabel = new Cesium.UrlTemplateImageryProvider({
    url: "https://webrd0{s}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=8&x={x}&y={y}&z={z}",
    subdomains: ["1", "2", "3", "4"],
    maximumLevel: 18,
    credit: new Cesium.Credit("高德地图"),
  });

  viewer = new Cesium.Viewer(cesiumContainer.value, {
    animation: false,
    timeline: false,
    baseLayerPicker: false,
    geocoder: false,
    homeButton: false,
    sceneModePicker: false,
    navigationHelpButton: false,
    fullscreenButton: false,
    infoBox: false,
    selectionIndicator: false,
    shouldAnimate: true,
    imageryProvider: arcgisImagery,
  });

  // 添加高德注记图层
  viewer.imageryLayers.addImageryProvider(gaodeLabel);

  // 设置相机为正俯视视角（2D视角，pitch -90度）
  viewer.camera.setView({
    destination: Cesium.Cartesian3.fromDegrees(CENTER_LON, CENTER_LAT, 1200),
    orientation: {
      heading: Cesium.Math.toRadians(0),
      pitch: Cesium.Math.toRadians(-90),
      roll: 0,
    },
  });

  viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(CENTER_LON, CENTER_LAT),
    label: {
      text: "无锡国家传感信息中心 D1栋",
      font: "bold 16px sans-serif",
      fillColor: Cesium.Color.YELLOW,
      outlineColor: Cesium.Color.BLACK,
      outlineWidth: 3,
      style: Cesium.LabelStyle.FILL_AND_OUTLINE,
      verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
      pixelOffset: new Cesium.Cartesian2(0, -30),
      disableDepthTestDistance: Number.POSITIVE_INFINITY,
    },
    point: {
      pixelSize: 12,
      color: Cesium.Color.YELLOW,
      outlineColor: Cesium.Color.BLACK,
      outlineWidth: 2,
      disableDepthTestDistance: Number.POSITIVE_INFINITY,
    },
  });

  const buildingPositions = [
    Cesium.Cartesian3.fromDegrees(CENTER_LON - 0.0008, CENTER_LAT - 0.0005),
    Cesium.Cartesian3.fromDegrees(CENTER_LON + 0.0008, CENTER_LAT - 0.0005),
    Cesium.Cartesian3.fromDegrees(CENTER_LON + 0.0008, CENTER_LAT + 0.0005),
    Cesium.Cartesian3.fromDegrees(CENTER_LON - 0.0008, CENTER_LAT + 0.0005),
  ];
  viewer.entities.add({
    polygon: {
      hierarchy: buildingPositions,
      material: Cesium.Color.fromCssColorString("#1565C0").withAlpha(0.4),
      outline: true,
      outlineColor: Cesium.Color.fromCssColorString("#42A5F5"),
      height: 0,
      extrudedHeight: 60,
    },
  });

  initVehicles();
  drawPaths();
  createVehicleEntities();
  animate();
});

onBeforeUnmount(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
  if (viewer) {
    viewer.destroy();
    viewer = null;
  }
});

defineExpose({ flyToVehicle });
</script>

<style scoped>
.cesium-container {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
}
</style>
