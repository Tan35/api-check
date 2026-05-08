<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount, nextTick } from 'vue';

const props = defineProps({
    modelValue: { type: [String, Number], required: true },
    options: { type: Array, required: true }, // [{ key, label }]
    placeholder: { type: String, default: 'Select...' },
    disabled: { type: Boolean, default: false },
});

const emit = defineEmits(['update:modelValue']);

const isOpen = ref(false);
const containerRef = ref(null);
const searchTerm = ref('');
const highlightedIndex = ref(-1);

const filteredOptions = computed(() => {
    const term = searchTerm.value.toLowerCase();
    if (!term) return props.options;
    return props.options.filter(o => o.label.toLowerCase().includes(term));
});

const optionKeys = computed(() => filteredOptions.value.map(o => o.key));

const selectedLabel = computed(() => {
    const found = props.options.find(o => o.key === props.modelValue);
    return found ? found.label : props.placeholder;
});

function toggle() {
    if (props.disabled) return;
    isOpen.value = !isOpen.value;
    if (isOpen.value) {
        highlightedIndex.value = -1;
        searchTerm.value = '';
        nextTick(() => {
            const input = containerRef.value?.querySelector('input');
            if (input) input.focus();
        });
    }
}

function select(key) {
    emit('update:modelValue', key);
    isOpen.value = false;
    searchTerm.value = '';
}

function close() {
    isOpen.value = false;
    searchTerm.value = '';
}

function handleKeyDown(e) {
    if (!isOpen.value) {
        if (e.key === 'Enter' || e.key === ' ') {
            e.preventDefault();
            toggle();
        }
        return;
    }

    if (e.key === 'Escape') {
        close();
        return;
    }

    const keys = optionKeys.value;
    if (!keys.length) return;

    if (e.key === 'ArrowDown') {
        e.preventDefault();
        highlightedIndex.value = (highlightedIndex.value + 1) % keys.length;
    } else if (e.key === 'ArrowUp') {
        e.preventDefault();
        highlightedIndex.value = (highlightedIndex.value - 1 + keys.length) % keys.length;
    } else if (e.key === 'Enter') {
        e.preventDefault();
        const key = keys[highlightedIndex.value];
        if (key !== undefined) select(key);
    }
}

function handleOutsideClick(e) {
    if (containerRef.value && !containerRef.value.contains(e.target)) {
        close();
    }
}

onMounted(() => document.addEventListener('click', handleOutsideClick));
onBeforeUnmount(() => document.removeEventListener('click', handleOutsideClick));

watch(isOpen, (v) => {
    if (v) highlightedIndex.value = -1;
});
</script>

<template>
    <div class="cs-wrapper" ref="containerRef" :class="{ disabled }">
        <button
            class="cs-trigger"
            :class="{ open: isOpen }"
            @click="toggle"
            @keydown="handleKeyDown"
            type="button"
            role="combobox"
            :aria-expanded="isOpen"
            :disabled="disabled"
        >
            <span class="cs-value">{{ selectedLabel }}</span>
            <span class="cs-chevron"></span>
        </button>
        <Transition name="cs-drop">
            <div v-if="isOpen" class="cs-dropdown">
                <input
                    type="text"
                    v-model="searchTerm"
                    class="cs-search"
                    placeholder="Search..."
                    @keydown="handleKeyDown"
                />
                <div class="cs-options">
                    <div
                        v-for="(opt, idx) in filteredOptions"
                        :key="opt.key"
                        class="cs-option"
                        :class="{ selected: opt.key === modelValue, highlighted: idx === highlightedIndex }"
                        @click="select(opt.key)"
                    >
                        {{ opt.label }}
                    </div>
                    <div v-if="filteredOptions.length === 0" class="cs-empty">No matches</div>
                </div>
            </div>
        </Transition>
    </div>
</template>

<style scoped>
    .cs-wrapper {
        position: relative;
        width: 100%;
    }

    .cs-wrapper.disabled {
        opacity: 0.5;
        pointer-events: none;
    }

    .cs-trigger {
        width: 100%;
        height: 40px;
        padding: 0 32px 0 12px;
        background: var(--ds-white);
        box-shadow: var(--shadow-ring);
        border: none;
        border-radius: var(--radius-md);
        font-size: 14px;
        font-family: var(--font-sans);
        color: var(--text-primary);
        cursor: pointer;
        display: flex;
        align-items: center;
        transition: box-shadow var(--transition-fast);
        text-align: left;
    }

    .cs-trigger:hover {
        box-shadow: var(--shadow-ring), 0 0 0 2px var(--ds-white), 0 0 0 4px var(--ds-focus-color);
    }

    .cs-trigger.open {
        box-shadow: var(--shadow-ring), 0 0 0 2px var(--ds-white), 0 0 0 4px var(--ds-focus-color);
    }

    .cs-value {
        flex: 1;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .cs-chevron {
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

    .cs-trigger.open .cs-chevron {
        transform: translateY(-35%) rotate(-135deg);
    }

    .cs-dropdown {
        position: absolute;
        top: calc(100% + 4px);
        left: 0;
        right: 0;
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-full-card);
        z-index: 100;
        overflow: hidden;
    }

    .cs-search {
        width: calc(100% - 24px);
        margin: 8px 12px;
        height: 36px;
        border: none;
        box-shadow: var(--shadow-ring);
        border-radius: var(--radius-sm);
        padding: 0 10px;
        font-size: 13px;
        font-family: var(--font-sans);
        outline: none;
    }

    .cs-options {
        max-height: 240px;
        overflow-y: auto;
        padding-bottom: 4px;
    }

    .cs-option {
        padding: 8px 12px;
        cursor: pointer;
        font-size: 14px;
        font-family: var(--font-sans);
        transition: background var(--transition-fast);
    }

    .cs-option:hover,
    .cs-option.highlighted {
        background: var(--ds-gray-50);
    }

    .cs-option.selected {
        color: var(--ds-console-blue);
        font-weight: 500;
    }

    .cs-empty {
        padding: 12px;
        text-align: center;
        font-size: 13px;
        color: var(--text-tertiary);
    }

    /* Transition */
    .cs-drop-enter-active {
        transition: all 0.15s ease-out;
    }

    .cs-drop-leave-active {
        transition: all 0.1s ease-in;
    }

    .cs-drop-enter-from,
    .cs-drop-leave-to {
        opacity: 0;
        transform: translateY(-4px);
    }
</style>
