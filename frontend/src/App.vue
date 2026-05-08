<script setup>
import { onMounted, onBeforeUnmount, watch, ref, computed } from 'vue';
import { useUiStore } from '@/stores/ui';
import { useCheckerStore } from '@/stores/checker';
import { useKeyManagerStore } from '@/stores/keyManager';
import { useConfigStore } from '@/stores/config';
import { RESULT_TAB_CONFIG } from '@/constants';

// 导入组件
import ProviderSelector from './components/ProviderSelector.vue';
import ApiConfig from './components/ApiConfig.vue';
import KeyInput from './components/KeyInput.vue';
import ActionButtons from './components/ActionButtons.vue';
import ResultsTabs from './components/ResultsTabs.vue';
import ResultPanel from './components/ResultPanel.vue';
import ToastContainer from './components/ToastContainer.vue';
import ModalContainer from './components/ModalContainer.vue';
import KeyManager from './components/KeyManager.vue';

/**
 * @description 结果标签页的配置数组。
 */
const resultTabsConfig = RESULT_TAB_CONFIG;

const uiStore = useUiStore();
const checkerStore = useCheckerStore();
const keyManager = useKeyManagerStore();
const configStore = useConfigStore();
const scrollPosition = ref(0);

/**
 * @description 动态计算当前年份，用于 Footer 版权信息。
 */
const currentYear = computed(() => new Date().getFullYear());

const currentProviderLabel = computed(() => {
    return configStore.providers[configStore.currentProvider]?.label || configStore.currentProvider;
});

const currentRegionLabel = computed(() => {
    return configStore.regions[configStore.currentRegion] || configStore.currentRegion;
});

const statusLabel = computed(() => {
    if (checkerStore.isPaused) return '已暂停';
    if (checkerStore.isChecking) return `检测中 ${checkerStore.progress}%`;
    return '待命';
});

/**
 * @description 监听 checkerStore 的 lastStatusMessage 变化，并触发 UI Toast 提示。
 */
watch(() => checkerStore.lastStatusMessage, (newMessage) => {
    if (newMessage && newMessage.text) {
        uiStore.showToast(newMessage.text, newMessage.type, newMessage.duration);
    }
}, { deep: true });

/**
 * @description 侦听弹窗状态，以实现可靠的滚动锁定。
 * 当模态框激活时，锁定页面滚动；模态框关闭时，恢复滚动。
 */
watch(() => uiStore.isModalActive, (isActive) => {
    const body = document.body;
    if (isActive) {
        scrollPosition.value = window.scrollY;
        body.style.position = 'fixed';
        body.style.top = `-${scrollPosition.value}px`;
        body.style.width = '100%';
        body.classList.add('modal-open');
    } else {
        body.style.position = '';
        body.style.top = '';
        body.style.width = '';
        body.classList.remove('modal-open');
        window.scrollTo(0, scrollPosition.value);
    }
});

/**
 * @description 处理 ESC 键按下事件，用于关闭模态框。
 * @param {KeyboardEvent} e - 键盘事件对象。
 */
const handleEscKey = (e) => {
    if (e.key !== 'Escape' || !uiStore.activeModal) return;
    if (uiStore.activeModal === 'modelSelector' && uiStore.modelSearch) {
        uiStore.modelSearch = '';
    } else {
        uiStore.closeModal();
    }
};

/**
 * @description 组件挂载时添加键盘事件监听器并初始化会话。
 */
onMounted(() => {
    checkerStore.initSession();
    keyManager.loadKeys();
    document.addEventListener('keydown', handleEscKey);
});

/**
 * @description 组件卸载前移除键盘事件监听器。
 */
onBeforeUnmount(() => {
    document.removeEventListener('keydown', handleEscKey);
    // 完整恢复 body 样式，防止残留
    const body = document.body;
    body.style.position = '';
    body.style.top = '';
    body.style.width = '';
    body.classList.remove('modal-open');
});
</script>

<template>
    <div class="page-wrapper">
        <header class="topbar">
            <div class="brand-lockup">
                <span class="brand-mark" aria-hidden="true"></span>
                <div class="brand-text">
                    <h1>API Check</h1>
                    <p class="brand-subtitle">{{ currentProviderLabel }} · {{ currentRegionLabel }}</p>
                </div>
            </div>

            <div class="topbar-actions">
                <div class="run-status" :class="{ active: checkerStore.isChecking && !checkerStore.isPaused }">
                    <span class="run-status-dot" aria-hidden="true"></span>
                    <span>{{ statusLabel }}</span>
                </div>

                <div class="view-tabs" role="tablist" aria-label="视图切换">
                <button
                    :class="['view-tab', { active: !keyManager.showManager }]"
                    @click="keyManager.showManager = false"
                    role="tab"
                    :aria-selected="!keyManager.showManager"
                >
                    检测台
                </button>
                <button
                    :class="['view-tab', { active: keyManager.showManager }]"
                    @click="keyManager.showManager = true"
                    role="tab"
                    :aria-selected="keyManager.showManager"
                >
                    Key 库 {{ keyManager.keys.length }}
                </button>
                </div>
            </div>
        </header>

        <main class="workspace">
            <div v-if="!keyManager.showManager" class="main-grid">
                <div class="main-content">
                    <section class="input-section">
                        <ProviderSelector />
                        <ApiConfig />
                    </section>
                    <section class="input-section">
                        <KeyInput />
                    </section>
                    <ActionButtons />
                </div>
                <div class="sidebar-content">
                    <div class="results-wrapper">
                        <ResultsTabs />
                        <div class="results-panels">
                            <ResultPanel v-for="tab in resultTabsConfig" :key="tab.id" :category="tab.id" :title="tab.name"
                                :sortable="tab.sortable" />
                        </div>
                    </div>
                </div>
            </div>

            <div v-else class="manager-grid">
                <KeyManager />
            </div>
        </main>

        <div class="footer">
            <p>© {{ currentYear }} API Check · <a href="https://github.com/Tan35/api-check" target="_blank"
                    rel="noopener noreferrer">GitHub</a></p>
        </div>
        <ToastContainer />
        <ModalContainer />
    </div>
</template>

<style>
    /* 防止 Vue 渲染时闪烁未编译内容 */
    [v-cloak] {
        display: none;
    }

    .results-wrapper {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        overflow: hidden;
        display: flex;
        flex-direction: column;
    }

    .results-panels {
        padding: 10px;
        flex: 1;
        display: flex;
        flex-direction: column;
        min-height: 0;
        overflow: hidden;
    }

    @media (max-width: 1024px) {
        .results-wrapper {
            position: static;
            height: 560px;
        }
    }

    @media (max-width: 768px) {
        .results-wrapper {
            height: 520px;
        }
    }

    @media (max-width: 480px) {
        .results-wrapper {
            height: 400px;
        }
    }
</style>
