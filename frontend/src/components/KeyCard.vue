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

/**
 * @description 隐藏的 token 显示。
 */
const maskedToken = computed(() => {
    const token = props.keyRecord.token;
    if (token.length <= 8) return '****';
    return token.substring(0, 4) + '****' + token.substring(token.length - 4);
});

/**
 * @description 状态标签样式。
 */
const statusClass = computed(() => {
    const map = {
        valid: 'status-valid',
        invalid: 'status-invalid',
        rateLimit: 'status-rateLimit',
        unknown: 'status-unknown',
    };
    return map[props.keyRecord.status] || 'status-unknown';
});

/**
 * @description 状态中文名。
 */
const statusText = computed(() => {
    const map = {
        valid: '有效',
        invalid: '无效',
        rateLimit: '限流',
        unknown: '未知',
    };
    return map[props.keyRecord.status] || '未知';
});

/**
 * @description 余额显示文本。
 */
const balanceText = computed(() => {
    const b = props.keyRecord.balance;
    if (b === null || b === undefined) return null;
    const c = props.keyRecord.currency || 'USD';
    if (c === 'USD') return `$${b.toFixed(2)}`;
    return `${b} ${c}`;
});

/**
 * @description 平台名称。
 */
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

/**
 * @description 格式化日期。
 */
function formatDate(iso) {
    if (!iso) return '-';
    return new Date(iso).toLocaleString('zh-CN', {
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
    });
}

/**
 * @description 复制 token 到剪贴板。
 */
async function copyToken() {
    try {
        await navigator.clipboard.writeText(props.keyRecord.token);
        uiStore.showToast('已复制到剪贴板', 'success', 1500);
    } catch {
        uiStore.showToast('复制失败', 'error');
    }
}

/**
 * @description 确认删除。
 */
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
            <span class="provider-badge">{{ providerLabel }}</span>
            <span class="status-badge" :class="statusClass">{{ statusText }}</span>
        </div>

        <div class="key-card-token" @click.stop="copyToken" title="点击复制">
            <span class="token-text">{{ maskedToken }}</span>
            <span class="copy-icon">📋</span>
        </div>

        <div v-if="keyRecord.alias" class="key-card-alias">
            {{ keyRecord.alias }}
        </div>

        <div class="key-card-meta">
            <span v-if="balanceText" class="meta-item balance">
                💰 {{ balanceText }}
            </span>
            <span v-if="keyRecord.models.length > 0" class="meta-item models">
                🤖 {{ keyRecord.models.length }} 模型
            </span>
        </div>

        <div class="key-card-footer">
            <span class="meta-date" v-if="keyRecord.lastChecked">
                检测: {{ formatDate(keyRecord.lastChecked) }}
            </span>
            <span class="meta-date" v-else>
                未检测
            </span>

            <div class="key-card-actions" @click.stop>
                <button @click="confirmDelete" class="action-btn delete-btn" title="删除">
                    🗑️
                </button>
            </div>
        </div>

        <div v-if="keyRecord.tags.length > 0" class="key-card-tags">
            <span v-for="tag in keyRecord.tags" :key="tag" class="tag-badge">
                {{ tag }}
            </span>
        </div>
    </div>
</template>

<style scoped>
    .key-card {
        background: var(--bg-surface);
        border: 1px solid var(--border-color);
        border-radius: var(--radius-md);
        padding: 12px;
        cursor: pointer;
        transition: all 0.2s ease;
    }

    .key-card:hover {
        border-color: var(--border-color-focus);
        box-shadow: var(--shadow-subtle);
    }

    .key-card.selected {
        border-color: var(--accent-primary);
        background: var(--bg-selected);
    }

    .key-card-header {
        display: flex;
        align-items: center;
        gap: 8px;
        margin-bottom: 8px;
    }

    .provider-badge {
        font-size: 11px;
        font-weight: 600;
        padding: 2px 8px;
        border-radius: 12px;
        background: var(--accent-dark);
        color: white;
    }

    .status-badge {
        font-size: 11px;
        font-weight: 500;
        padding: 2px 8px;
        border-radius: 12px;
    }

    .status-valid {
        background: #dcfce7;
        color: #166534;
    }

    .status-invalid {
        background: #fee2e2;
        color: #991b1b;
    }

    .status-rateLimit {
        background: #fef3c7;
        color: #92400e;
    }

    .status-unknown {
        background: #f3f4f6;
        color: #6b7280;
    }

    .key-card-token {
        display: flex;
        align-items: center;
        gap: 6px;
        padding: 6px 8px;
        background: var(--bg-secondary);
        border-radius: var(--radius-sm);
        margin-bottom: 6px;
        cursor: pointer;
        transition: background 0.2s;
    }

    .key-card-token:hover {
        background: var(--bg-tertiary);
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
        font-size: 12px;
        opacity: 0.5;
    }

    .key-card-alias {
        font-size: 13px;
        font-weight: 600;
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
    }

    .meta-item.balance {
        font-weight: 600;
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
        font-size: 14px;
        padding: 2px 4px;
        border-radius: 4px;
        opacity: 0.5;
        transition: all 0.2s;
    }

    .action-btn:hover {
        opacity: 1;
        background: var(--bg-tertiary);
    }

    .key-card-tags {
        display: flex;
        gap: 4px;
        margin-top: 8px;
        flex-wrap: wrap;
    }

    .tag-badge {
        font-size: 10px;
        padding: 2px 6px;
        border-radius: 8px;
        background: var(--bg-tertiary);
        color: var(--text-secondary);
    }
</style>
