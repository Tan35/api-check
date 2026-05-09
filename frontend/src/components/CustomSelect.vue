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
const inputRef = ref(null);
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

function open() {
    if (props.disabled || isOpen.value) return;
    isOpen.value = true;
    highlightedIndex.value = -1;
    searchTerm.value = '';
    nextTick(() => inputRef.value?.focus());
}

function close() {
    isOpen.value = false;
    searchTerm.value = '';
}

function toggle() {
    if (isOpen.value) close();
    else open();
}

function select(key) {
    emit('update:modelValue', key);
    close();
}

function handleKeyDown(e) {
    if (!isOpen.value) {
        if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); open(); }
        return;
    }
    if (e.key === 'Escape') { close(); return; }
    if (e.key === 'Tab') { close(); return; }

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
        <div
            class="cs-trigger"
            :class="{ open: isOpen }"
            @click="toggle"
            role="combobox"
            :aria-expanded="isOpen"
        >
            <input
                v-if="isOpen"
                ref="inputRef"
                class="cs-input"
                type="text"
                v-model="searchTerm"
                :placeholder="selectedLabel"
                @keydown="handleKeyDown"
                @click.stop
            />
            <span v-else class="cs-value">{{ selectedLabel }}</span>
            <span class="cs-chevron"></span>
        </div>

        <Transition name="cs-drop">
            <div v-if="isOpen" class="cs-dropdown">
                <div class="cs-options">
                    <div
                        v-for="(opt, idx) in filteredOptions"
                        :key="opt.key"
                        class="cs-option"
                        :class="{ selected: opt.key === modelValue, highlighted: idx === highlightedIndex }"
                        @mousedown.prevent="select(opt.key)"
                    >
                        {{ opt.label }}
                    </div>
                    <div v-if="filteredOptions.length === 0" class="cs-empty">无匹配项</div>
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
        height: var(--ctrl-height-md);
        padding: 0 28px 0 12px;
        background: var(--bg-input);
        box-shadow: var(--shadow-ring);
        border: none;
        border-radius: var(--radius-md);
        font-size: var(--ctrl-font-md);
        font-family: var(--font-sans);
        color: var(--text-primary);
        cursor: pointer;
        display: flex;
        align-items: center;
        transition: box-shadow var(--transition-fast), background var(--transition-fast);
        text-align: left;
        position: relative;
        box-sizing: border-box;
    }

    .cs-trigger:hover {
        background: var(--bg-secondary);
    }

    .cs-trigger.open {
        background: var(--bg-input);
        box-shadow: var(--shadow-ring);
        cursor: text;
    }

    /* 沉浸式搜索输入框——完全清除浏览器默认样式，融入触发器 */
    .cs-input {
        flex: 1;
        width: 0;           /* 让 flex 控制宽度，避免撑破容器 */
        min-width: 0;
        height: 100%;
        border: none;
        outline: none;
        box-shadow: none;
        -webkit-appearance: none;
        appearance: none;
        background: transparent;
        font-size: var(--ctrl-font-md);
        font-family: var(--font-sans);
        color: var(--text-primary);
        padding: 0;
        margin: 0;
    }

    .cs-input::placeholder {
        color: var(--ds-gray-400);
    }

    /* 覆盖全局 input:focus 的 box-shadow */
    .cs-input:focus {
        box-shadow: none !important;
        outline: none;
    }

    .cs-value {
        flex: 1;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .cs-chevron {
        position: absolute;
        right: 10px;
        top: 50%;
        width: 6px;
        height: 6px;
        border-right: 1.5px solid var(--ds-gray-400);
        border-bottom: 1.5px solid var(--ds-gray-400);
        transform: translateY(-65%) rotate(45deg);
        transition: transform var(--transition-fast);
        pointer-events: none;
        flex-shrink: 0;
    }

    .cs-trigger.open .cs-chevron {
        transform: translateY(-35%) rotate(-135deg);
    }

    .cs-dropdown {
        position: absolute;
        top: calc(100% + 4px);
        left: 0;
        right: 0;
        background: var(--bg-surface);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-full-card);
        z-index: 100;
        overflow: hidden;
    }

    .cs-options {
        max-height: 240px;
        overflow-y: auto;
        padding: 4px 0;
    }

    .cs-option {
        padding: 8px 12px;
        cursor: pointer;
        font-size: 14px;
        font-family: var(--font-sans);
        transition: background var(--transition-fast);
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    .cs-option:hover,
    .cs-option.highlighted {
        background: var(--bg-secondary);
    }

    .cs-option.selected {
        color: var(--text-primary);
        font-weight: 500;
    }

    .cs-empty {
        padding: 12px;
        text-align: center;
        font-size: 13px;
        color: var(--text-tertiary);
    }

    /* Transition */
    .cs-drop-enter-active { transition: all 0.15s ease-out; }
    .cs-drop-leave-active { transition: all 0.1s ease-in; }
    .cs-drop-enter-from,
    .cs-drop-leave-to {
        opacity: 0;
        transform: translateY(-4px);
    }
</style>
