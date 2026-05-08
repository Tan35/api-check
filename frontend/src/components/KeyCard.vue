<script setup>
import { computed } from 'vue';
import { useKeyManagerStore } from '@/stores/keyManager';
import { useUiStore } from '@/stores/ui';

const props = defineProps({
    keyRecord: {
        type: Object,
        required: true,
    },
    selected: {
        type: Boolean,
        default: false,
    },
});

const emit = defineEmits(['select', 'delete']);

const keyManager = useKeyManagerStore();
const uiStore = useUiStore();

const maskedToken = computed(() => {
    const token = props.keyRecord.token;
    if (token.length <= 8) return '****';
    return token.substring(0, 4) + '****' + token.substring(token.length - 4);
});

const statusClass = computed(() => {
    const map = {
        valid: 'status-valid',
        invalid: 'status-invalid',
        rateLimit: 'status-rateLimit',
        unknown: 'status-unknown',
    };
    return map[props.keyRecord.status] || 'status-unknown';
});

const statusText = computed(() => {
    const map = {
        valid: '有效',
        invalid: '无效',
        rateLimit: '限流',
        unknown: '未知',
    };
    return map[props.keyRecord.status] || '未知';
});

const balanceText = computed(() => {
    const b = props.keyRecord.balance;
    if (b === null || b === undefined) return null;
    const c = props.keyRecord.currency || 'USD';
    if (c === 'USD') return `$${b.toFixed(2)}`;
    return `${b} ${c}`;
});

const providerLabel = computed(() => {
    const map = {
        openai_responses: 'OpenAI (R)',
        openai: 'OpenAI',
        anthropic: 'Anthropic',
        gemini: 'Gemini',
        deepseek: 'DeepSeek',
        groq: 'Groq',
        moonshot: 'Moonshot',
        siliconflow: 'SiliconFlow',
        xai: 'xAI',
        zhipu: 'Zhipu',
        qwen: 'Qwen',
    };
    return map[props.keyRecord.provider] || props.keyRecord.provider;
});

function formatDate(iso) {
    if (!iso) return '-';
    return new Date(iso).toLocaleString('zh-CN', {
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
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
    if (confirmed) {
        emit('delete', props.keyRecord.id);
    }
}
</script>

<template>
    <div class="key-card" :class="{ selected }" @click="emit('select', keyRecord.id)">
        <div class="key-card-header">
            <span class="provider-pill">{{ providerLabel }}</span>
            <span class="status-pill" :class="statusClass">{{ statusText }}</span>
        </div>

        <div class="key-card-token" @click.stop="copyToken" title="点击复制">
            <span class="token-text">{{ maskedToken }}</span>
            <svg class="copy-icon" width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.5">
                <rect x="4.5" y="4.5" width="7" height="7" rx="1"/>
                <path d="M9.5 4.5V2.5a1 1 0 00-1-1h-6a1 1 0 00-1 1v6a1 1 0 001 1h2"/>
            </svg>
        </div>

        <div v-if="keyRecord.alias" class="key-card-alias">
            {{ keyRecord.alias }}
        </div>

        <div class="key-card-meta">
            <span v-if="balanceText" class="meta-item">
                {{ balanceText }}
            </span>
            <span v-if="keyRecord.models.length > 0" class="meta-item">
                {{ keyRecord.models.length }} 模型
            </span>
        </div>

        <div class="key-card-footer">
            <span class="meta-date" v-if="keyRecord.lastChecked">
                {{ formatDate(keyRecord.lastChecked) }}
            </span>
            <span class="meta-date" v-else>
                未检测
            </span>

            <div class="key-card-actions" @click.stop>
                <button @click="confirmDelete" class="action-btn" title="删除">
                    <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="1.5">
                        <path d="M2.5 4h9M5 4V2.5a.5.5 0 01.5-.5h3a.5.5 0 01.5.5V4M11 4v7.5a1 1 0 01-1 1H4a1 1 0 01-1-1V4"/>
                    </svg>
                </button>
            </div>
        </div>

        <div v-if="keyRecord.tags.length > 0" class="key-card-tags">
            <span v-for="tag in keyRecord.tags" :key="tag" class="tag-pill">
                {{ tag }}
            </span>
        </div>
    </div>
</template>

<style scoped>
    .key-card {
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        padding: 12px;
        cursor: pointer;
        transition: box-shadow var(--transition-fast), background var(--transition-fast);
        box-shadow: var(--shadow-light-ring);
    }

    .key-card:hover {
        background: var(--ds-gray-50);
        box-shadow: var(--shadow-card);
    }

    .key-card.selected {
        box-shadow: var(--shadow-full-card),
                    0 0 0 2px var(--ds-white),
                    0 0 0 4px var(--ds-focus-color);
    }

    .key-card-header {
        display: flex;
        align-items: center;
        gap: 6px;
        margin-bottom: 8px;
    }

    .provider-pill {
        font-size: 11px;
        font-weight: 500;
        padding: 2px 8px;
        border-radius: 9999px;
        background: var(--ds-gray-1000);
        color: var(--ds-white);
    }

    .status-pill {
        font-size: 11px;
        font-weight: 500;
        padding: 2px 8px;
        border-radius: 9999px;
    }

    .status-valid {
        background: var(--ds-blue-soft);
        color: var(--ds-blue-text);
    }

    .status-invalid {
        background: #fff1f0;
        color: var(--ds-red-dark);
    }

    .status-rateLimit {
        background: #fdf2fa;
        color: var(--ds-pink);
    }

    .status-unknown {
        background: var(--ds-gray-100);
        color: var(--ds-gray-500);
    }

    .key-card-token {
        display: flex;
        align-items: center;
        gap: 6px;
        padding: 6px 8px;
        background: var(--ds-gray-50);
        border-radius: var(--radius-sm);
        margin-bottom: 6px;
        cursor: pointer;
        transition: background var(--transition-fast);
    }

    .key-card-token:hover {
        background: var(--ds-gray-100);
    }

    .token-text {
        font-family: var(--font-mono);
        font-size: 12px;
        color: var(--text-secondary);
        flex: 1;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .copy-icon {
        color: var(--text-tertiary);
        flex-shrink: 0;
    }

    .key-card-alias {
        font-size: 13px;
        font-weight: 500;
        color: var(--text-primary);
        margin-bottom: 6px;
    }

    .key-card-meta {
        display: flex;
        gap: 10px;
        margin-bottom: 6px;
        flex-wrap: wrap;
    }

    .meta-item {
        font-size: 12px;
        color: var(--text-secondary);
        font-family: var(--font-mono);
        font-variant-numeric: tabular-nums;
    }

    .key-card-footer {
        display: flex;
        align-items: center;
        justify-content: space-between;
    }

    .meta-date {
        font-size: 11px;
        color: var(--text-tertiary);
    }

    .key-card-actions {
        display: flex;
        gap: 4px;
    }

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
        transition: color var(--transition-fast);
    }

    .action-btn:hover {
        color: var(--ds-ship-red);
    }

    .key-card-tags {
        display: flex;
        gap: 4px;
        margin-top: 8px;
        flex-wrap: wrap;
    }

    .tag-pill {
        font-size: 11px;
        padding: 2px 8px;
        border-radius: 9999px;
        background: var(--ds-badge-blue-bg);
        color: var(--ds-badge-blue-text);
        font-weight: 500;
    }
</style>
