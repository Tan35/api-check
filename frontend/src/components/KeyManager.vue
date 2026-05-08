<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { useKeyManagerStore } from '@/stores/keyManager';
import { useUiStore } from '@/stores/ui';
import { useConfigStore } from '@/stores/config';
import { useCheckerStore } from '@/stores/checker';
import KeyCard from './KeyCard.vue';

const keyManager = useKeyManagerStore();
const uiStore = useUiStore();
const configStore = useConfigStore();
const checkerStore = useCheckerStore();

const addModalOpen = ref(false);
const newKeyText = ref('');
const newKeyAlias = ref('');
const newKeyProvider = ref('openai');
const newKeyBaseUrl = ref('');
const newKeyModel = ref('');
const selectedIds = ref(new Set());
const selectAll = ref(false);
const showImportExport = ref(false);
const importText = ref('');

/**
 * @description 切换平台时自动填充默认 Base URL 和模型。
 */
watch(newKeyProvider, (val) => {
    const p = configStore.providers[val];
    if (p) {
        newKeyBaseUrl.value = p.defaultBase || '';
        newKeyModel.value = p.defaultModel || '';
    }
}, { immediate: true });

/**
 * @description 打开添加模态框时重置表单。
 */
function openAddModal() {
    newKeyText.value = '';
    newKeyAlias.value = '';
    newKeyProvider.value = 'openai';
    addModalOpen.value = true;
}

/**
 * @description 支持的提供商列表。
 */
const providerOptions = computed(() => {
    return Object.entries(configStore.providers).map(([key, val]) => ({
        key,
        label: val.label,
    }));
});

/**
 * @description 当前筛选的提供商选项。
 */
const filterProviderOptions = computed(() => {
    const options = [{ key: null, label: '全部平台' }];
    for (const [key, val] of Object.entries(configStore.providers)) {
        options.push({ key, label: val.label });
    }
    return options;
});

/**
 * @description 状态筛选选项。
 */
const filterStatusOptions = [
    { key: null, label: '全部状态' },
    { key: 'valid', label: '有效' },
    { key: 'invalid', label: '无效' },
    { key: 'rateLimit', label: '限流' },
    { key: 'unknown', label: '未知' },
];

/**
 * @description 排序选项。
 */
const sortOptions = [
    { key: 'createdAt', label: '创建时间' },
    { key: 'lastChecked', label: '最近检测' },
    { key: 'balance', label: '余额' },
    { key: 'provider', label: '平台' },
];

onMounted(() => {
    keyManager.loadKeys();
});

/**
 * @description 添加单个 Key。
 */
async function handleAddKey() {
    const token = newKeyText.value.trim();
    if (!token) {
        uiStore.showToast('请输入 API Key', 'warning');
        return;
    }

    // 检查重复
    const existing = keyManager.keys.find(k => k.token === token);
    if (existing) {
        uiStore.showToast('该 Key 已存在', 'warning');
        return;
    }

    await keyManager.addKey({
        token,
        alias: newKeyAlias.value.trim(),
        provider: newKeyProvider.value,
        baseUrl: newKeyBaseUrl.value.trim() || configStore.providers[newKeyProvider.value]?.defaultBase || '',
        model: newKeyModel.value.trim() || configStore.providers[newKeyProvider.value]?.defaultModel || '',
    });

    newKeyText.value = '';
    newKeyAlias.value = '';
    newKeyBaseUrl.value = '';
    newKeyModel.value = '';
    addModalOpen.value = false;
    uiStore.showToast('Key 已添加', 'success');
}

/**
 * @description 批量导入 Key（每行一个）。
 */
async function handleBatchImport() {
    const lines = importText.value.split('\n').map(l => l.trim()).filter(l => l);
    if (lines.length === 0) {
        uiStore.showToast('没有有效的 Key', 'warning');
        return;
    }

    const records = lines.map(token => ({
        token,
        provider: newKeyProvider.value,
        baseUrl: configStore.providers[newKeyProvider.value]?.defaultBase || '',
        model: configStore.providers[newKeyProvider.value]?.defaultModel || '',
    }));

    const count = await keyManager.addKeysBatch(records);
    importText.value = '';
    showImportExport.value = false;
    uiStore.showToast(`成功导入 ${count} 个 Key`, 'success');
}

/**
 * @description 选中/取消选中单个 Key。
 */
function toggleSelect(id) {
    if (selectedIds.value.has(id)) {
        selectedIds.value.delete(id);
    } else {
        selectedIds.value.add(id);
    }
    selectedIds.value = new Set(selectedIds.value); // 触发响应式更新
    selectAll.value = selectedIds.value.size === keyManager.filteredKeys.length;
}

/**
 * @description 全选/取消全选。
 */
function toggleSelectAll() {
    if (selectAll.value) {
        selectedIds.value = new Set();
    } else {
        selectedIds.value = new Set(keyManager.filteredKeys.map(k => k.id));
    }
    selectAll.value = !selectAll.value;
}

/**
 * @description 批量删除选中的 Key。
 */
async function handleBatchDelete() {
    if (selectedIds.value.size === 0) {
        uiStore.showToast('请先选择要删除的 Key', 'warning');
        return;
    }
    const confirmed = await uiStore.showConfirmation(`确定删除选中的 ${selectedIds.value.size} 个 Key？`);
    if (confirmed) {
        await keyManager.deleteKeysBatch([...selectedIds.value]);
        selectedIds.value = new Set();
        selectAll.value = false;
        uiStore.showToast('已删除', 'success');
    }
}

/**
 * @description 导出 Key 到剪贴板。
 */
async function handleExport() {
    const json = await keyManager.exportKeys();
    try {
        await navigator.clipboard.writeText(json);
        uiStore.showToast('已复制到剪贴板', 'success');
    } catch {
        // 降级：显示在 textarea
        showImportExport.value = true;
        importText.value = json;
        uiStore.showToast('请手动复制下方内容', 'info');
    }
}

/**
 * @description 处理 Key 删除事件。
 */
async function handleDeleteKey(id) {
    await keyManager.deleteKey(id);
    selectedIds.value.delete(id);
    selectedIds.value = new Set(selectedIds.value);
    uiStore.showToast('已删除', 'success');
}

/**
 * @description 打开 Key 详情。
 */
function openDetail(id) {
    keyManager.selectKey(id);
    uiStore.openModal('keyDetail', { keyId: id });
}

/**
 * @description 全选 checkbox 状态同步。
 */
watch(() => keyManager.filteredKeys.length, () => {
    selectAll.value = false;
    selectedIds.value = new Set();
});
</script>

<template>
    <div class="key-manager">
        <!-- 顶部操作栏 -->
        <div class="km-toolbar">
            <div class="km-toolbar-left">
                <button @click="openAddModal" class="km-btn primary">
                    添加 Key
                </button>
                <button @click="showImportExport = !showImportExport" class="km-btn">
                    导入/导出
                </button>
            </div>
            <div class="km-toolbar-right">
                <span class="km-count">
                    {{ keyManager.filteredKeys.length }} / {{ keyManager.keys.length }} 个 Key
                </span>
            </div>
        </div>

        <!-- 搜索和筛选 -->
        <div class="km-filters">
            <input
                v-model="keyManager.searchTerm"
                type="text"
                class="km-search"
                placeholder="搜索 Key、别名、标签..."
            />

            <select v-model="keyManager.filterProvider" class="km-filter-select">
                <option v-for="opt in filterProviderOptions" :key="opt.key" :value="opt.key">
                    {{ opt.label }}
                </option>
            </select>

            <select v-model="keyManager.filterStatus" class="km-filter-select">
                <option v-for="opt in filterStatusOptions" :key="opt.key" :value="opt.key">
                    {{ opt.label }}
                </option>
            </select>

            <select v-model="keyManager.sortBy" class="km-filter-select">
                <option v-for="opt in sortOptions" :key="opt.key" :value="opt.key">
                    排序: {{ opt.label }}
                </option>
            </select>

            <button
                @click="keyManager.sortDir = keyManager.sortDir === 'asc' ? 'desc' : 'asc'"
                class="km-btn small"
            >
                {{ keyManager.sortDir === 'asc' ? '↑' : '↓' }}
            </button>
        </div>

        <!-- 批量操作栏 -->
        <div v-if="selectedIds.size > 0" class="km-batch-actions">
            <label class="km-checkbox-label">
                <input
                    type="checkbox"
                    :checked="selectAll"
                    @change="toggleSelectAll"
                />
                全选
            </label>
            <span class="km-selected-count">已选 {{ selectedIds.size }} 个</span>
            <button @click="handleBatchDelete" class="km-btn danger small">
                批量删除
            </button>
        </div>

        <!-- 导入/导出面板 -->
        <div v-if="showImportExport" class="km-import-export">
            <div class="km-ie-header">
                <span>导入/导出 Keys</span>
                <button @click="showImportExport = false" class="km-btn small">关闭</button>
            </div>
            <textarea
                v-model="importText"
                class="km-ie-textarea"
                placeholder="导入：每行一个 API Key，或粘贴 JSON 数组&#10;导出：点击下方按钮复制所有 Key"
                rows="6"
            ></textarea>
            <div class="km-ie-actions">
                <button @click="handleBatchImport" class="km-btn primary small">
                    导入
                </button>
                <button @click="handleExport" class="km-btn small">
                    导出全部
                </button>
            </div>
        </div>

        <!-- Key 列表 -->
        <div class="km-key-list" v-if="keyManager.keys.length > 0">
            <label class="km-checkbox-label" v-if="keyManager.filteredKeys.length > 0">
                <input
                    type="checkbox"
                    :checked="selectAll"
                    @change="toggleSelectAll"
                />
                全选
            </label>

            <div
                v-for="key in keyManager.filteredKeys"
                :key="key.id"
                class="km-key-item"
                :class="{ selected: selectedIds.has(key.id) }"
            >
                <input
                    type="checkbox"
                    :checked="selectedIds.has(key.id)"
                    @change="toggleSelect(key.id)"
                    class="km-key-checkbox"
                />
                <div class="km-key-card-wrapper" @click="openDetail(key.id)">
                    <KeyCard
                        :key-record="key"
                        :selected="selectedIds.has(key.id)"
                        @delete="handleDeleteKey"
                    />
                </div>
            </div>
        </div>

        <!-- 空状态 -->
        <div v-else class="km-empty">
            <div class="km-empty-text">还没有保存任何 API Key</div>
            <div class="km-empty-hint">点击上方“添加 Key”开始管理你的密钥</div>
        </div>

        <!-- 添加 Key 模态框 -->
        <Teleport to="body">
            <div v-if="addModalOpen" class="km-modal-overlay" @click.self="addModalOpen = false">
                <div class="km-modal t-modal is-open" role="dialog" aria-modal="true">
                    <div class="km-modal-header">
                        <h3>添加 API Key</h3>
                        <button @click="addModalOpen = false" class="km-modal-close">×</button>
                    </div>
                    <div class="km-modal-body">
                        <div class="km-form-group">
                            <label>平台</label>
                            <select v-model="newKeyProvider" class="km-form-select">
                                <option v-for="opt in providerOptions" :key="opt.key" :value="opt.key">
                                    {{ opt.label }}
                                </option>
                            </select>
                        </div>
                        <div class="km-form-group">
                            <label>API Key</label>
                            <input
                                v-model="newKeyText"
                                type="text"
                                class="km-form-input"
                                placeholder="sk-..."
                                autofocus
                            />
                        </div>
                        <div class="km-form-group">
                            <label>Base URL</label>
                            <input
                                v-model="newKeyBaseUrl"
                                type="text"
                                class="km-form-input"
                                placeholder="https://api.example.com/v1"
                            />
                        </div>
                        <div class="km-form-group">
                            <label>模型</label>
                            <input
                                v-model="newKeyModel"
                                type="text"
                                class="km-form-input"
                                placeholder="gpt-4o"
                            />
                        </div>
                        <div class="km-form-group">
                            <label>别名（可选）</label>
                            <input
                                v-model="newKeyAlias"
                                type="text"
                                class="km-form-input"
                                placeholder="例如：我的 OpenAI Key"
                            />
                        </div>
                    </div>
                    <div class="km-modal-footer">
                        <button @click="addModalOpen = false" class="km-btn">取消</button>
                        <button @click="handleAddKey" class="km-btn primary">添加</button>
                    </div>
                </div>
            </div>
        </Teleport>
    </div>
</template>

<style scoped>
    .key-manager {
        padding: 16px;
        height: 100%;
        display: flex;
        flex-direction: column;
        gap: 12px;
    }

    /* 工具栏 */
    .km-toolbar {
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-wrap: wrap;
        gap: 8px;
    }

    .km-toolbar-left {
        display: flex;
        gap: 8px;
    }

    .km-count {
        font-size: 13px;
        color: var(--text-secondary);
        font-variant-numeric: tabular-nums;
    }

    /* 筛选栏 */
    .km-filters {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;
    }

    .km-search {
        flex: 1;
        min-width: 200px;
        height: 36px;
        padding: 0 12px;
        border-radius: var(--radius-md);
        font-size: 14px;
        font-family: var(--font-sans);
    }

    .km-filter-select {
        height: 36px;
        padding: 0 12px;
        border-radius: var(--radius-md);
        font-size: 13px;
        font-family: var(--font-sans);
        background: var(--ds-white);
    }

    /* 批量操作 */
    .km-batch-actions {
        display: flex;
        align-items: center;
        gap: 12px;
        padding: 10px 12px;
        background: var(--ds-gray-50);
        border-radius: var(--radius-md);
        box-shadow: var(--shadow-light-ring);
    }

    .km-selected-count {
        font-size: 13px;
        color: var(--text-secondary);
        font-weight: 600;
    }

    /* 导入导出 */
    .km-import-export {
        padding: 12px;
        background: var(--ds-gray-50);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-light-ring);
    }

    .km-ie-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 8px;
        font-weight: 600;
    }

    .km-ie-textarea {
        width: 100%;
        padding: 8px;
        border-radius: var(--radius-md);
        font-family: var(--font-mono);
        font-size: 12px;
        resize: vertical;
        margin-bottom: 8px;
    }

    .km-ie-actions {
        display: flex;
        gap: 8px;
    }

    /* Key 列表 */
    .km-key-list {
        flex: 1;
        overflow-y: auto;
        padding-right: 4px;
    }

    .km-checkbox-label {
        display: flex;
        align-items: center;
        gap: 6px;
        font-size: 13px;
        color: var(--text-secondary);
        margin-bottom: 0;
        cursor: pointer;
    }

    .km-key-item {
        display: flex;
        align-items: flex-start;
        gap: 8px;
        margin-bottom: 8px;
    }

    .km-key-checkbox {
        margin-top: 14px;
        cursor: pointer;
    }

    .km-key-card-wrapper {
        flex: 1;
    }

    /* 空状态 */
    .km-empty {
        flex: 1;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 6px;
        min-height: 420px;
        color: var(--text-secondary);
    }

    .km-empty-icon {
        font-size: 48px;
        opacity: 0.3;
    }

    .km-empty-text {
        font-size: 15px;
        color: var(--text-secondary);
    }

    .km-empty-hint {
        font-size: 13px;
        color: var(--text-tertiary);
    }

    /* 按钮 */
    .km-btn {
        height: 36px;
        padding: 0 16px;
        box-shadow: var(--shadow-light-ring);
        border-radius: var(--radius-md);
        background: var(--ds-white);
        font-size: 13px;
        font-family: var(--font-sans);
        cursor: pointer;
        transition: all 0.2s;
    }

    .km-btn:hover {
        background: var(--ds-gray-50);
    }

    .km-btn.primary {
        background: var(--ds-gray-1000);
        color: var(--ds-white);
        box-shadow: none;
    }

    .km-btn.primary:hover {
        background: var(--ds-black);
    }

    .km-btn.danger {
        background: var(--ds-red);
        color: var(--ds-white);
        box-shadow: none;
    }

    .km-btn.danger:hover {
        background: var(--ds-red-dark);
    }

    .km-btn.small {
        height: 28px;
        padding: 0 10px;
        font-size: 12px;
    }

    /* 模态框 */
    .km-modal-overlay {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: var(--ds-overlay-backdrop);
        -webkit-backdrop-filter: blur(4px);
        backdrop-filter: blur(4px);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 1000;
    }

    .km-modal {
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        width: 90%;
        max-width: 440px;
        box-shadow: var(--shadow-full-card);
    }

    .km-modal-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 16px 20px;
        box-shadow: inset 0 -1px 0 0 var(--ds-gray-100);
    }

    .km-modal-header h3 {
        margin: 0;
        font-size: 16px;
        font-family: var(--font-serif);
        font-weight: 600;
    }

    .km-modal-close {
        background: transparent;
        font-size: 18px;
        cursor: pointer;
        opacity: 0.5;
    }

    .km-modal-close:hover {
        opacity: 1;
    }

    .km-modal-body {
        padding: 20px;
    }

    .km-form-group {
        margin-bottom: 16px;
    }

    .km-form-group label {
        display: block;
        font-size: 13px;
        font-weight: 600;
        color: var(--text-secondary);
        margin-bottom: 6px;
    }

    .km-form-input,
    .km-form-select {
        width: 100%;
        height: 40px;
        padding: 0 12px;
        box-shadow: var(--shadow-ring);
        border: none;
        border-radius: var(--radius-md);
        font-size: 14px;
        font-family: var(--font-sans);
        background: var(--ds-white);
        transition: box-shadow var(--transition-fast);
    }

    .km-form-input:focus,
    .km-form-select:focus {
        outline: none;
        box-shadow: var(--shadow-ring), var(--shadow-focus);
    }

    .km-modal-footer {
        display: flex;
        justify-content: flex-end;
        gap: 8px;
        padding: 12px 20px;
        box-shadow: inset 0 1px 0 0 var(--ds-gray-100);
    }
</style>
