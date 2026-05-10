<template>
    <div class="results-tabs" role="tablist" :aria-label="t('tabChecker')">
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
import { t, currentLang } from '@/i18n';

const resultsStore = useResultsStore();
const configStore  = useConfigStore();

/** 标签 ID 到 i18n key 的映射 */
const TAB_I18N_KEYS = {
    valid:       'tabValid',
    lowBalance:  'tabLowBalance',
    zeroBalance: 'tabZeroBalance',
    rateLimit:   'tabRateLimit',
    invalid:     'tabInvalid',
    duplicate:   'tabDuplicate',
};

const resultTabs = computed(() => {
    // 依赖 currentLang 以便语言切换时重新计算
    void currentLang.value;
    const hasBalance = configStore.providers[configStore.currentProvider].hasBalance;
    return RESULT_TAB_CONFIG.map(tab => ({
        id:      tab.id,
        name:    t(TAB_I18N_KEYS[tab.id] || tab.id),
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
    background: var(--bg-surface);
    overflow-x: auto;
    flex-shrink: 0;
    box-shadow: inset 0 -1px 0 0 var(--border-color);
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
.tab-btn:hover { background: var(--bg-tertiary); color: var(--text-primary); }
.tab-btn.active {
    background: var(--bg-secondary);
    color: var(--text-primary);
    font-weight: 600;
    box-shadow: var(--shadow-ring);
}

.counter {
    min-width: 18px;
    height: 18px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-tertiary);
    color: inherit;
    padding: 0 5px;
    border-radius: 9999px;
    font-size: 11px;
    font-family: var(--font-mono);
    font-variant-numeric: tabular-nums;
    transition: background var(--transition-fast);
}
.tab-btn.active .counter { background: var(--bg-tertiary); }
</style>
