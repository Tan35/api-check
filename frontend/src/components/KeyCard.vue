<script setup>
import { computed } from 'vue';
import { useKeyManagerStore } from '@/stores/keyManager';
import { useUiStore } from '@/stores/ui';

const props = defineProps({
    keyRecord: { type: Object, required: true },
    selected:  { type: Boolean, default: false },
});
const emit = defineEmits(['select', 'delete']);

const keyManager = useKeyManagerStore();
const uiStore    = useUiStore();

const maskedToken = computed(() => {
    const t = props.keyRecord.token;
    if (t.length <= 8) return '****';
    return t.substring(0, 4) + '****' + t.substring(t.length - 4);
});

const statusClass = computed(() => ({
    valid:     'status-valid',
    invalid:   'status-invalid',
    rateLimit: 'status-rateLimit',
    unknown:   'status-unknown',
}[props.keyRecord.status] || 'status-unknown'));

const statusText = computed(() => ({
    valid: '有效', invalid: '无效', rateLimit: '限流', unknown: '未知',
}[props.keyRecord.status] || '未知'));

const balanceText = computed(() => {
    const b = props.keyRecord.balance;
    if (b === null || b === undefined) return null;
    const c = props.keyRecord.currency || 'USD';
    return c === 'USD' ? `$${b.toFixed(2)}` : `${b} ${c}`;
});

const providerLabel = computed(() => {
    const map = {
        openai_responses: 'OpenAI (R)', openai: 'OpenAI', anthropic: 'Anthropic',
        gemini: 'Gemini', deepseek: 'DeepSeek', groq: 'Groq', moonshot: 'Moonshot',
        siliconflow: 'SiliconFlow', xai: 'xAI', zhipu: 'Zhipu', qwen: 'Qwen',
    };
    return map[props.keyRecord.provider] || props.keyRecord.provider;
});

function formatDate(iso) {
    if (!iso) return '-';
    return new Date(iso).toLocaleString('zh-CN', {
        month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit',
    });
}

async function copyToken() {
    try {
        await navigator.clipboard.writeText(props.keyRecord.token);
        uiStore.showToast('已复制到剪贴板', 'success', 1500);
    } catch {
        uiStore.showToast('复制失败', 'error');
    }
}

async function confirmDelete() {
    const confirmed = await uiStore.showConfirmation(`确定删除这个 Key？\n${maskedToken.value}`);
    if (confirmed) emit('delete', props.keyRecord.id);
}
</script>

<template>
    <div class="key-card" :class="{ selected }" @click="emit('select', keyRecord.id)">
        <div class="key-card-header">
            <span class="provider-pill">{{ providerLabel }}</span>
            <span class="status-pill" :class="statusClass">{{ statusText }}</span>
            <span v-if="keyRecord.alias" class="alias-text">{{ keyRecord.alias }}</span>
        </div>

        <div class="key-card-token" @click.stop="copyToken" title="点击复制">
            <svg class="token-icon" width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="1.5">
                <rect x="1" y="3" width="10" height="7" rx="1.5"/>
                <path d="M4 3V2a2 2 0 014 0v1"/>
            </svg>
            <span class="token-text">{{ maskedToken }}</span>
            <svg class="copy-icon" width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="1.5">
                <rect x="4" y="4" width="6" height="6" rx="1"/>
                <path d="M8 4V2.5a.5.5 0 00-.5-.5h-5a.5.5 0 00-.5.5v5a.5.5 0 00.5.5H4"/>
            </svg>
        </div>

        <div class="key-card-footer">
            <div class="key-card-meta">
                <span v-if="balanceText" class="meta-balance">{{ balanceText }}</span>
                <span v-if="keyRecord.models.length > 0" class="meta-chip">{{ keyRecord.models.length }} 模型</span>
                <span class="meta-date">{{ keyRecord.lastChecked ? formatDate(keyRecord.lastChecked) : '未检测' }}</span>
            </div>
            <div class="key-card-actions" @click.stop>
                <button @click="confirmDelete" class="action-btn" title="删除">
                    <svg width="13" height="13" viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.5">
                        <path d="M2.5 4h9M5 4V2.5a.5.5 0 01.5-.5h3a.5.5 0 01.5.5V4M11 4v7.5a1 1 0 01-1 1H4a1 1 0 01-1-1V4"/>
                    </svg>
                </button>
            </div>
        </div>

        <div v-if="keyRecord.tags.length > 0" class="key-card-tags">
            <span v-for="tag in keyRecord.tags" :key="tag" class="tag-pill">{{ tag }}</span>
        </div>
    </div>
</template>

<style scoped>
.key-card {
    background: var(--ds-white);
    border-radius: var(--radius-md);
    padding: 10px 12px;
    cursor: pointer;
    transition: box-shadow var(--transition-fast), background var(--transition-fast);
    box-shadow: var(--shadow-light-ring);
}
.key-card:hover { background: var(--ds-gray-50); box-shadow: var(--shadow-card); }
.key-card.selected {
    box-shadow: var(--shadow-card), 0 0 0 2px var(--ds-white), 0 0 0 3px var(--ds-gray-400);
}

.key-card-header {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-bottom: 7px;
    flex-wrap: wrap;
}

.provider-pill {
    font-size: 11px;
    font-weight: 500;
    padding: 2px 7px;
    border-radius: 9999px;
    background: var(--ds-gray-100);
    color: var(--ds-gray-700);
}

.status-pill {
    font-size: 11px;
    font-weight: 500;
    padding: 2px 7px;
    border-radius: 9999px;
}
.status-valid     { background: #f0fdf4; color: #0a7c42; }
.status-invalid   { background: #fff1f0; color: var(--ds-red-dark); }
.status-rateLimit { background: #fdf2fa; color: var(--ds-pink); }
.status-unknown   { background: var(--ds-gray-100); color: var(--ds-gray-500); }

.alias-text {
    font-size: 12px;
    color: var(--text-secondary);
    font-weight: 500;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    flex: 1;
    min-width: 0;
}

.key-card-token {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 5px 8px;
    background: var(--ds-gray-50);
    border-radius: var(--radius-sm);
    margin-bottom: 8px;
    cursor: pointer;
    transition: background var(--transition-fast);
}
.key-card-token:hover { background: var(--ds-gray-100); }
.token-icon { color: var(--ds-gray-400); flex-shrink: 0; }
.token-text {
    font-family: var(--font-mono);
    font-size: 12px;
    color: var(--text-secondary);
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
.copy-icon { color: var(--ds-gray-400); flex-shrink: 0; }

.key-card-footer {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
}
.key-card-meta {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-wrap: wrap;
    min-width: 0;
}
.meta-balance {
    font-size: 12px;
    font-weight: 600;
    font-family: var(--font-mono);
    color: var(--text-primary);
    font-variant-numeric: tabular-nums;
}
.meta-chip {
    font-size: 11px;
    padding: 1px 6px;
    border-radius: 9999px;
    background: var(--ds-gray-100);
    color: var(--ds-gray-500);
}
.meta-date {
    font-size: 11px;
    color: var(--text-tertiary);
    font-variant-numeric: tabular-nums;
}

.key-card-actions { display: flex; gap: 2px; }
.action-btn {
    background: transparent;
    border: none;
    cursor: pointer;
    padding: 4px;
    border-radius: var(--radius-sm);
    color: var(--text-tertiary);
    display: flex;
    align-items: center;
    justify-content: center;
    transition: color var(--transition-fast), background var(--transition-fast);
}
.action-btn:hover { color: var(--ds-red); background: #fff1f0; }

.key-card-tags {
    display: flex;
    gap: 4px;
    margin-top: 8px;
    flex-wrap: wrap;
}
.tag-pill {
    font-size: 11px;
    padding: 2px 7px;
    border-radius: 9999px;
    background: var(--ds-gray-100);
    color: var(--ds-gray-600);
    font-weight: 500;
}
</style>
