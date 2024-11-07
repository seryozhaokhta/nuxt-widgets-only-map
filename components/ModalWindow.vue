<!-- components/ModalWindow.vue -->

<template>
    <div class="modal-content" ref="modalContent">
        <header class="modal-header">
            <slot name="header"></slot>
        </header>
        <div class="modal-body">
            <slot name="body"></slot>
        </div>
        <footer class="modal-footer">
            <button class="close-button" @click="close">Close</button>
        </footer>
    </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, defineExpose } from 'vue';
import gsap from 'gsap';

const emits = defineEmits(['close']);

const modalContent = ref(null);

function open() {
    gsap.fromTo(
        modalContent.value,
        { y: '100%', opacity: 0 },
        { y: '0%', opacity: 1, duration: 0.5, ease: 'power1.out' }
    );
}

function close() {
    gsap.to(modalContent.value, {
        y: '100%',
        opacity: 0,
        duration: 0.5,
        ease: 'power1.in',
        onComplete: () => {
            emits('close');
        },
    });
}

function handleKeydown(event) {
    if (event.key === 'Escape') {
        close();
    }
}

onMounted(() => {
    open();
    window.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
    window.removeEventListener('keydown', handleKeydown);
});

defineExpose({
    open,
    close,
});
</script>

<style scoped>
.modal-content {
    background: rgba(126, 126, 126, 0.5);
    backdrop-filter: blur(12px);
    color: #fff;
    padding: 20px;
    border-radius: 16px 16px 0 0;
    max-width: 500px;
    width: 100%;
    text-align: left;
    position: absolute;
    bottom: 0;
    left: 50%;
    transform: translate(-50%, 100%);
    box-sizing: border-box;
    box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.3);
}

.modal-header {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: 0;
    margin-bottom: 10px;
}

.modal-body,
.modal-footer {
    margin-bottom: 10px;
}

.modal-footer {
    margin-bottom: 0;
}

.close-button {
    background-color: black;
    color: #fff;
    padding: 10px 20px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.close-button:hover {
    background-color: darkgrey;
}

@media screen and (max-width: 600px) {
    .modal-content {
        max-height: 80vh;
        overflow-y: auto;
        padding: 15px;
    }

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

    .close-button {
        padding: 8px 16px;
    }
}
</style>
