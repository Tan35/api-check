<template>
    <div class="provider-config-area" v-if="currentConfig">
        <div class="config-grid">
            <div class="config-item">
                <label :for="configStore.currentProvider + '__base'">{{ t('labelBaseUrl') }}</label>
                <input type="text" :id="configStore.currentProvider + '__base'" v-model="currentConfig.baseUrl"
                    :placeholder="t('placeholderBaseUrl')" :disabled="checkerStore.isChecking">
            </div>
            <div class="config-item">
                <label :for="configStore.currentProvider + '__model'">{{ t('labelModel') }}</label>
                <div class="input-with-button">
                    <input type="text" :id="configStore.currentProvider + '__model'" v-model="currentConfig.model"
                        :placeholder="t('placeholderModel')" :disabled="checkerStore.isChecking">
                    <button type="button" class="fetch-models-btn" @click="handleFetchModels"
                        :disabled="uiStore.isFetchingModels || checkerStore.isChecking">
                        <span v-if="!uiStore.isFetchingModels">{{ t('btnFetch') }}</span>
                        <span v-else class="loader"></span>
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { computed } from 'vue';
import { useConfigStore } from '@/stores/config';
import { useUiStore } from '@/stores/ui';
import { useCheckerStore } from '@/stores/checker';
import { fetchModels } from '@/api';
import { parseKeys } from '@/utils/keyParser';
import { t } from '@/i18n';

const configStore = useConfigStore();
const uiStore = useUiStore();
const checkerStore = useCheckerStore();

/**
 * @description 计算属性，获取当前选中提供商的配置。
 */
const currentConfig = computed(() => {
    return configStore.getCurrentProviderConfig();
});

/**
 * @description 处理获取模型列表的逻辑。
 * 根据输入 Key 的数量，决定是直接显示错误还是遍历尝试。
 * 最多尝试 5 个 Key，避免等待时间过长。
 */
const handleFetchModels = async () => {
    uiStore.isFetchingModels = true;
    try {
        const keys = parseKeys(configStore.tokensInput);
        if (keys.length === 0) {
            uiStore.showToast(t('toastNoKeyInput'), "warning");
            return;
        }

        const providerConfig = {
            currentProvider: configStore.currentProvider,
            baseUrl: currentConfig.value.baseUrl,
            currentRegion: configStore.currentRegion,
        };

        if (keys.length === 1) {
            try {
                const models = await fetchModels(keys[0], providerConfig);
                if (models && models.length > 0) {
                    models.sort((a, b) => a.localeCompare(b));
                    uiStore.openModal('modelSelector', { models, currentModel: currentConfig.value.model });
                } else {
                    uiStore.showToast(t('toastNoModels'), "warning");
                }
            } catch (error) {
                uiStore.showToast(t('toastFetchModelFailed', { msg: error.message }), "error");
            }
        } else {
            const maxAttempts = Math.min(keys.length, 5);
            for (let i = 0; i < maxAttempts; i++) {
                const key = keys[i];
                try {
                    const models = await fetchModels(key, providerConfig);
                    if (models && models.length > 0) {
                        models.sort((a, b) => a.localeCompare(b));
                        uiStore.openModal('modelSelector', { models, currentModel: currentConfig.value.model });
                        return;
                    }
                } catch (error) {
                    console.log(`Key ${key.substring(0, 10)}... failed, trying next.`);
                }
            }
            const attemptedMsg = maxAttempts < keys.length
                ? t('toastAllKeysFailedPrefix', { n: maxAttempts })
                : t('toastAllKeysFailedAll');
            uiStore.showToast(t('toastAllKeysFailed', { attempted: attemptedMsg }), "error");
        }
    } finally {
        uiStore.isFetchingModels = false;
    }
};
</script>

<style scoped>
    .provider-config-area {
        margin-top: 14px;
        padding-top: 14px;
        box-shadow: inset 0 1px 0 0 var(--border-color);
    }

    .config-grid {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 12px;
    }

    @media (max-width: 768px) {
        .config-grid {
            grid-template-columns: 1fr;
        }
    }

    .config-item {
        display: flex;
        flex-direction: column;
        gap: 7px;
    }

    .config-item label {
        font-size: 0.9rem;
        margin-bottom: 0;
    }

    .input-with-button {
        display: flex;
        gap: 8px;
    }

    .input-with-button input {
        flex-grow: 1;
    }

    .fetch-models-btn {
        padding: 0 14px;
        color: var(--text-secondary);
        border-radius: var(--radius-md);
        font-size: 13px;
        font-weight: 500;
        font-family: var(--font-sans);
        cursor: pointer;
        transition: background var(--transition-fast), color var(--transition-fast), box-shadow var(--transition-fast);
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 6px;
        min-width: 56px;
        height: var(--ctrl-height-md);
        background: var(--bg-input);
        box-shadow: var(--shadow-light-ring);
        flex-shrink: 0;
    }

    .fetch-models-btn:hover:not(:disabled) {
        background: var(--bg-secondary);
        color: var(--text-primary);
    }

    .fetch-models-btn:focus-visible {
        outline: none;
        box-shadow: var(--shadow-light-ring);
    }

    input:disabled {
        background-color: var(--bg-secondary);
        cursor: not-allowed;
    }

    .fetch-models-btn:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
</style>
