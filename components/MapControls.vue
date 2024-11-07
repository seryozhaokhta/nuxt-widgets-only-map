<!-- components/MapControls.vue -->

<template>
    <div class="map-controls">
        <button class="zoom-button" @click="$emit('zoomIn')">+</button>
        <button class="zoom-button" @click="$emit('zoomOut')">−</button>
        <button class="fullscreen-button" @click="$emit('toggleFullScreen')">
            <span v-if="isFullScreen">⤢</span>
            <span v-else>⤡</span>
        </button>
        <MapCustomSlider v-model="currentYearProxy" :tooltip="true" :tooltipFormatter="formatYearTooltip" :min="minYear"
            :max="maxYear" :step="50" class="map-slider" />
    </div>
</template>

<script setup>
import { computed } from 'vue';
import MapCustomSlider from './MapCustomSlider.vue';
import { formatYearTooltip } from '@/composables/mapUtils';

const props = defineProps({
    currentYear: {
        type: Number,
        required: true,
    },
    minYear: {
        type: Number,
        required: true,
    },
    maxYear: {
        type: Number,
        required: true,
    },
    isFullScreen: {
        type: Boolean,
        default: false,
    },
});

const emits = defineEmits(['update:currentYear', 'zoomIn', 'zoomOut', 'toggleFullScreen']);

const currentYearProxy = computed({
    get() {
        return props.currentYear;
    },
    set(value) {
        emits('update:currentYear', value);
    },
});
</script>

<style scoped>
.map-controls {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
    gap: 10px;
    margin-top: 10px;
}

.zoom-button,
.fullscreen-button {
    padding: 5px 10px;
    font-size: 18px;
    cursor: pointer;
    background-color: #ff0505;
    color: #fff;
    border: none;
    border-radius: 4px;
    transition: background-color 0.3s;
}

.zoom-button:hover,
.fullscreen-button:hover {
    background-color: #e60000;
}

.map-slider {
    flex: 1;
    margin: 0 10px;
}

@media screen and (max-width: 600px) {
    .map-controls {
        flex-direction: row;
    }

    .map-slider {
        margin: 10px 0;
    }
}
</style>

