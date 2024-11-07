<!-- components/MapWidget.vue -->

<template>
    <div class="map-widget" :class="{ 'map-widget--fullscreen': isFullScreen }" ref="mapWidget">
        <div class="map-widget__current-year">{{ formattedCurrentYear }}</div>

        <div class="map-widget__container" ref="mapContainer" @wheel="handleWheel" @mousedown="startPan"
            @mousemove="panMap" @mouseup="endPan" @mouseleave="endPan" @touchstart="startPanTouch"
            @touchmove="panMapTouch" @touchend="endPan">
            <div class="map-widget__content" :style="mapStyle">
                <!-- Изображение карты -->
                <img ref="mapImage" :src="mapSrc" alt="Карта мира" class="map-widget__image" />

                <!-- Компонент границ -->
                <MapBoundaries :currentYear="currentYear" :zoomScale="zoomScale" :mapPosition="mapPosition"
                    :currentPeriodId="currentPeriodId" @selectBoundary="selectBoundary" />

                <!-- Точки на карте -->
                <MapPoint v-for="point in sortedPoints" :key="point.id" :point="point" :currentYear="currentYear"
                    :zoomScale="zoomScale" @selectPoint="selectPoint" />
            </div>

            <!-- Модальное окно -->
            <ModalWindow v-if="modalVisible" ref="modalRef" @close="closeModal">
                <template #header>
                    <h2 class="modal-title">{{ selectedPoint.name }}</h2>
                    <p class="modal-founded">
                        {{ selectedPoint.founded }} <sup>Основание</sup>
                    </p>
                </template>
                <template #body>
                    <!-- Отображение текущего периода, если есть -->
                    <div v-if="currentPeriod">
                        <h3 class="modal-period-title">{{ currentPeriod.title }}</h3>
                        <p class="modal-period-description">
                            {{ currentPeriod.description }}
                        </p>
                        <p class="modal-period-year">
                            <em>{{ currentPeriod.yearRange }}</em>
                        </p>
                    </div>
                    <p v-else class="modal-description">
                        {{ selectedPoint.description }}
                    </p>

                    <!-- Таймлайн -->
                    <MapTimeline v-if="selectedPoint.periods" :periods="selectedPoint.periods"
                        v-model:currentPeriodIndex="currentPeriodIndex" @periodSelected="onPeriodSelected" />
                </template>
            </ModalWindow>
        </div>

        <MapControls v-model:currentYear="currentYear" :minYear="minYear" :maxYear="maxYear" @zoomIn="zoomIn"
            @zoomOut="zoomOut" @toggleFullScreen="toggleFullScreen" :isFullScreen="isFullScreen" />
    </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount, watch, nextTick } from 'vue';
import MapControls from './MapControls.vue';
import MapPoint from './MapPoint.vue';
import ModalWindow from './ModalWindow.vue';
import MapTimeline from './MapTimeline.vue';
import MapBoundaries from './MapBoundaries.vue';
import { formatYearTooltip } from '@/composables/mapUtils';
import { minZoom, maxZoom, zoomStep } from '@/composables/constants';
import gsap from 'gsap';
import debounce from 'lodash/debounce';

const props = defineProps({
    data: {
        type: Object,
        required: true,
    },
});

const mapSrc = ref('/assets/map/World_map_blank_without_borders.svg');
const mapPoints = ref([]);
const sortedPoints = ref([]);
const currentYear = ref(0);
const zoomScale = ref(1);
const mapPosition = ref({ x: 0, y: 0 });
const isPanning = ref(false);
const startPanPosition = ref({ x: 0, y: 0 });
const mapContainer = ref(null);
const selectedPoint = ref(null);
const modalVisible = ref(false);
const modalRef = ref(null);
const pendingPoint = ref(null);
const currentPeriodIndex = ref(0);
const isFullScreen = ref(false);

const minYear = ref(0);
const maxYear = ref(0);

function parseYearRange(yearRange) {
    const sanitized = yearRange.replace(/[^\d–BCECE]/g, '');
    const rangeMatch = sanitized.match(/(\d+)(BCE|CE)?–(\d+)(BCE|CE)?/);
    const singleMatch = sanitized.match(/^(\d+)(BCE|CE)?$/);

    if (rangeMatch) {
        let [_, startYear, startEra, endYear, endEra] = rangeMatch;
        startYear = parseInt(startYear, 10);
        endYear = parseInt(endYear, 10);
        startEra = startEra || 'BCE';
        endEra = endEra || 'BCE';

        startYear = startEra === 'BCE' ? -startYear : startYear;
        endYear = endEra === 'BCE' ? -endYear : endYear;

        return {
            startYear: Math.min(startYear, endYear),
            endYear: Math.max(startYear, endYear),
        };
    } else if (singleMatch) {
        let [_, year, era] = singleMatch;
        year = parseInt(year, 10);
        era = era || 'BCE';
        year = era === 'BCE' ? -year : year;
        return {
            startYear: year,
            endYear: year,
        };
    }

    return null;
}

onMounted(() => {
    mapPoints.value = props.data.mapPoints || [];
    mapPoints.value.forEach((point) => {
        point.periods = point.periods.map((period) => {
            const years = parseYearRange(period.yearRange);
            return {
                ...period,
                ...years,
            };
        });
    });

    sortedPoints.value = [...mapPoints.value].sort(
        (a, b) => a.year - b.year
    );

    if (sortedPoints.value.length > 0) {
        minYear.value = sortedPoints.value.reduce(
            (min, point) => Math.min(min, point.year),
            Infinity
        );
        maxYear.value = new Date().getFullYear();
        currentYear.value = minYear.value;
    }
});

const debouncedCurrentYear = ref(currentYear.value);

watch(
    () => currentYear.value,
    debounce((newYear) => {
        debouncedCurrentYear.value = newYear;
    }, 100)
);

const mapStyle = computed(() => ({
    transform: `translate(${mapPosition.value.x}px, ${mapPosition.value.y}px) scale(${zoomScale.value})`,
    transition: isPanning.value ? 'none' : 'transform 0.3s ease',
    transformOrigin: 'top left',
}));

const formattedCurrentYear = computed(() =>
    formatYearTooltip(debouncedCurrentYear.value)
);

function handleWheel(event) {
    event.preventDefault();
    const zoomFactor = event.deltaY < 0 ? zoomStep : -zoomStep;
    zoom(zoomFactor, event);
}

function zoomIn() {
    zoom(zoomStep);
}

function zoomOut() {
    zoom(-zoomStep);
}

function zoom(step, event = null) {
    const newZoomScale = Math.min(
        Math.max(zoomScale.value + step, minZoom),
        maxZoom
    );

    if (event) {
        const rect = mapContainer.value.getBoundingClientRect();
        const offsetX = event.clientX - rect.left;
        const offsetY = event.clientY - rect.top;
        const dx =
            (offsetX - mapPosition.value.x) *
            (newZoomScale / zoomScale.value - 1);
        const dy =
            (offsetY - mapPosition.value.y) *
            (newZoomScale / zoomScale.value - 1);

        mapPosition.value.x -= dx;
        mapPosition.value.y -= dy;
    } else {
        const rect = mapContainer.value.getBoundingClientRect();
        const centerX = rect.width / 2;
        const centerY = rect.height / 2;
        const dx =
            (centerX - mapPosition.value.x) *
            (newZoomScale / zoomScale.value - 1);
        const dy =
            (centerY - mapPosition.value.y) *
            (newZoomScale / zoomScale.value - 1);

        mapPosition.value.x -= dx;
        mapPosition.value.y -= dy;
    }

    zoomScale.value = newZoomScale;
}

function startPan(event) {
    isPanning.value = true;
    startPanPosition.value = { x: event.clientX, y: event.clientY };
}

function panMap(event) {
    if (!isPanning.value) return;
    const deltaX = event.clientX - startPanPosition.value.x;
    const deltaY = event.clientY - startPanPosition.value.y;
    mapPosition.value.x += deltaX;
    mapPosition.value.y += deltaY;
    startPanPosition.value = { x: event.clientX, y: event.clientY };
}

function startPanTouch(event) {
    isPanning.value = true;
    startPanPosition.value = {
        x: event.touches[0].clientX,
        y: event.touches[0].clientY,
    };
}

function panMapTouch(event) {
    if (!isPanning.value) return;
    const deltaX = event.touches[0].clientX - startPanPosition.value.x;
    const deltaY = event.touches[0].clientY - startPanPosition.value.y;
    mapPosition.value.x += deltaX;
    mapPosition.value.y += deltaY;
    startPanPosition.value = {
        x: event.touches[0].clientX,
        y: event.touches[0].clientY,
    };
}

function endPan() {
    isPanning.value = false;
}

function selectPoint(point) {
    currentPeriodIndex.value = 0;
    if (modalVisible.value) {
        pendingPoint.value = point;
        modalRef.value.close();
    } else {
        selectedPoint.value = point;
        modalVisible.value = true;
    }
}

function closeModal() {
    if (pendingPoint.value) {
        selectedPoint.value = pendingPoint.value;
        pendingPoint.value = null;
        nextTick(() => {
            modalRef.value.open();
        });
    } else {
        modalVisible.value = false;
        selectedPoint.value = null;
    }
}

const currentPeriod = computed(() => {
    if (selectedPoint.value && selectedPoint.value.periods) {
        return selectedPoint.value.periods[currentPeriodIndex.value];
    } else {
        return null;
    }
});

function onPeriodSelected(period) {
    currentYear.value = Math.floor(
        (period.startYear + period.endYear) / 2
    );
}

watch(currentYear, () => {
    if (selectedPoint.value && selectedPoint.value.periods) {
        const index = selectedPoint.value.periods.findIndex((period) => {
            return (
                currentYear.value >= period.startYear &&
                currentYear.value <= period.endYear
            );
        });
        if (index !== -1 && index !== currentPeriodIndex.value) {
            currentPeriodIndex.value = index;
        }
    }
});

watch(debouncedCurrentYear, () => {
    gsap.fromTo(
        '.map-widget__current-year',
        { opacity: 0, y: -10 },
        { opacity: 1, y: 0, duration: 0.3 }
    );
});

function selectBoundary(feature) {
    const point = mapPoints.value.find(
        (p) => p.periods.some(period => period.id === feature.properties.id)
    );
    if (point) {
        selectPoint(point);
        // Устанавливаем текущий период в соответствии с выбранной границей
        const periodIndex = point.periods.findIndex(period => period.id === feature.properties.id);
        if (periodIndex !== -1) {
            currentPeriodIndex.value = periodIndex;
        }
    }
}

const currentPeriodId = computed(() => {
    if (selectedPoint.value && selectedPoint.value.periods) {
        return selectedPoint.value.periods[currentPeriodIndex.value].id;
    }
    return null;
});

const mapWidget = ref(null);

function toggleFullScreen() {
    if (!isFullScreen.value) {
        enterFullScreen();
    } else {
        exitFullScreen();
    }
}

function enterFullScreen() {
    const element = mapWidget.value;
    if (element.requestFullscreen) {
        element.requestFullscreen();
    } else if (element.webkitRequestFullscreen) { /* Safari */
        element.webkitRequestFullscreen();
    } else if (element.msRequestFullscreen) { /* IE11 */
        element.msRequestFullscreen();
    }
}

function exitFullScreen() {
    if (document.exitFullscreen) {
        document.exitFullscreen();
    } else if (document.webkitExitFullscreen) { /* Safari */
        document.webkitExitFullscreen();
    } else if (document.msExitFullscreen) { /* IE11 */
        document.msExitFullscreen();
    }
}

function onFullScreenChange() {
    isFullScreen.value = !!document.fullscreenElement;
    if (isFullScreen.value) {
        centerMapInFullscreen();
    } else {
        resetMapPosition();
    }
}

function centerMapInFullscreen() {
    nextTick(() => {
        const containerWidth = mapContainer.value.clientWidth;
        const containerHeight = mapContainer.value.clientHeight;
        const contentWidth = mapContentWidth.value * zoomScale.value;
        const contentHeight = mapContainer.value.clientHeight * zoomScale.value;

        const initialMapPositionX = (containerWidth - contentWidth) / 2;
        const initialMapPositionY = (containerHeight - contentHeight) / 2;

        mapPosition.value.x = initialMapPositionX;
        mapPosition.value.y = initialMapPositionY;
    });
}

function resetMapPosition() {
    mapPosition.value.x = 0;
    mapPosition.value.y = 0;
}

const mapContentWidth = computed(() => {
    if (isFullScreen.value) {
        // Ширина карты в полноэкранном режиме
        const vh = window.innerHeight;
        return (vh * 16) / 9; // Соотношение сторон 16:9
    } else {
        return mapContainer.value.clientWidth;
    }
});

onMounted(() => {
    document.addEventListener('fullscreenchange', onFullScreenChange);
});

onBeforeUnmount(() => {
    document.removeEventListener('fullscreenchange', onFullScreenChange);
});
</script>

<style scoped>
.map-widget {
    position: relative;
    width: 100%;
    max-width: 800px;
    margin: 0 auto;
}

.map-widget--fullscreen {
    width: 100%;
    max-width: none;
    height: 100%;
    margin: 0;
    display: flex;
    flex-direction: column;
}

.map-widget__current-year {
    text-align: center;
    font-size: 24px;
    margin-bottom: 10px;
    color: #fff;
}

.map-widget__container {
    overflow: hidden;
    position: relative;
    cursor: grab;
    width: 100%;
    aspect-ratio: 16 / 9;
    height: 0;
    padding-bottom: 56.25%;
}

.map-widget--fullscreen .map-widget__container {
    flex: 1;
    width: 100%;
    height: 100%;
    padding-bottom: 0;
}

.map-widget__container:active {
    cursor: grabbing;
}

.map-widget__content {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    transform-origin: top left;
}

.map-widget--fullscreen .map-widget__content {
    width: calc(100vh * 16 / 9);
    height: 100%;
    position: absolute;
    left: 0%;
    transform: none;
    transform-origin: top left;
}

.map-widget__image {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    user-select: none;
    pointer-events: none;
}

.map-point {
    position: absolute;
    z-index: 3;
}

.map-widget__controls {
    margin-top: 10px;
}

.map-widget--fullscreen .map-widget__controls {
    order: 1;
}

.map-widget--fullscreen .map-widget__current-year {
    order: 2;
}

.map-widget--fullscreen .map-widget__container {
    order: 3;
}

.modal-title {
    font-size: 24px;
    margin: 0;
}

.modal-founded {
    font-size: 16px;
    margin: 0;
    color: #ccc;
}

.modal-period-title {
    font-size: 20px;
    margin-top: 10px;
    margin-bottom: 5px;
}

.modal-period-description {
    font-size: 16px;
    margin-bottom: 5px;
}

.modal-period-year {
    font-size: 14px;
    color: #aaa;
}

.modal-description {
    font-size: 16px;
    margin-bottom: 10px;
}

@media screen and (max-width: 1270px) {
    .modal-title {
        font-size: 18px;
    }

    .modal-founded {
        font-size: 14px;
    }

    .modal-period-title {
        font-size: 16px;
    }

    .modal-period-description {
        font-size: 14px;
        margin-bottom: 5px;
    }

    .modal-period-year {
        font-size: 12px;
        color: #aaa;
    }

    .modal-description {
        font-size: 14px;
        margin-bottom: 5px;
    }
}
</style>
