<template>
    <div class="modal-content">
        <div class="modal-header">
            <div class="modal-icon error">!</div>
            <h3 class="modal-title">{{ t('confirmTitle') }}</h3>
        </div>
        <div class="modal-message">{{ uiStore.modalData.message }}</div>
        <div class="modal-actions confirmation">
            <button class="modal-btn secondary" @click="uiStore.handleConfirmation(false)">{{ t('btnConfirmNo') }}</button>
            <button class="modal-btn primary" @click="uiStore.handleConfirmation(true)">{{ t('btnConfirmYes') }}</button>
        </div>
    </div>
</template>

<script setup>
import { useUiStore } from '@/stores/ui';
import { t } from '@/i18n';
const uiStore = useUiStore();
</script>

<style scoped>
    /* 确认模态框特定样式 */
    .modal-content {
        max-width: 500px;
    }

    .modal-message {
        text-align: center;
    }
</style>
