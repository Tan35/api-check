<template>
    <div class="provider-header">
        <label for="providerSelect">API 提供商</label>
        <div class="header-actions">
            <label class="switch-label" title="启用流式检测 (Stream Mode)">
                <span class="switch-title">流式检测</span>
                <input type="checkbox" v-model="currentConfig.enableStream" :disabled="checkerStore.isChecking">
                <span class="slider"></span>
            </label>
            <button @click="uiStore.openModal('regionSelector')" class="settings-btn" title="检测设置"
                :disabled="checkerStore.isChecking">
                <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
                    <circle cx="8" cy="8" r="2.5"/>
                    <path d="M13.5 8a5.5 5.5 0 01-.3 1.8l1.2.9-1.5 2.6-1.4-.5a5.5 5.5 0 01-1.5 1l.2 1.5h-3l.2-1.5a5.5 5.5 0 01-1.5-1l-1.4.5L3.6 10.7l1.2-.9A5.5 5.5 0 014.5 8a5.5 5.5 0 01.3-1.8L3.6 5.3l1.5-2.6 1.4.5a5.5 5.5 0 011.5-1L7.8.7h3l-.2 1.5a5.5 5.5 0 011.5 1l1.4-.5 1.5 2.6-1.2.9c.2.6.3 1.2.3 1.8z"/>
                </svg>
            </button>
        </div>
    </div>

    <div class="custom-provider-select" :class="{ disabled: checkerStore.isChecking }" id="providerSelectWrapper"
        ref="providerSelectWrapper">
        <div class="custom-provider-trigger" :class="{ open: uiStore.providerDropdownOpen }"
            @click="!checkerStore.isChecking && (uiStore.providerDropdownOpen = !uiStore.providerDropdownOpen)"
            role="combobox"
            aria-haspopup="listbox"
            :aria-expanded="uiStore.providerDropdownOpen"
            aria-labelledby="providerSelect providerDisplay">
            <span id="providerDisplay">{{ configStore.providers[configStore.currentProvider].label }}</span>
            <span class="trigger-chevron"></span>
        </div>
        <div class="custom-provider-dropdown" :class="{ open: uiStore.providerDropdownOpen }" ref="dropdownContainer"
            role="listbox"
            aria-label="API 提供商列表">
            <input type="search" v-model="providerSearchTerm" placeholder="搜索提供商..." class="provider-search-input" ref="searchInputElement"
                aria-label="搜索提供商">
            <div v-for="(provider, key) in filteredProviders" :key="key" class="provider-option"
                :class="{ selected: key === configStore.currentProvider, highlighted: providerKeys[highlightedIndex] === key }" @click="handleProviderSelect(key)"
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
};

const closeDropdown = (e) => {
    if (providerSelectWrapper.value && !providerSelectWrapper.value.contains(e.target)) {
        uiStore.providerDropdownOpen = false;
        providerSearchTerm.value = '';
    }
};

const handleKeyDown = (e) => {
    if (!uiStore.providerDropdownOpen) return;

    if (e.key === 'Escape') {
        uiStore.providerDropdownOpen = false;
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
            uiStore.providerDropdownOpen = false;
        }
    }
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
});

watch(() => uiStore.providerDropdownOpen, (isOpen) => {
    if (isOpen) {
        highlightedIndex.value = -1;
        nextTick(() => {
            setTimeout(() => {
                searchInputElement.value?.focus();
            }, 100);
        });
    }
});
</script>

<style scoped>
    .provider-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 12px;
    }

    .provider-header label {
        margin-bottom: 0;
    }

    .header-actions {
        display: flex;
        align-items: center;
        gap: 16px;
    }

    /* Settings button — clean, no emoji */
    .settings-btn {
        background: transparent;
        border: none;
        cursor: pointer;
        padding: 4px;
        border-radius: var(--radius-sm);
        color: var(--text-tertiary);
        display: flex;
        align-items: center;
        justify-content: center;
        transition: color var(--transition-fast);
    }

    .settings-btn:hover {
        color: var(--text-primary);
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
        background: var(--ds-white);
        box-shadow: var(--shadow-ring);
        border: none;
        border-radius: var(--radius-md);
        font-size: 14px;
        font-family: var(--font-sans);
        cursor: pointer;
        display: flex;
        align-items: center;
        transition: box-shadow var(--transition-fast);
    }

    .custom-provider-trigger:hover {
        box-shadow: var(--shadow-ring),
                    0 0 0 2px var(--ds-white),
                    0 0 0 4px var(--ds-focus-color);
    }

    .custom-provider-trigger.open {
        box-shadow: var(--shadow-ring),
                    0 0 0 2px var(--ds-white),
                    0 0 0 4px var(--ds-focus-color);
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
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-full-card);
        z-index: 100;
        opacity: 0;
        visibility: hidden;
        transform: translateY(-4px);
        transition: all var(--transition-fast);
        max-height: 300px;
        overflow-y: auto;
    }

    .custom-provider-dropdown.open {
        opacity: 1;
        visibility: visible;
        transform: translateY(0);
    }

    .provider-search-input {
        width: calc(100% - 24px);
        margin: 8px 12px;
        height: 36px;
        border-radius: var(--radius-sm);
        box-shadow: var(--shadow-ring);
        border: none;
        padding: 0 10px;
        font-size: 13px;
        font-family: var(--font-sans);
    }

    .provider-option {
        padding: 8px 12px;
        cursor: pointer;
        transition: background var(--transition-fast);
        display: flex;
        align-items: center;
        font-size: 14px;
        font-family: var(--font-sans);
    }

    .provider-option:hover {
        background: var(--ds-gray-50);
    }

    .provider-option.selected {
        color: var(--ds-console-blue);
        font-weight: 500;
    }

    .provider-option.highlighted {
        background: var(--ds-gray-50);
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
    }

    .switch-label input[type="checkbox"] {
        display: none;
    }

    .slider {
        position: relative;
        width: 36px;
        height: 20px;
        background-color: var(--ds-gray-100);
        border-radius: 10px;
        transition: background-color var(--transition-fast);
        flex-shrink: 0;
    }

    .slider::before {
        content: '';
        position: absolute;
        height: 16px;
        width: 16px;
        left: 2px;
        top: 2px;
        background-color: var(--ds-white);
        border-radius: 50%;
        transition: transform var(--transition-fast);
        box-shadow: 0 1px 2px rgba(0,0,0,0.1);
    }

    .switch-label input:checked + .slider {
        background-color: var(--ds-gray-900);
    }

    .switch-label input:checked + .slider::before {
        transform: translateX(16px);
    }

    .switch-title {
        font-weight: 500;
        color: var(--text-secondary);
        font-size: 13px;
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
