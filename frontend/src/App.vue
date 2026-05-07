<script setup>
import { onMounted, onBeforeUnmount, watch, ref, computed } from 'vue';
import { useUiStore } from '@/stores/ui';
import { useCheckerStore } from '@/stores/checker';
import { useKeyManagerStore } from '@/stores/keyManager';
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
const scrollPosition = ref(0);

/**
 * @description 动态计算当前年份，用于 Footer 版权信息。
 */
const currentYear = computed(() => new Date().getFullYear());

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
        <div class="header">
            <h1>API KEY 检测工具</h1>
            <div class="view-tabs">
                <button
                    :class="['view-tab', { active: !keyManager.showManager }]"
                    @click="keyManager.showManager = false"
                >
                    🔍 检测工具
                </button>
                <button
                    :class="['view-tab', { active: keyManager.showManager }]"
                    @click="keyManager.showManager = true"
                >
                    🔑 我的 Key ({{ keyManager.keys.length }})
                </button>
            </div>
        </div>

        <!-- 检测工具视图 -->
        <div v-if="!keyManager.showManager" class="main-grid">
            <div class="main-content">
                <div class="input-section">
                    <ProviderSelector />
                    <ApiConfig />
                </div>
                <div class="input-section">
                    <KeyInput />
                </div>
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

        <!-- Key 管理视图 -->
        <div v-else class="manager-grid">
            <KeyManager />
        </div>

        <div class="footer">
            <p>© {{ currentYear }} LLM API KEY 检测工具 | <a href="https://github.com/ssfun/llm-api-key-checker" target="_blank"
                    rel="noopener noreferrer">@SFUN</a></p>
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

    /* 视图切换 Tab — Vercel pill style */
    .view-tabs {
        display: flex;
        gap: 2px;
        background: var(--ds-gray-50);
        border-radius: 64px;
        padding: 2px;
        margin-top: 16px;
        box-shadow: var(--shadow-light-ring);
    }

    .view-tab {
        padding: 6px 20px;
        border: none;
        border-radius: 64px;
        background: transparent;
        font-size: 13px;
        font-family: var(--font-sans);
        cursor: pointer;
        transition: all 0.15s ease;
        color: var(--ds-gray-500);
    }

    .view-tab:hover {
        color: var(--ds-gray-900);
    }

    .view-tab.active {
        background: var(--ds-white);
        color: var(--ds-gray-900);
        font-weight: 500;
        box-shadow: var(--shadow-ring);
    }

    /* Key 管理视图 */
    .manager-grid {
        flex: 1;
        min-height: 0;
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-card);
        overflow: hidden;
    }

    /* 结果面板 */
    .results-wrapper {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: var(--ds-white);
        border-radius: var(--radius-lg);
        box-shadow: var(--shadow-card);
        overflow: hidden;
        display: flex;
        flex-direction: column;
    }

    .results-panels {
        padding: 8px;
        flex: 1;
        display: flex;
        flex-direction: column;
        min-height: 0;
        overflow: hidden;
    }

    @media (max-width: 1024px) {
        .results-wrapper {
            position: static;
            height: 500px;
        }
    }

    @media (max-width: 768px) {
        .results-wrapper {
            height: 450px;
        }
    }

    @media (max-width: 480px) {
        .results-wrapper {
            height: 400px;
        }
    }
</style>
