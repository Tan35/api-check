<template>
    <div class="model-selector-content">
        <div class="model-selector-header">
            <h3 id="regionSelectorTitle">{{ t('settingsTitle') }}</h3>
            <button class="model-selector-close" @click="uiStore.closeModal()" :aria-label="t('btnClose')">
                <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75">
                    <path d="M3 3l10 10M13 3L3 13"/>
                </svg>
            </button>
        </div>
        <div class="model-selector-body">
            <div class="settings-section">
                <h4 class="settings-title">{{ t('settingsSectionRegion') }}</h4>
                <select class="region-select" :value="configStore.currentRegion"
                    @change="configStore.selectRegion($event.target.value)">
                    <option v-for="(label, key) in configStore.regions" :key="key" :value="key">
                        {{ t('region' + key.charAt(0).toUpperCase() + key.slice(1)) || label }}
                    </option>
                </select>
            </div>
            <div class="settings-section">
                <h4 class="settings-title">{{ t('settingsSectionParams') }}</h4>
                <div class="advanced-settings-grid">
                    <div class="config-item">
                        <label for="threshold">{{ t('labelThreshold') }}</label>
                        <input id="threshold" type="number" v-model.number="configStore.threshold" min="0" step="0.1">
                    </div>
                    <div class="config-item">
                        <div class="label-with-hint">
                            <label for="concurrency">{{ t('labelConcurrency') }}</label>
                            <span class="config-hint">{{ t('hintConcurrency') }}</span>
                        </div>
                        <input id="concurrency" type="number" :value="configStore.concurrency"
                            @input="handleConcurrencyInput" min="1" max="20">
                    </div>

                    <div class="config-item">
                        <label for="max-tokens">{{ t('labelMaxTokens') }}</label>
                        <input id="max-tokens" type="number" v-model.number="configStore.validationMaxTokens" min="1">
                    </div>
                    
                    <div class="config-item">
                        <label for="max-output-tokens">{{ t('labelMaxOutputTokens') }}</label>
                        <input id="max-output-tokens" type="number" v-model.number="configStore.validationMaxOutputTokens" min="1">
                    </div>

                    <div class="config-item prompt-item">
                        <label for="prompt">{{ t('labelPrompt') }}</label>
                        <input id="prompt" type="text" v-model="configStore.validationPrompt">
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { onBeforeUnmount } from 'vue';
import { useUiStore } from '@/stores/ui';
import { useConfigStore } from '@/stores/config';
import { t } from '@/i18n';
const uiStore = useUiStore();
const configStore = useConfigStore();

/**
 * @description 存储 onblur 事件处理器引用，用于组件卸载时清理。
 */
let currentBlurHandler = null;
let currentInputElement = null;

/**
 * @description 处理并发请求数的输入，确保其在有效范围内（1-20）。
 * @param {Event} event - 输入事件对象。
 */
const handleConcurrencyInput = (event) => {
    let value = parseInt(event.target.value, 10);

    // 如果输入不是有效数字，则不进行处理
    if (isNaN(value)) {
        return;
    }

    // 清理之前的 blur 事件监听器
    if (currentInputElement && currentBlurHandler) {
        currentInputElement.removeEventListener('blur', currentBlurHandler);
    }

    // 创建新的 blur 事件处理器
    currentBlurHandler = () => {
        if (event.target.value === '' || parseInt(event.target.value, 10) < 1) {
            configStore.concurrency = 1;
        }
    };
    currentInputElement = event.target;
    event.target.addEventListener('blur', currentBlurHandler, { once: true });

    // 限制范围 1-20
    if (value < 1) {
        value = 1;
    } else if (value > 20) {
        value = 20;
    }

    configStore.concurrency = value;
};

/**
 * @description 组件卸载前清理事件监听器。
 */
onBeforeUnmount(() => {
    if (currentInputElement && currentBlurHandler) {
        currentInputElement.removeEventListener('blur', currentBlurHandler);
    }
});
</script>

<style scoped>
    /* 设置部分 */
    .settings-section {
        margin-bottom: 20px;
    }

    .settings-section:last-child {
        margin-bottom: 8px;
    }

    /* 设置标题 */
    .settings-title {
        font-size: 1rem;
        font-weight: 600;
        font-family: var(--font-sans);
        color: var(--text-primary);
        margin-bottom: 12px;
        padding-bottom: 8px;
        box-shadow: inset 0 -1px 0 0 var(--border-color);
    }

    /* 区域下拉 */
    .region-select {
        width: 100%;
        height: var(--ctrl-height-md);
        padding: 0 12px;
        border: none;
        border-radius: var(--radius-md);
        background: var(--bg-input);
        color: var(--text-primary);
        box-shadow: var(--shadow-ring);
        font-family: var(--font-sans);
        font-size: 13px;
        cursor: pointer;
        appearance: none;
        -webkit-appearance: none;
        background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12' fill='none'%3E%3Cpath d='M2.5 4.5L6 8L9.5 4.5' stroke='%23888' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
        background-repeat: no-repeat;
        background-position: right 12px center;
        padding-right: 32px;
    }

    .region-select:focus {
        outline: none;
        box-shadow: var(--shadow-ring);
    }

    /* 高级设置网格布局 */
    .advanced-settings-grid {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 12px;
        padding: 0;
    }

    .prompt-item {
        grid-column: 1 / -1;
    }

    .config-item {
        display: flex;
        flex-direction: column;
        gap: 8px;
    }

    /* 带有提示的标签 */
    .label-with-hint {
        display: flex;
        justify-content: space-between;
        align-items: center;
    }

    .config-item label {
        font-size: 12px;
        margin-bottom: 0;
        font-weight: 500;
        color: var(--text-secondary);
    }

    .config-hint {
        font-size: 12px;
        color: var(--text-tertiary);
    }

    .config-item input {
        height: 40px;
    }

    /* 媒体查询：小屏幕设备 */
    @media (max-width: 768px) {
        .advanced-settings-grid {
            grid-template-columns: 1fr;
        }
    }

</style>
