<template>
    <div class="provider-header">
        <label for="providerSelect">Provider</label>
        <div class="header-actions">
            <label class="switch-label" title="启用流式检测 (Stream Mode)">
                <span class="switch-title">Stream</span>
                <input type="checkbox" v-model="currentConfig.enableStream" :disabled="checkerStore.isChecking">
                <span class="slider"></span>
            </label>
            <button @click="uiStore.openModal('regionSelector')" class="settings-btn" title="检测设置"
                :disabled="checkerStore.isChecking">
                <svg viewBox="0 0 24 24" aria-hidden="true">
                    <path d="M12 15.5a3.5 3.5 0 1 0 0-7 3.5 3.5 0 0 0 0 7Z" />
                    <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1-2.83 2.83l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-4 0v-.09a1.65 1.65 0 0 0-1.08-1.51 1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1 0-4h.09a1.65 1.65 0 0 0 1.51-1.08 1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 2.83-2.83l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 4 0v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9c.16.66.76 1.12 1.44 1.12H21a2 2 0 0 1 0 4h-.09c-.68 0-1.28.46-1.51 1Z" />
                </svg>
            </button>
        </div>
    </div>

    <div class="custom-provider-select" :class="{ disabled: checkerStore.isChecking }" id="providerSelectWrapper"
        ref="providerSelectWrapper">
        <!-- 沉浸式触发器：打开时变为 input，关闭时显示选中值 -->
        <div class="custom-provider-trigger" :class="{ open: uiStore.providerDropdownOpen }"
            @click="!checkerStore.isChecking && toggleProviderDropdown()"
            role="combobox"
            aria-haspopup="listbox"
            :aria-expanded="uiStore.providerDropdownOpen"
            aria-labelledby="providerSelect providerDisplay">
            <input
                v-if="uiStore.providerDropdownOpen"
                ref="searchInputElement"
                class="provider-inline-search"
                type="text"
                v-model="providerSearchTerm"
                :placeholder="configStore.providers[configStore.currentProvider].label"
                @click.stop
                aria-label="搜索提供商"
            />
            <span v-else id="providerDisplay">{{ configStore.providers[configStore.currentProvider].label }}</span>
            <span class="trigger-chevron"></span>
        </div>
        <div
            class="custom-provider-dropdown t-dropdown"
            data-origin="top-left"
            :class="{ 'is-open': uiStore.providerDropdownOpen, 'is-closing': dropdownClosing }"
            ref="dropdownContainer"
            role="listbox"
            aria-label="API 提供商列表">
            <div v-for="(provider, key) in filteredProviders" :key="key" class="provider-option"
                :class="{ selected: key === configStore.currentProvider, highlighted: providerKeys[highlightedIndex] === key }" @mousedown.prevent="handleProviderSelect(key)"
                role="option"
                :aria-selected="key === configStore.currentProvider">
                <span>{{ provider.label }}</span>
            </div>
        </div>
    </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, computed, watch, nextTick } from 'vue';
import { useConfigStore } from '@/stores/config';
import { useUiStore } from '@/stores/ui';
import { useResultsStore } from '@/stores/results';
import { useCheckerStore } from '@/stores/checker';

const configStore = useConfigStore();
const uiStore = useUiStore();
const resultsStore = useResultsStore();
const checkerStore = useCheckerStore();

const providerSelectWrapper = ref(null);
const providerSearchTerm = ref('');
const searchInputElement = ref(null);
const dropdownContainer = ref(null);
const highlightedIndex = ref(-1);

const providerKeys = computed(() => Object.keys(filteredProviders.value));

const currentConfig = computed(() => {
    return configStore.getCurrentProviderConfig();
});

const filteredProviders = computed(() => {
    const searchTerm = providerSearchTerm.value.toLowerCase();
    const providers = configStore.providers;
    if (!searchTerm) {
        return providers;
    }
    const filtered = {};
    for (const key in providers) {
        const provider = providers[key];
        if (provider.label.toLowerCase().includes(searchTerm)) {
            filtered[key] = provider;
        }
    }
    const keys = Object.keys(filtered);
    if (highlightedIndex.value >= keys.length) highlightedIndex.value = keys.length - 1;
    return filtered;
});

const handleProviderSelect = (key) => {
    configStore.selectProvider(key);
    resultsStore.clearResults();
    if (resultsStore.activeTab !== 'valid') {
        resultsStore.activeTab = 'valid';
    }
    providerSearchTerm.value = '';
    closeProviderDropdown();
};

const closeDropdown = (e) => {
    if (providerSelectWrapper.value && !providerSelectWrapper.value.contains(e.target)) {
        closeProviderDropdown();
        providerSearchTerm.value = '';
    }
};

const handleKeyDown = (e) => {
    if (!uiStore.providerDropdownOpen) return;

    if (e.key === 'Escape') {
        closeProviderDropdown();
        return;
    }

    const keys = providerKeys.value;
    if (!keys.length) return;

    if (e.key === 'ArrowDown') {
        e.preventDefault();
        highlightedIndex.value = (highlightedIndex.value + 1) % keys.length;
        scrollHighlightedIntoView();
    } else if (e.key === 'ArrowUp') {
        e.preventDefault();
        highlightedIndex.value = (highlightedIndex.value - 1 + keys.length) % keys.length;
        scrollHighlightedIntoView();
    } else if (e.key === 'Enter' || e.key === 'Return') {
        e.preventDefault();
        const key = keys[highlightedIndex.value];
        if (key) {
            handleProviderSelect(key);
        }
    }
};

const dropdownClosing = ref(false);
let dropdownCloseTimer = null;

const getDropdownCloseMs = () => {
    try {
        const raw = getComputedStyle(document.documentElement)
            .getPropertyValue('--dropdown-close-dur')
            .trim();
        if (!raw) return 150;
        if (raw.endsWith('ms')) return parseFloat(raw) || 150;
        if (raw.endsWith('s')) return (parseFloat(raw) || 0.15) * 1000;
        return parseFloat(raw) || 150;
    } catch {
        return 150;
    }
};

const toggleProviderDropdown = () => {
    if (uiStore.providerDropdownOpen) {
        closeProviderDropdown();
    } else {
        openProviderDropdown();
    }
};

const openProviderDropdown = () => {
    if (dropdownCloseTimer) {
        clearTimeout(dropdownCloseTimer);
        dropdownCloseTimer = null;
    }
    dropdownClosing.value = false;
    providerSearchTerm.value = '';
    uiStore.providerDropdownOpen = true;
    nextTick(() => searchInputElement.value?.focus());
};

const closeProviderDropdown = () => {
    if (!uiStore.providerDropdownOpen && !dropdownClosing.value) return;
    uiStore.providerDropdownOpen = false;
    dropdownClosing.value = true;
    if (dropdownCloseTimer) {
        clearTimeout(dropdownCloseTimer);
    }
    const closeMs = getDropdownCloseMs();
    dropdownCloseTimer = setTimeout(() => {
        dropdownClosing.value = false;
        dropdownCloseTimer = null;
    }, closeMs);
};

const scrollHighlightedIntoView = () => {
    nextTick(() => {
        const container = dropdownContainer.value;
        if (!container) return;
        const options = container.querySelectorAll('.provider-option');
        const idx = highlightedIndex.value;
        if (idx < 0 || idx >= options.length) return;
        const el = options[idx];
        const elTop = el.offsetTop;
        const elBottom = elTop + el.offsetHeight;
        const viewTop = container.scrollTop;
        const viewBottom = viewTop + container.clientHeight;
        if (elTop < viewTop) {
            container.scrollTop = elTop;
        } else if (elBottom > viewBottom) {
            container.scrollTop = elBottom - container.clientHeight;
        }
    });
};

onMounted(() => {
    document.addEventListener('click', closeDropdown);
    document.addEventListener('keydown', handleKeyDown);
});

onBeforeUnmount(() => {
    document.removeEventListener('click', closeDropdown);
    document.removeEventListener('keydown', handleKeyDown);
    if (dropdownCloseTimer) {
        clearTimeout(dropdownCloseTimer);
        dropdownCloseTimer = null;
    }
});

watch(() => uiStore.providerDropdownOpen, (isOpen) => {
    if (isOpen) {
        highlightedIndex.value = -1;
    }
});
</script>

<style scoped>
    .provider-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 12px;
        margin-bottom: 10px;
    }

    .provider-header label {
        margin-bottom: 0;
    }

    .header-actions {
        display: flex;
        align-items: center;
        gap: 8px;
    }

    .settings-btn {
        width: 32px;
        height: 32px;
        background: transparent;
        cursor: pointer;
        border-radius: var(--radius-md);
        color: var(--text-tertiary);
        display: flex;
        align-items: center;
        justify-content: center;
        box-shadow: none;
        transition: color var(--transition-fast), background var(--transition-fast), box-shadow var(--transition-fast);
    }

    .settings-btn svg {
        width: 18px;
        height: 18px;
        fill: none;
        stroke: currentColor;
        stroke-width: 1.8;
        stroke-linecap: round;
        stroke-linejoin: round;
    }

    .settings-btn:hover {
        background: var(--bg-secondary);
        color: var(--text-primary);
    }

    .settings-btn:focus-visible {
        outline: none;
        box-shadow: var(--shadow-light-ring);
    }

    .settings-btn:disabled {
        opacity: 0.3;
        cursor: not-allowed;
    }

    /* Custom select */
    .custom-provider-select {
        position: relative;
        width: 100%;
    }

    .custom-provider-trigger {
        width: 100%;
        height: 40px;
        padding: 0 32px 0 12px;
        background: var(--bg-surface);
        box-shadow: var(--shadow-ring);
        border-radius: var(--radius-md);
        font-size: 14px;
        font-family: var(--font-sans);
        cursor: pointer;
        display: flex;
        align-items: center;
        transition: box-shadow var(--transition-fast);
    }

    .custom-provider-trigger:hover {
        background: var(--bg-secondary);
    }

    .custom-provider-trigger.open {
        box-shadow: var(--shadow-ring);
        background: var(--bg-surface);
        cursor: text;
    }

    /* 沉浸式搜索输入框——完全清除浏览器默认样式，融入触发器 */
    .provider-inline-search {
        flex: 1;
        width: 0;
        min-width: 0;
        height: 100%;
        border: none;
        outline: none;
        box-shadow: none;
        -webkit-appearance: none;
        appearance: none;
        background: transparent;
        font-size: 14px;
        font-family: var(--font-sans);
        color: var(--text-primary);
        padding: 0;
        margin: 0;
    }

    .provider-inline-search:focus {
        box-shadow: none !important;
        outline: none;
    }

    .provider-inline-search::placeholder {
        color: var(--ds-gray-400);
    }

    #providerDisplay {
        flex: 1;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        pointer-events: none;  /* let clicks fall through to trigger */
        user-select: none;
    }

    .trigger-chevron {
        position: absolute;
        right: 12px;
        top: 50%;
        width: 6px;
        height: 6px;
        border-right: 1.5px solid var(--ds-gray-400);
        border-bottom: 1.5px solid var(--ds-gray-400);
        transform: translateY(-65%) rotate(45deg);
        transition: transform var(--transition-fast);
        pointer-events: none;
    }

    .custom-provider-trigger.open .trigger-chevron {
        transform: translateY(-35%) rotate(-135deg);
    }

    .custom-provider-dropdown {
        position: absolute;
        top: calc(100% + 4px);
        left: 0;
        right: 0;
        background: var(--bg-surface);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-full-card);
        z-index: 100;
        max-height: 300px;
        overflow-y: auto;
        padding: 4px 8px;
    }

    .provider-option {
        min-height: 34px;
        padding: 0 10px;
        cursor: pointer;
        transition: background var(--transition-fast), color var(--transition-fast);
        display: flex;
        align-items: center;
        font-size: 14px;
        font-family: var(--font-sans);
        border-radius: var(--radius-md);
        color: var(--text-secondary);
    }

    .provider-option:hover {
        background: var(--bg-secondary);
        color: var(--text-primary);
    }

    .provider-option.selected {
        background: var(--ds-accent-soft);
        color: var(--text-primary);
        font-weight: 500;
    }

    .provider-option.highlighted {
        background: var(--bg-secondary);
    }

    .custom-provider-select.disabled {
        opacity: 0.5;
        pointer-events: none;
    }

    /* Toggle switch */
    .switch-label {
        display: inline-flex;
        align-items: center;
        cursor: pointer;
        gap: 8px;
        user-select: none;
        height: 32px;
        padding: 0 10px;
        border-radius: var(--radius-md);
        background: var(--bg-secondary);
        box-shadow: var(--shadow-light-ring);
        transition: background var(--transition-fast), box-shadow var(--transition-fast);
    }

    .switch-label:hover {
        background: var(--bg-tertiary);
    }

    .switch-label input[type="checkbox"] {
        display: none;
    }

    .slider {
        position: relative;
        width: 34px;
        height: 18px;
        background-color: var(--ds-gray-200);
        border-radius: var(--radius-badge);
        transition: background-color var(--transition-fast);
        flex-shrink: 0;
        box-shadow: inset 0 0 0 1px var(--border-color);
    }

    .slider::before {
        content: '';
        position: absolute;
        height: 14px;
        width: 14px;
        left: 2px;
        top: 2px;
        background-color: var(--bg-surface);
        border-radius: 50%;
        transition: transform var(--transition-fast);
        box-shadow: 0 1px 2px rgba(0,0,0,0.1);
    }

    .switch-label input:checked + .slider {
        background-color: var(--ds-gray-900);
        box-shadow: none;
    }

    .switch-label input:checked + .slider::before {
        transform: translateX(16px);
        box-shadow: none;
    }

    .switch-title {
        font-weight: 500;
        color: var(--text-secondary);
        font-size: 12px;
    }

    .switch-label:has(input:disabled) {
        cursor: not-allowed;
        opacity: 0.5;
    }

    @media (max-width: 480px) {
        .custom-provider-dropdown {
            max-height: 60vh;
        }
    }
</style>
