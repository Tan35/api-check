<script setup>
import { ref, onMounted, computed } from 'vue';
import { useKeyManagerStore } from '@/stores/keyManager';
import { useUiStore } from '@/stores/ui';
import { useCheckerStore } from '@/stores/checker';
import { useConfigStore } from '@/stores/config';
import { fetchModels } from '@/api';

const keyManager  = useKeyManagerStore();
const uiStore     = useUiStore();
const checkerStore = useCheckerStore();
const configStore = useConfigStore();

const keyId = computed(() => uiStore.modalData?.keyId || null);

const isEditing       = ref(false);
const editAlias       = ref('');
const editProvider    = ref('');
const isTesting       = ref(false);
const isLoadingModels = ref(false);
const modelSearch     = ref('');
const showBalanceHistory = ref(false);
const newTag          = ref('');

const keyRecord = computed(() => keyManager.keys.find(k => k.id === keyId.value) || null);

const providerLabel = computed(() =>
    !keyRecord.value ? '' : (configStore.providers[keyRecord.value.provider]?.label || keyRecord.value.provider)
);

const statusText = computed(() => ({
    valid: '有效', invalid: '无效', rateLimit: '限流', unknown: '未知',
}[keyRecord.value?.status] || '未知'));

const statusClass = computed(() => ({
    valid: 'status-valid', invalid: 'status-invalid',
    rateLimit: 'status-rateLimit', unknown: 'status-unknown',
}[keyRecord.value?.status] || 'status-unknown'));

const filteredModels = computed(() => {
    if (!keyRecord.value) return [];
    if (!modelSearch.value) return keyRecord.value.models;
    const term = modelSearch.value.toLowerCase();
    return keyRecord.value.models.filter(m => m.toLowerCase().includes(term));
});

function formatDate(iso) {
    if (!iso) return '-';
    return new Date(iso).toLocaleString('zh-CN');
}

function formatBalance(balance, currency) {
    if (balance === null || balance === undefined) return '-';
    return currency === 'USD' ? `$${balance.toFixed(4)}` : `${balance} ${currency || ''}`;
}

onMounted(() => {
    if (keyRecord.value) {
        editAlias.value    = keyRecord.value.alias;
        editProvider.value = keyRecord.value.provider;
        keyManager.loadBalanceHistory(keyId.value);
    }
});

async function saveEdit() {
    await keyManager.updateKey(keyId.value, {
        alias:    editAlias.value.trim(),
        provider: editProvider.value,
    });
    isEditing.value = false;
    uiStore.showToast('已保存', 'success');
}

async function testConnection() {
    if (!keyRecord.value || isTesting.value) return;
    isTesting.value = true;
    try {
        const providerConfig = {
            provider: keyRecord.value.provider,
            baseUrl:  keyRecord.value.baseUrl || configStore.providers[keyRecord.value.provider]?.defaultBase || '',
            model:    keyRecord.value.model   || configStore.providers[keyRecord.value.provider]?.defaultModel || '',
            enableStream: false,
            region: configStore.currentRegion,
            validationPrompt: configStore.validationPrompt,
            validationMaxTokens: configStore.validationMaxTokens,
            validationMaxOutputTokens: configStore.validationMaxOutputTokens,
        };
        const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
        const host = window.location.host;
        const result = await new Promise((resolve, reject) => {
            const ws = new WebSocket(`${protocol}//${host}/check`);
            const timeout = setTimeout(() => { ws.close(); reject(new Error('测试超时')); }, 30000);
            ws.onopen = () => ws.send(JSON.stringify({
                command: 'start',
                data: { tokens: [{ token: keyRecord.value.token, order: 0 }], providerConfig, concurrency: 1 },
            }));
            ws.onmessage = (event) => {
                const msg = JSON.parse(event.data);
                if (msg.type === 'result') { clearTimeout(timeout); ws.close(); resolve(msg.data); }
                else if (msg.type === 'error') { clearTimeout(timeout); ws.close(); reject(new Error(msg.message)); }
            };
            ws.onerror = () => { clearTimeout(timeout); reject(new Error('WebSocket 连接失败')); };
            ws.onclose = () => { clearTimeout(timeout); };
        });
        await keyManager.updateKeyFromCheck(keyRecord.value.token, result);
        if (result.isValid) {
            const balMsg = result.balance !== undefined && result.balance !== -1
                ? `，余额: ${formatBalance(result.balance, result.currency)}` : '';
            uiStore.showToast(`测试通过${balMsg}`, 'success');
        } else {
            uiStore.showToast(`测试失败: ${result.message || 'Key 无效'}`, 'error');
        }
    } catch (err) {
        const msg = err.message || '未知错误';
        if (msg.includes('WebSocket') || msg.includes('连接失败') || msg.includes('超时'))
            uiStore.showToast('测试失败：后端服务未运行', 'error', 5000);
        else uiStore.showToast(`测试失败: ${msg}`, 'error');
    } finally { isTesting.value = false; }
}

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
        uiStore.showToast(`获取模型失败: ${err.message || '未知错误'}`, 'error', 5000);
    } finally { isLoadingModels.value = false; }
}

async function copyToken() {
    if (!keyRecord.value) return;
    try {
        await navigator.clipboard.writeText(keyRecord.value.token);
        uiStore.showToast('已复制', 'success', 1500);
    } catch { uiStore.showToast('复制失败', 'error'); }
}

async function handleDelete() {
    const idToDelete = keyId.value;
    if (!idToDelete) return;
    const confirmed = await uiStore.showConfirmation('确定删除这个 Key？');
    if (confirmed) {
        await keyManager.deleteKey(idToDelete);
        uiStore.closeModal();
        uiStore.showToast('已删除', 'success');
    }
}

async function addTag() {
    const tag = newTag.value.trim();
    if (!tag) return;
    await keyManager.addTag(keyId.value, tag);
    newTag.value = '';
}

async function removeTag(tag) {
    await keyManager.removeTag(keyId.value, tag);
}
</script>

<template>
    <div class="detail-modal" v-if="keyRecord">
        <div class="detail-header">
            <div class="detail-header-left">
                <span class="provider-pill">{{ providerLabel }}</span>
                <span class="status-badge" :class="statusClass">{{ statusText }}</span>
            </div>
            <button @click="uiStore.closeModal()" class="detail-close" aria-label="关闭">
                <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75">
                    <path d="M3 3l10 10M13 3L3 13"/>
                </svg>
            </button>
        </div>

        <div class="detail-body">

            <!-- ── Info grid ── -->
            <div class="detail-grid">
                <!-- Token -->
                <div class="detail-field full-width">
                    <span class="field-label">API Key</span>
                    <div class="token-row" @click="copyToken" title="点击复制">
                        <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="1.5">
                            <rect x="1" y="3" width="10" height="7" rx="1.5"/>
                            <path d="M4 3V2a2 2 0 014 0v1"/>
                        </svg>
                        <span class="token-text">{{ keyRecord.token.substring(0, 20) }}...</span>
                        <span class="copy-hint">复制</span>
                    </div>
                </div>

                <!-- Alias -->
                <div class="detail-field">
                    <span class="field-label">别名</span>
                    <span v-if="!isEditing" class="field-value">{{ keyRecord.alias || '—' }}</span>
                    <input v-else v-model="editAlias" class="field-input" placeholder="输入别名" />
                </div>

                <!-- Provider -->
                <div class="detail-field">
                    <span class="field-label">平台</span>
                    <span v-if="!isEditing" class="field-value">{{ providerLabel }}</span>
                    <select v-else v-model="editProvider" class="field-select">
                        <option v-for="(val, key) in configStore.providers" :key="key" :value="key">{{ val.label }}</option>
                    </select>
                </div>

                <!-- Balance -->
                <div class="detail-field" v-if="keyRecord.balance !== null && keyRecord.balance !== undefined">
                    <span class="field-label">余额</span>
                    <span class="field-value balance-value">{{ formatBalance(keyRecord.balance, keyRecord.currency) }}</span>
                </div>

                <!-- Model -->
                <div class="detail-field" v-if="keyRecord.model">
                    <span class="field-label">模型</span>
                    <span class="field-value mono">{{ keyRecord.model }}</span>
                </div>

                <!-- Base URL -->
                <div class="detail-field full-width" v-if="keyRecord.baseUrl">
                    <span class="field-label">Base URL</span>
                    <span class="field-value mono small">{{ keyRecord.baseUrl }}</span>
                </div>

                <!-- Dates -->
                <div class="detail-field">
                    <span class="field-label">创建时间</span>
                    <span class="field-value small">{{ formatDate(keyRecord.createdAt) }}</span>
                </div>
                <div class="detail-field">
                    <span class="field-label">最近检测</span>
                    <span class="field-value small">{{ formatDate(keyRecord.lastChecked) }}</span>
                </div>
            </div>

            <!-- ── Actions ── -->
            <div class="detail-actions">
                <template v-if="!isEditing">
                    <button @click="isEditing = true" class="detail-btn">编辑</button>
                </template>
                <template v-else>
                    <button @click="saveEdit" class="detail-btn primary">保存</button>
                    <button @click="isEditing = false" class="detail-btn">取消</button>
                </template>
                <button @click="testConnection" :disabled="isTesting" class="detail-btn primary">
                    <svg v-if="isTesting" class="spin-icon" width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="1.75">
                        <path d="M6 1a5 5 0 100 10A5 5 0 006 1z" opacity=".25"/>
                        <path d="M11 6a5 5 0 00-5-5"/>
                    </svg>
                    {{ isTesting ? '测试中...' : '测试连接' }}
                </button>
                <button @click="handleFetchModels" :disabled="isLoadingModels" class="detail-btn">
                    {{ isLoadingModels ? '获取中...' : '获取模型' }}
                </button>
                <button @click="handleDelete" class="detail-btn danger">删除</button>
            </div>

            <!-- ── Tags ── -->
            <div class="detail-section">
                <h4 class="section-title">标签</h4>
                <div class="tag-list" v-if="keyRecord.tags.length > 0">
                    <span v-for="tag in keyRecord.tags" :key="tag" class="tag-item">
                        {{ tag }}
                        <button @click="removeTag(tag)" class="tag-remove" aria-label="移除标签">
                            <svg width="10" height="10" viewBox="0 0 10 10" fill="none" stroke="currentColor" stroke-width="1.5">
                                <path d="M2 2l6 6M8 2L2 8"/>
                            </svg>
                        </button>
                    </span>
                </div>
                <div class="tag-add">
                    <input v-model="newTag" class="field-input" placeholder="添加标签..." @keyup.enter="addTag" />
                    <button @click="addTag" class="detail-btn detail-btn-sm">添加</button>
                </div>
            </div>

            <!-- ── Balance history ── -->
            <div class="detail-section" v-if="keyManager.balanceHistory.length > 0">
                <button class="section-toggle" @click="showBalanceHistory = !showBalanceHistory">
                    <span class="section-title">余额历史 ({{ keyManager.balanceHistory.length }})</span>
                    <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="1.5"
                         :style="{ transform: showBalanceHistory ? 'rotate(180deg)' : 'none', transition: 'transform 0.2s' }">
                        <path d="M2 4l4 4 4-4"/>
                    </svg>
                </button>
                <div v-if="showBalanceHistory" class="balance-history">
                    <div v-for="snap in keyManager.balanceHistory" :key="snap.id" class="balance-snap">
                        <span class="snap-time">{{ formatDate(snap.timestamp) }}</span>
                        <span class="snap-balance">{{ formatBalance(snap.balance, snap.currency) }}</span>
                    </div>
                </div>
            </div>

            <!-- ── Models ── -->
            <div class="detail-section" v-if="keyRecord.models.length > 0">
                <h4 class="section-title">可用模型 ({{ keyRecord.models.length }})</h4>
                <input v-model="modelSearch" class="field-input" placeholder="搜索模型..." style="margin-bottom:8px;" />
                <div class="model-list">
                    <div v-for="model in filteredModels" :key="model" class="model-item">{{ model }}</div>
                </div>
            </div>

        </div>
    </div>
</template>

<style scoped>
/* ── Modal shell ── */
.detail-modal {
    background: var(--ds-white);
    border-radius: var(--radius-xl);
    width: 90vw;
    max-width: 520px;
    max-height: 88vh;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    box-shadow: var(--shadow-full-card);
}

/* ── Header ── */
.detail-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 14px 20px;
    box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
    flex-shrink: 0;
}
.detail-header-left {
    display: flex;
    align-items: center;
    gap: 8px;
}
.provider-pill {
    font-size: 12px;
    font-weight: 500;
    padding: 3px 9px;
    border-radius: 9999px;
    background: var(--ds-gray-100);
    color: var(--ds-gray-700);
}
.status-badge {
    font-size: 12px;
    font-weight: 500;
    padding: 3px 9px;
    border-radius: 9999px;
}
.status-valid     { background: #f0fdf4; color: #0a7c42; }
.status-invalid   { background: #fff1f0; color: var(--ds-red-dark); }
.status-rateLimit { background: #fdf2fa; color: var(--ds-pink); }
.status-unknown   { background: var(--ds-gray-100); color: var(--ds-gray-500); }

.detail-close {
    background: transparent;
    border: none;
    cursor: pointer;
    color: var(--text-tertiary);
    padding: 4px;
    border-radius: var(--radius-sm);
    display: flex;
    align-items: center;
    transition: color var(--transition-fast), background var(--transition-fast);
}
.detail-close:hover { background: var(--ds-gray-100); color: var(--text-primary); }

/* ── Body ── */
.detail-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px;
    display: flex;
    flex-direction: column;
    gap: 16px;
}

/* ── Info grid ── */
.detail-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
}
.detail-field {
    display: flex;
    flex-direction: column;
    gap: 4px;
}
.detail-field.full-width { grid-column: 1 / -1; }

.field-label {
    font-size: 11px;
    font-weight: 600;
    color: var(--text-tertiary);
    text-transform: uppercase;
    letter-spacing: 0.04em;
}
.field-value {
    font-size: 13px;
    color: var(--text-primary);
    word-break: break-all;
}
.field-value.mono  { font-family: var(--font-mono); font-size: 12px; }
.field-value.small { font-size: 12px; color: var(--text-secondary); }
.field-value.balance-value { font-weight: 600; font-family: var(--font-mono); }

/* Token row */
.token-row {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 10px;
    background: var(--ds-gray-50);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: background var(--transition-fast);
}
.token-row:hover { background: var(--ds-gray-100); }
.token-text {
    font-family: var(--font-mono);
    font-size: 12px;
    color: var(--text-secondary);
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
.copy-hint {
    font-size: 11px;
    color: var(--text-tertiary);
    flex-shrink: 0;
}

/* Inputs */
.field-input {
    width: 100%;
    height: var(--ctrl-height-md);
    padding: 0 10px;
    border: none;
    border-radius: var(--radius-sm);
    background: var(--ds-white);
    box-shadow: var(--shadow-ring);
    font-size: 13px;
    font-family: var(--font-sans);
    color: var(--text-primary);
    transition: box-shadow var(--transition-fast);
}
.field-input:focus { outline: none; box-shadow: var(--shadow-ring), var(--shadow-focus); }

.field-select {
    width: 100%;
    height: var(--ctrl-height-md);
    padding: 0 28px 0 10px;
    border: none;
    border-radius: var(--radius-sm);
    background: var(--ds-white);
    box-shadow: var(--shadow-ring);
    font-size: 13px;
    font-family: var(--font-sans);
    color: var(--text-primary);
    appearance: none;
    -webkit-appearance: none;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='none' stroke='%238a8a8a' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round' d='M2 4l4 4 4-4'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 8px center;
    transition: box-shadow var(--transition-fast);
}
.field-select:focus { outline: none; box-shadow: var(--shadow-ring), var(--shadow-focus); }

/* ── Actions ── */
.detail-actions {
    display: flex;
    gap: 6px;
    flex-wrap: wrap;
}
.detail-btn {
    height: var(--ctrl-height-md);
    padding: 0 14px;
    border: none;
    border-radius: var(--radius-md);
    background: var(--ds-white);
    color: var(--text-primary);
    box-shadow: var(--shadow-light-ring);
    font-size: var(--ctrl-font-md);
    font-family: var(--font-sans);
    font-weight: 500;
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    white-space: nowrap;
    transition: background var(--transition-fast), box-shadow var(--transition-fast);
}
.detail-btn:hover { background: var(--ds-gray-50); }
.detail-btn.primary { background: var(--ds-gray-1000); color: var(--ds-white); box-shadow: none; }
.detail-btn.primary:hover { background: var(--ds-black); }
.detail-btn.danger  { background: var(--ds-red); color: var(--ds-white); box-shadow: none; }
.detail-btn.danger:hover  { background: var(--ds-red-dark); }
.detail-btn:disabled { opacity: 0.5; cursor: not-allowed; }
.detail-btn-sm { height: var(--ctrl-height-sm); padding: 0 10px; font-size: var(--ctrl-font-sm); }

.spin-icon { animation: spin 0.7s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

/* ── Sections ── */
.detail-section {
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding-top: 16px;
    box-shadow: inset 0 1px 0 0 var(--ds-gray-100);
}
.section-title {
    font-size: 12px;
    font-weight: 600;
    color: var(--text-secondary);
    text-transform: uppercase;
    letter-spacing: 0.04em;
}
.section-toggle {
    background: transparent;
    border: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0;
    width: 100%;
}

/* Tags */
.tag-list { display: flex; gap: 6px; flex-wrap: wrap; }
.tag-item {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 2px 8px;
    background: var(--ds-gray-100);
    border-radius: 9999px;
    font-size: 12px;
    color: var(--text-secondary);
}
.tag-remove {
    background: transparent;
    border: none;
    cursor: pointer;
    color: var(--text-tertiary);
    display: flex;
    align-items: center;
    padding: 1px;
    border-radius: 2px;
    transition: color var(--transition-fast);
}
.tag-remove:hover { color: var(--ds-red); }
.tag-add { display: flex; gap: 6px; align-items: center; }

/* Balance history */
.balance-history { max-height: 180px; overflow-y: auto; }
.balance-snap {
    display: flex;
    justify-content: space-between;
    padding: 6px 0;
    box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
    font-size: 12px;
}
.snap-time    { color: var(--text-secondary); }
.snap-balance { font-weight: 600; font-family: var(--font-mono); color: var(--text-primary); }

/* Model list */
.model-list { max-height: 180px; overflow-y: auto; }
.model-item {
    padding: 5px 0;
    box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
    font-size: 12px;
    font-family: var(--font-mono);
    color: var(--text-secondary);
}
.model-item:last-child { box-shadow: none; }
</style>
