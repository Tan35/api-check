<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { useKeyManagerStore } from '@/stores/keyManager';
import { useUiStore } from '@/stores/ui';
import { useCheckerStore } from '@/stores/checker';
import { useConfigStore } from '@/stores/config';
import { fetchModels } from '@/api';

const keyManager = useKeyManagerStore();
const uiStore = useUiStore();
const checkerStore = useCheckerStore();
const configStore = useConfigStore();

/**
 * @description 从 uiStore.modalData 读取 keyId（由 ModalContainer 调用）。
 */
const keyId = computed(() => uiStore.modalData?.keyId || null);

const isEditing = ref(false);
const editAlias = ref('');
const editProvider = ref('');
const isTesting = ref(false);
const isLoadingModels = ref(false);
const modelSearch = ref('');
const showBalanceHistory = ref(false);

/**
 * @description 当前 Key 记录。
 */
const keyRecord = computed(() => {
    return keyManager.keys.find(k => k.id === keyId.value) || null;
});

/**
 * @description 平台名称。
 */
const providerLabel = computed(() => {
    if (!keyRecord.value) return '';
    return configStore.providers[keyRecord.value.provider]?.label || keyRecord.value.provider;
});

/**
 * @description 状态中文名。
 */
const statusText = computed(() => {
    if (!keyRecord.value) return '';
    const map = {
        valid: '有效',
        invalid: '无效',
        rateLimit: '限流',
        unknown: '未知',
    };
    return map[keyRecord.value.status] || '未知';
});

/**
 * @description 状态样式类。
 */
const statusClass = computed(() => {
    if (!keyRecord.value) return '';
    const map = {
        valid: 'status-valid',
        invalid: 'status-invalid',
        rateLimit: 'status-rateLimit',
        unknown: 'status-unknown',
    };
    return map[keyRecord.value.status] || 'status-unknown';
});

/**
 * @description 过滤后的模型列表。
 */
const filteredModels = computed(() => {
    if (!keyRecord.value) return [];
    if (!modelSearch.value) return keyRecord.value.models;
    const term = modelSearch.value.toLowerCase();
    return keyRecord.value.models.filter(m => m.toLowerCase().includes(term));
});

/**
 * @description 格式化日期。
 */
function formatDate(iso) {
    if (!iso) return '-';
    return new Date(iso).toLocaleString('zh-CN');
}

/**
 * @description 格式化余额。
 */
function formatBalance(balance, currency) {
    if (balance === null || balance === undefined) return '-';
    if (currency === 'USD') return `$${balance.toFixed(4)}`;
    return `${balance} ${currency || ''}`;
}

onMounted(() => {
    if (keyRecord.value) {
        editAlias.value = keyRecord.value.alias;
        editProvider.value = keyRecord.value.provider;
        keyManager.loadBalanceHistory(keyId.value);
    }
});

/**
 * @description 保存编辑。
 */
async function saveEdit() {
    await keyManager.updateKey(keyId.value, {
        alias: editAlias.value.trim(),
        provider: editProvider.value,
    });
    isEditing.value = false;
    uiStore.showToast('已保存', 'success');
}

/**
 * @description 测试连接。
 */
async function testConnection() {
    if (!keyRecord.value || isTesting.value) return;

    isTesting.value = true;
    try {
        // 构建检测参数
        const providerConfig = {
            provider: keyRecord.value.provider,
            baseUrl: keyRecord.value.baseUrl || configStore.providers[keyRecord.value.provider]?.defaultBase || '',
            model: keyRecord.value.model || configStore.providers[keyRecord.value.provider]?.defaultModel || '',
            enableStream: false,
            region: configStore.currentRegion,
            validationPrompt: configStore.validationPrompt,
            validationMaxTokens: configStore.validationMaxTokens,
            validationMaxOutputTokens: configStore.validationMaxOutputTokens,
        };

        // 直接调用后端 WebSocket 检测
        const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
        const host = window.location.host;

        const result = await new Promise((resolve, reject) => {
            const ws = new WebSocket(`${protocol}//${host}/check`);
            const timeout = setTimeout(() => {
                ws.close();
                reject(new Error('测试超时'));
            }, 30000);

            ws.onopen = () => {
                ws.send(JSON.stringify({
                    command: 'start',
                    data: {
                        tokens: [{ token: keyRecord.value.token, order: 0 }],
                        providerConfig,
                        concurrency: 1,
                    },
                }));
            };

            ws.onmessage = (event) => {
                const msg = JSON.parse(event.data);
                if (msg.type === 'result') {
                    clearTimeout(timeout);
                    ws.close();
                    resolve(msg.data);
                } else if (msg.type === 'error') {
                    clearTimeout(timeout);
                    ws.close();
                    reject(new Error(msg.message));
                }
            };

            ws.onerror = () => {
                clearTimeout(timeout);
                reject(new Error('WebSocket 连接失败'));
            };

            ws.onclose = () => {
                clearTimeout(timeout);
            };
        });

        // 更新 Key 状态
        await keyManager.updateKeyFromCheck(keyRecord.value.token, result);

        if (result.isValid) {
            const balMsg = result.balance !== undefined && result.balance !== -1
                ? `，余额: ${formatBalance(result.balance, result.currency)}`
                : '';
            uiStore.showToast(`测试通过${balMsg}`, 'success');
        } else {
            uiStore.showToast(`测试失败: ${result.message || 'Key 无效'}`, 'error');
        }
    } catch (err) {
        const msg = err.message || '未知错误';
        if (msg.includes('WebSocket') || msg.includes('连接失败') || msg.includes('超时')) {
            uiStore.showToast('测试失败：后端服务未运行，请使用 npm run dev:full 启动', 'error', 5000);
        } else {
            uiStore.showToast(`测试失败: ${msg}`, 'error');
        }
    } finally {
        isTesting.value = false;
    }
}

/**
 * @description 获取可用模型列表。
 */
async function handleFetchModels() {
    if (!keyRecord.value || isLoadingModels.value) return;

    isLoadingModels.value = true;
    try {
        const providerConfig = {
            currentProvider: keyRecord.value.provider,
            baseUrl: keyRecord.value.baseUrl || configStore.providers[keyRecord.value.provider]?.defaultBase || '',
            currentRegion: configStore.currentRegion,
        };

        const models = await fetchModels(keyRecord.value.token, providerConfig);
        await keyManager.updateKeyModels(keyId.value, models);
        uiStore.showToast(`获取到 ${models.length} 个模型`, 'success');
    } catch (err) {
        const msg = err.message || '未知错误';
        if (msg.includes('fetch') || msg.includes('network') || msg.includes('Failed') || msg.includes('unknown error')) {
            uiStore.showToast('获取模型失败：后端服务未运行，请使用 npm run dev:full 启动', 'error', 5000);
        } else {
            uiStore.showToast(`获取模型失败: ${msg}`, 'error');
        }
    } finally {
        isLoadingModels.value = false;
    }
}

/**
 * @description 复制 Token。
 */
async function copyToken() {
    if (!keyRecord.value) return;
    try {
        await navigator.clipboard.writeText(keyRecord.value.token);
        uiStore.showToast('已复制', 'success', 1500);
    } catch {
        uiStore.showToast('复制失败', 'error');
    }
}

/**
 * @description 删除 Key。
 */
async function handleDelete() {
    const confirmed = await uiStore.showConfirmation('确定删除这个 Key？');
    if (confirmed) {
        await keyManager.deleteKey(keyId.value);
        uiStore.closeModal();
        uiStore.showToast('已删除', 'success');
    }
}

/**
 * @description 添加标签。
 */
const newTag = ref('');
async function addTag() {
    const tag = newTag.value.trim();
    if (!tag) return;
    await keyManager.addTag(keyId.value, tag);
    newTag.value = '';
}

/**
 * @description 移除标签。
 */
async function removeTag(tag) {
    await keyManager.removeTag(keyId.value, tag);
}
</script>

<template>
        <div class="detail-modal" v-if="keyRecord">
            <div class="detail-header">
                <h3>Key 详情</h3>
                <button @click="uiStore.closeModal()" class="detail-close">×</button>
            </div>

            <div class="detail-body">
                <!-- 基本信息 -->
                <div class="detail-section">
                    <div class="detail-row">
                        <span class="detail-label">平台</span>
                        <span v-if="!isEditing" class="detail-value">{{ providerLabel }}</span>
                        <select v-else v-model="editProvider" class="detail-select">
                            <option v-for="(val, key) in configStore.providers" :key="key" :value="key">
                                {{ val.label }}
                            </option>
                        </select>
                    </div>

                    <div class="detail-row">
                        <span class="detail-label">状态</span>
                        <span class="status-badge" :class="statusClass">{{ statusText }}</span>
                    </div>

                    <div class="detail-row">
                        <span class="detail-label">别名</span>
                        <span v-if="!isEditing" class="detail-value">{{ keyRecord.alias || '-' }}</span>
                        <input v-else v-model="editAlias" class="detail-input" placeholder="输入别名" />
                    </div>

                    <div class="detail-row token-row">
                        <span class="detail-label">Key</span>
                        <span class="detail-value token-value" @click="copyToken" title="点击复制">
                            {{ keyRecord.token.substring(0, 20) }}...
                            <span class="copy-hint">复制</span>
                        </span>
                    </div>

                    <div class="detail-row" v-if="keyRecord.baseUrl">
                        <span class="detail-label">Base URL</span>
                        <span class="detail-value url-value">{{ keyRecord.baseUrl }}</span>
                    </div>

                    <div class="detail-row" v-if="keyRecord.model">
                        <span class="detail-label">模型</span>
                        <span class="detail-value">{{ keyRecord.model }}</span>
                    </div>

                    <div class="detail-row" v-if="keyRecord.balance !== null && keyRecord.balance !== undefined">
                        <span class="detail-label">余额</span>
                        <span class="detail-value balance-value">
                            {{ formatBalance(keyRecord.balance, keyRecord.currency) }}
                        </span>
                    </div>

                    <div class="detail-row">
                        <span class="detail-label">创建时间</span>
                        <span class="detail-value">{{ formatDate(keyRecord.createdAt) }}</span>
                    </div>

                    <div class="detail-row">
                        <span class="detail-label">最近检测</span>
                        <span class="detail-value">{{ formatDate(keyRecord.lastChecked) }}</span>
                    </div>
                </div>

                <!-- 操作按钮 -->
                <div class="detail-actions">
                    <button
                        v-if="!isEditing"
                        @click="isEditing = true"
                        class="detail-btn"
                    >
                        编辑
                    </button>
                    <button
                        v-if="isEditing"
                        @click="saveEdit"
                        class="detail-btn primary"
                    >
                        保存
                    </button>
                    <button
                        v-if="isEditing"
                        @click="isEditing = false"
                        class="detail-btn"
                    >
                        取消
                    </button>

                    <button
                        @click="testConnection"
                        :disabled="isTesting"
                        class="detail-btn primary"
                    >
                        {{ isTesting ? '测试中...' : '测试连接' }}
                    </button>

                    <button
                        @click="handleFetchModels"
                        :disabled="isLoadingModels"
                        class="detail-btn"
                    >
                        {{ isLoadingModels ? '获取中...' : '获取模型' }}
                    </button>

                    <button @click="handleDelete" class="detail-btn danger">
                        删除
                    </button>
                </div>

                <!-- 标签 -->
                <div class="detail-section">
                    <h4>标签</h4>
                    <div class="tag-list">
                        <span
                            v-for="tag in keyRecord.tags"
                            :key="tag"
                            class="tag-item"
                        >
                            {{ tag }}
                            <button @click="removeTag(tag)" class="tag-remove">✕</button>
                        </span>
                    </div>
                    <div class="tag-add">
                        <input
                            v-model="newTag"
                            class="tag-input"
                            placeholder="添加标签..."
                            @keyup.enter="addTag"
                        />
                        <button @click="addTag" class="detail-btn small">+</button>
                    </div>
                </div>

                <!-- 余额历史 -->
                <div class="detail-section" v-if="keyManager.balanceHistory.length > 0">
                    <h4
                        @click="showBalanceHistory = !showBalanceHistory"
                        class="section-toggle"
                    >
                        余额历史 ({{ keyManager.balanceHistory.length }})
                        <span>{{ showBalanceHistory ? '收起' : '展开' }}</span>
                    </h4>
                    <div v-if="showBalanceHistory" class="balance-history">
                        <div
                            v-for="snap in keyManager.balanceHistory"
                            :key="snap.id"
                            class="balance-snap"
                        >
                            <span class="snap-time">{{ formatDate(snap.timestamp) }}</span>
                            <span class="snap-balance">{{ formatBalance(snap.balance, snap.currency) }}</span>
                        </div>
                    </div>
                </div>

                <!-- 可用模型 -->
                <div class="detail-section" v-if="keyRecord.models.length > 0">
                    <h4>可用模型 ({{ keyRecord.models.length }})</h4>
                    <input
                        v-model="modelSearch"
                        class="model-search"
                        placeholder="搜索模型..."
                    />
                    <div class="model-list">
                        <div
                            v-for="model in filteredModels"
                            :key="model"
                            class="model-item"
                        >
                            {{ model }}
                        </div>
                    </div>
                </div>
            </div>
        </div>
</template>

<style scoped>
    .detail-modal {
        background: var(--bg-paper);
        border-radius: var(--radius-lg);
        width: 90%;
        max-width: 520px;
        max-height: 80vh;
        display: flex;
        flex-direction: column;
        box-shadow: var(--shadow-full-card);
    }

    .detail-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 16px 20px;
        box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
    }

    .detail-header h3 {
        margin: 0;
        font-size: 16px;
        font-family: var(--font-serif);
        font-weight: 600;
    }

    .detail-close {
        background: transparent;
        font-size: 18px;
        cursor: pointer;
        opacity: 0.5;
    }

    .detail-close:hover {
        opacity: 1;
    }

    .detail-body {
        padding: 20px;
        overflow-y: auto;
        flex: 1;
    }

    .detail-section {
        margin-bottom: 20px;
    }

    .detail-section h4 {
        margin: 0 0 10px 0;
        font-size: 14px;
        color: var(--text-secondary);
        box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
        padding-bottom: 6px;
    }

    .section-toggle {
        cursor: pointer;
        display: flex;
        justify-content: space-between;
        align-items: center;
    }

    .section-toggle:hover {
        color: var(--accent-primary);
    }

    .detail-row {
        display: flex;
        align-items: center;
        padding: 8px 0;
        box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
    }

    .detail-row:last-child {
        border-bottom: none;
    }

    .detail-label {
        width: 80px;
        font-size: 13px;
        color: var(--text-secondary);
        flex-shrink: 0;
    }

    .detail-value {
        flex: 1;
        font-size: 14px;
        word-break: break-all;
    }

    .token-value {
        font-family: var(--font-mono);
        font-size: 13px;
        cursor: pointer;
        padding: 4px 8px;
        background: var(--bg-secondary);
        border-radius: var(--radius-sm);
        display: inline-flex;
        align-items: center;
        gap: 8px;
    }

    .token-value:hover {
        background: var(--bg-tertiary);
    }

    .copy-hint {
        color: var(--ds-blue-text);
        font-family: var(--font-sans);
        font-size: 12px;
    }

    .url-value {
        font-family: var(--font-mono);
        font-size: 12px;
        color: var(--text-tertiary);
    }

    .balance-value {
        font-weight: 600;
        color: var(--accent-primary);
    }

    .status-badge {
        font-size: 12px;
        font-weight: 500;
        padding: 2px 10px;
        border-radius: var(--radius-badge);
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
        background: #f3f4f6;
        color: #6b7280;
    }

    .detail-select,
    .detail-input {
        flex: 1;
        height: 32px;
        padding: 0 8px;
        border: none;
        box-shadow: var(--shadow-ring);
        border-radius: var(--radius-sm);
        font-size: 13px;
        font-family: var(--font-sans);
    }

    /* 操作按钮 */
    .detail-actions {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;
        margin-bottom: 20px;
    }

    .detail-btn {
        height: 32px;
        padding: 0 12px;
        border: none;
        box-shadow: var(--shadow-light-ring);
        border-radius: var(--radius-sm);
        background: var(--bg-surface);
        font-size: 12px;
        font-family: var(--font-sans);
        cursor: pointer;
        transition: all 0.2s;
    }

    .detail-btn:hover {
        background: var(--ds-gray-50);
    }

    .detail-btn.primary {
        background: var(--ds-gray-1000);
        color: var(--ds-white);
        box-shadow: none;
    }

    .detail-btn.primary:hover {
        background: var(--ds-black);
    }

    .detail-btn.danger {
        background: var(--ds-red);
        color: var(--ds-white);
        box-shadow: none;
    }

    .detail-btn.danger:hover {
        background: var(--ds-red-dark);
    }

    .detail-btn.small {
        height: 28px;
        padding: 0 8px;
    }

    .detail-btn:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    /* 标签 */
    .tag-list {
        display: flex;
        gap: 6px;
        flex-wrap: wrap;
        margin-bottom: 8px;
    }

    .tag-item {
        display: flex;
        align-items: center;
        gap: 4px;
        padding: 2px 8px;
        background: var(--bg-tertiary);
        border-radius: var(--radius-badge);
        font-size: 12px;
        color: var(--text-secondary);
    }

    .tag-remove {
        background: transparent;
        border: none;
        font-size: 10px;
        cursor: pointer;
        opacity: 0.5;
    }

    .tag-remove:hover {
        opacity: 1;
        color: var(--ds-red);
    }

    .tag-add {
        display: flex;
        gap: 6px;
    }

    .tag-input {
        flex: 1;
        height: 28px;
        padding: 0 8px;
        border: none;
        box-shadow: var(--shadow-ring);
        border-radius: var(--radius-sm);
        font-size: 12px;
        font-family: var(--font-sans);
    }

    /* 余额历史 */
    .balance-history {
        max-height: 200px;
        overflow-y: auto;
    }

    .balance-snap {
        display: flex;
        justify-content: space-between;
        padding: 6px 0;
        box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
        font-size: 13px;
    }

    .snap-time {
        color: var(--text-secondary);
    }

    .snap-balance {
        font-weight: 600;
        color: var(--accent-primary);
    }

    /* 模型列表 */
    .model-search {
        width: 100%;
        height: 32px;
        padding: 0 8px;
        border: none;
        box-shadow: var(--shadow-ring);
        border-radius: var(--radius-sm);
        font-size: 12px;
        font-family: var(--font-sans);
        margin-bottom: 8px;
    }

    .model-list {
        max-height: 200px;
        overflow-y: auto;
    }

    .model-item {
        padding: 6px 8px;
        box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
        font-size: 12px;
        font-family: var(--font-mono);
        color: var(--text-secondary);
    }

    .model-item:last-child {
        border-bottom: none;
    }
</style>
