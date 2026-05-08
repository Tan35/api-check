<template>
    <div class="results-tabs">
        <button v-for="tab in resultTabs" :key="tab.id" class="tab-btn" :class="{ active: resultsStore.activeTab === tab.id }"
            @click="resultsStore.activeTab = tab.id" v-show="tab.visible">
            {{ tab.name }}
            <span class="counter">{{ resultsStore.results[tab.id].length }}</span>
        </button>
    </div>
</template>

<script setup>
import { computed } from 'vue';
import { useResultsStore } from '@/stores/results';
import { useConfigStore } from '@/stores/config';
import { RESULT_TAB_CONFIG } from '@/constants';

const resultsStore = useResultsStore();
const configStore = useConfigStore();

/**
 * @description 计算属性，根据当前提供商是否支持余额，动态显示结果标签页。
 * @returns {Array<object>} - 标签页配置数组。
 */
const resultTabs = computed(() => {
    const hasBalance = configStore.providers[configStore.currentProvider].hasBalance;
    return RESULT_TAB_CONFIG.map(tab => ({
        id: tab.id,
        name: tab.name,
        visible: hasBalance || !tab.balanceOnly,
    }));
});
</script>

<style scoped>
    /* 结果标签页容器 */
    .results-tabs {
        display: flex;
        overflow-x: auto;
        padding: 10px 10px 0;
        gap: 4px;
        background: var(--ds-white);
    }

    .tab-btn {
        height: 34px;
        padding: 0 10px;
        border-radius: var(--radius-md);
        background: transparent;
        cursor: pointer;
        font-size: 13px;
        font-weight: 500;
        font-family: var(--font-sans);
        color: var(--text-secondary);
        transition: background var(--transition-fast), color var(--transition-fast), box-shadow var(--transition-fast);
        flex-shrink: 0;
        display: flex;
        align-items: center;
        gap: 6px;
    }

    .tab-btn:hover {
        background: var(--ds-gray-50);
        color: var(--text-primary);
    }

    .tab-btn.active {
        background: var(--ds-gray-1000);
        color: var(--ds-white);
        box-shadow: none;
    }

    .tab-btn .counter {
        min-width: 20px;
        height: 20px;
        display: inline-flex;
        align-items: center;
        justify-content: center;
        background: var(--ds-gray-100);
        color: var(--text-secondary);
        padding: 0 6px;
        border-radius: var(--radius-badge);
        font-size: 12px;
        font-variant-numeric: tabular-nums;
        transition: all var(--transition-fast);
    }

    .tab-btn.active .counter {
        background-color: rgba(255, 255, 255, 0.16);
        color: var(--ds-white);
    }
</style>
