<template>
    <div class="results-tabs" role="tablist" aria-label="结果分类">
        <button
            v-for="tab in resultTabs"
            :key="tab.id"
            class="tab-btn"
            :class="{ active: resultsStore.activeTab === tab.id }"
            @click="resultsStore.activeTab = tab.id"
            v-show="tab.visible"
            role="tab"
            :aria-selected="resultsStore.activeTab === tab.id"
        >
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
const configStore  = useConfigStore();

const resultTabs = computed(() => {
    const hasBalance = configStore.providers[configStore.currentProvider].hasBalance;
    return RESULT_TAB_CONFIG.map(tab => ({
        id:      tab.id,
        name:    tab.name,
        visible: hasBalance || !tab.balanceOnly,
    }));
});
</script>

<style scoped>
/* ── Tabs container ── */
.results-tabs {
    display: flex;
    align-items: center;
    gap: 3px;
    padding: 10px 12px 8px;
    background: var(--ds-white);
    overflow-x: auto;
    flex-shrink: 0;
    box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
}

/* ── Tab button — capsule shape, matches topbar segmented control ── */
.tab-btn {
    height: 30px;
    padding: 0 10px;
    border-radius: var(--radius-tabs);
    background: transparent;
    border: none;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    font-family: var(--font-sans);
    color: var(--text-secondary);
    transition: background var(--transition-fast), color var(--transition-fast);
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    white-space: nowrap;
}
.tab-btn:hover { background: var(--ds-gray-100); color: var(--text-primary); }
.tab-btn.active { background: var(--ds-gray-1000); color: var(--ds-white); }

.counter {
    min-width: 18px;
    height: 18px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: rgba(0, 0, 0, 0.08);
    color: inherit;
    padding: 0 5px;
    border-radius: 9999px;
    font-size: 11px;
    font-family: var(--font-mono);
    font-variant-numeric: tabular-nums;
    transition: background var(--transition-fast);
}
.tab-btn.active .counter { background: rgba(255, 255, 255, 0.18); }
</style>
