<!-- pages/index.vue -->

<template>
    <div class="widget-container">
        <!-- Lower container for the map widget -->
        <div class="lower-container">
            <MapWidget v-for="(widget, index) in lowerWidgets" :key="index" :data="widget" />
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import MapWidget from '~/components/MapWidget.vue';

const widgets = ref([]);

const lowerWidgets = computed(() => widgets.value.filter((widget) => widget.type === 'map'));

onMounted(async () => {
    try {
        const response = await fetch('/data/widgets.json');
        if (!response.ok) {
            throw new Error('Failed to fetch widgets');
        }
        widgets.value = await response.json();
    } catch (error) {
        console.error(error.message);
    }
});
</script>

<style scoped>
@import '../css/main.css';

/* Lower container for the map widget */
.lower-container {
    display: flex;
    justify-content: center;
    flex-direction: column;
    /* Измените на column для мобильных устройств */
    border-radius: var(--border-radius);
    width: 100%;
    padding: 20px;
    background-color: rgba(0, 0, 0, 0.8);
}

@media screen and (max-width: 600px) {
    .lower-container {
        flex-direction: column;
    }
}
</style>
