<template>
    <teleport to="body">
        <div v-if="uiStore.activeModal" class="custom-modal show" @click.self="uiStore.closeModal()">
            <component :is="activeModalComponent" />
        </div>
    </teleport>
</template>

<script setup>
import { computed, defineAsyncComponent } from 'vue';
import { useUiStore } from '@/stores/ui';

const uiStore = useUiStore();

// 异步导入模态框组件，以优化初始加载性能
const DetailsModal = defineAsyncComponent(() => import('./modals/DetailsModal.vue'));
const ModelSelectorModal = defineAsyncComponent(() => import('./modals/ModelSelectorModal.vue'));
const SettingsModal = defineAsyncComponent(() => import('./modals/SettingsModal.vue'));
const ConfirmationModal = defineAsyncComponent(() => import('./modals/ConfirmationModal.vue'));
const KeyDetailModal = defineAsyncComponent(() => import('./modals/KeyDetailModal.vue'));

/**
 * @description 计算属性，根据 uiStore.activeModal 的值动态选择要渲染的模态框组件。
 */
const activeModalComponent = computed(() => {
    switch (uiStore.activeModal) {
        case 'details':
            return DetailsModal;
        case 'modelSelector':
            return ModelSelectorModal;
        case 'regionSelector': // 注意：这里 'regionSelector' 实际对应的是 SettingsModal
            return SettingsModal;
        case 'confirmation':
            return ConfirmationModal;
        case 'keyDetail':
            return KeyDetailModal;
        default:
            return null;
    }
});
</script>

<style scoped>
    /* Vercel 遮罩层 — 浅色 hsla(0, 0%, 98%, 1) */
    .custom-modal {
        position: fixed;
        top: 0;
        left: 0;
        width: 100vw;
        height: 100vh;
        height: 100dvh;
        background: rgba(0, 0, 0, 0.4);
        -webkit-backdrop-filter: blur(4px);
        backdrop-filter: blur(4px);
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 10000;
        opacity: 0;
        visibility: hidden;
        transition: opacity 0.2s ease;
    }

    .custom-modal.show {
        opacity: 1;
        visibility: visible;
    }

    .custom-modal.show :deep(.modal-content),
    .custom-modal.show :deep(.model-selector-content) {
        transform: scale(1) translateY(0);
        max-height: 90vh;
        max-height: 90dvh;
    }

    /* 模态框内容 — Shadow-as-Border + multi-layer shadow */
    :deep(.modal-content),
    :deep(.model-selector-content) {
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-full-card);
        transform: scale(0.95) translateY(10px);
        transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
    }

    @media (max-width: 480px) {
        .custom-modal {
            -webkit-backdrop-filter: blur(2px);
            backdrop-filter: blur(2px);
        }
    }
</style>