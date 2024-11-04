<!-- components/MapBoundaries.vue -->

<template>
    <svg class="map-boundaries" :style="svgStyle" viewBox="0 0 100 100" preserveAspectRatio="none">
        <g v-for="(feature, index) in filteredFeatures" :key="index">
            <polygon :points="getPolygonPoints(feature.geometry.coordinates)"
                :fill="getFillColor(feature.properties.id)" stroke="white" :stroke-width="0.5"
                vector-effect="non-scaling-stroke" @click="selectBoundary(feature)" />
        </g>
    </svg>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';

const props = defineProps({
    currentYear: {
        type: Number,
        required: true,
    },
    zoomScale: {
        type: Number,
        required: true,
    },
    mapPosition: {
        type: Object,
        required: true,
    },
    currentPeriodId: {
        type: String,
        required: false,
    },
});

const emits = defineEmits(['selectBoundary']);

const geoJsonData = ref(null);
const filteredFeatures = ref([]);

async function loadGeoJson() {
    try {
        const response = await fetch('/data/boundaries.geojson');
        if (!response.ok) {
            throw new Error(`Failed to load GeoJSON data: ${response.statusText}`);
        }
        geoJsonData.value = await response.json();
        filterFeatures();
    } catch (error) {
        console.error('Error loading GeoJSON:', error);
    }
}

function filterFeatures() {
    if (!geoJsonData.value) return;

    filteredFeatures.value = geoJsonData.value.features.filter((feature) => {
        const withinYearRange =
            feature.properties.startYear <= props.currentYear &&
            feature.properties.endYear >= props.currentYear;

        return withinYearRange;
    });
}

function getPolygonPoints(coordinates) {
    return coordinates[0]
        .map((coord) => {
            const [lon, lat] = coord;
            const x = ((lon + 180) / 360) * 100;
            const latRad = (lat * Math.PI) / 180;
            const mercN = Math.log(Math.tan(Math.PI / 4 + latRad / 2));
            const y = (1 - mercN / Math.PI) * 50;
            return `${x},${y}`;
        })
        .join(' ');
}

function getFillColor(id) {
    if (props.currentPeriodId && id === props.currentPeriodId) {
        return 'rgba(255, 0, 0, 0.5)'; // Выделенный период
    } else {
        return 'rgba(0, 0, 255, 0.3)'; // Остальные периоды
    }
}

function selectBoundary(feature) {
    emits('selectBoundary', feature);
}

const svgStyle = computed(() => ({
    position: 'absolute',
    top: 0,
    left: 0,
    width: '100%',
    height: '100%',
    transformOrigin: 'top left',
    pointerEvents: 'none',
    zIndex: 2,
}));

onMounted(() => {
    loadGeoJson();
});

watch(
    () => [props.currentYear, props.currentPeriodId],
    () => {
        filterFeatures();
    }
);
</script>

<style scoped>
.map-boundaries {
    pointer-events: none;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 2;
}

.map-boundaries polygon {
    cursor: pointer;
    pointer-events: auto;
    transition: fill 0.3s;
}

.map-boundaries polygon:hover {
    fill-opacity: 0.7;
}
</style>
