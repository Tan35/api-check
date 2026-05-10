<template>
    <div class="model-selector-content">
        <div class="model-selector-header">
            <h3 id="modelSelectorTitle">{{ t('labelModel') }}</h3>
            <button class="model-selector-close" @click="uiStore.closeModal()" :aria-label="t('btnClose')">
                <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75">
                    <path d="M3 3l10 10M13 3L3 13"/>
                </svg>
            </button>
        </div>
        <div class="model-selector-search">
            <input type="search" v-model="uiStore.modelSearch" :placeholder="t('placeholderSearchModelSelector')">
        </div>
        <div class="model-selector-body">
            <ul class="model-list">
                <li v-for="model in filteredModels" :key="model" @click="selectModel(model)">{{ model }}</li>
            </ul>
        </div>
        <div class="model-selector-footer">
            <span id="modelCount">{{ t('modelCount', { visible: filteredModels.length, total: uiStore.modalData.models?.length || 0 }) }}</span>
            <button class="copy-btn" @click="copyAllModels">{{ t('btnCopyAll') }}</button>
        </div>
    </div>
</template>

<script setup>
import { computed } from 'vue';
import { useUiStore } from '@/stores/ui';
import { useConfigStore } from '@/stores/config';
import { t } from '@/i18n';

const uiStore = useUiStore();
const configStore = useConfigStore();

const filteredModels = computed(() => {
    if (!uiStore.modalData.models) return [];
    const searchTerm = uiStore.modelSearch.toLowerCase();
    return uiStore.modalData.models.filter(m => m.toLowerCase().includes(searchTerm));
});

const selectModel = (model) => {
    configStore.getCurrentProviderConfig().model = model;
    uiStore.showToast(t('toastModelSelected', { model }), 'info', 2000);
    uiStore.closeModal();
};

const copyAllModels = () => {
    navigator.clipboard.writeText(filteredModels.value.join('\n')).then(() => {
        uiStore.showToast(t('toastModelsCopied', { count: filteredModels.value.length }), 'success');
    });
};
</script>

<style scoped>
    .model-selector-content {
        max-width: 500px;
    }

    .model-selector-search {
        padding: 8px 16px;
        box-shadow: inset 0 -1px 0 0 var(--border-color);
    }

    .model-selector-search input {
        height: 40px;
    }

    .copy-btn {
        padding: 0 16px;
        background: var(--bg-surface);
        color: var(--text-primary);
        box-shadow: var(--shadow-light-ring);
        border-radius: var(--radius-sm);
        font-size: 13px;
        font-weight: 500;
        font-family: var(--font-sans);
        cursor: pointer;
        transition: all 0.2s ease;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 6px;
        white-space: nowrap;
        height: 32px;
    }

    .copy-btn:hover {
        background: var(--bg-secondary);
    }
</style>
