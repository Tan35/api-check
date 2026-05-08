<template>
    <div class="actions-container">
        <div class="button-group-wrapper">
            <TransitionGroup name="button-fade">
                <button v-if="!checkerStore.isChecking" key="start" id="checkButton" class="button primary"
                    @click="checkerStore.startCheck">
                    开始检测
                </button>
                <template v-else>
                    <button key="stop" class="button stop" @click="checkerStore.stopCheck">
                        停止
                    </button>
                    <button v-if="!checkerStore.isPaused" key="pause" class="button pause" @click="checkerStore.pauseCheck">
                        暂停
                    </button>
                    <button v-else key="resume" class="button resume" @click="checkerStore.resumeCheck">
                        继续
                    </button>
                </template>
            </TransitionGroup>
        </div>
        <div id="progress-container" v-show="checkerStore.isChecking">
            <div class="progress-bar-wrapper"
                role="progressbar"
                :aria-valuenow="checkerStore.progress"
                aria-valuemin="0"
                aria-valuemax="100"
                :aria-label="`检测进度: ${checkerStore.progress}%`">
                <div id="progressBar" :style="{ width: checkerStore.progress + '%' }"></div>
            </div>
            <span id="progressText" aria-live="polite">{{ checkerStore.completedCount }} / {{ checkerStore.totalTasks }} ({{ checkerStore.progress }}%)</span>
        </div>
    </div>
</template>

<script setup>
import { useCheckerStore } from '@/stores/checker';

const checkerStore = useCheckerStore();
</script>

<style scoped>
    .actions-container {
        display: flex;
        flex-direction: column;
        gap: 16px;
    }

    .button-group-wrapper {
        display: flex;
        gap: 8px;
        width: 100%;
        position: relative;
        min-height: 40px;
    }

    .button {
        flex: 1;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .button.stop {
        background: var(--ds-ship-red);
        color: var(--ds-white);
    }

    .button.pause {
        background: var(--ds-gray-500);
        color: var(--ds-white);
    }

    .button.resume {
        background: var(--ds-console-blue);
        color: var(--ds-white);
    }

    .button-fade-enter-active,
    .button-fade-leave-active {
        transition: opacity 0.15s ease;
    }

    .button-fade-enter-from,
    .button-fade-leave-to {
        opacity: 0;
    }

    .button-fade-leave-active {
        position: absolute;
        width: calc(50% - 4px);
    }

    #progress-container {
        display: flex;
        flex-direction: column;
        gap: 8px;
        align-items: center;
        width: 100%;
    }

    .progress-bar-wrapper {
        width: 100%;
        height: 4px;
        background-color: var(--ds-gray-100);
        border-radius: 2px;
        overflow: hidden;
    }

    #progressBar {
        width: 0%;
        height: 100%;
        background: var(--ds-gray-900);
        border-radius: 2px;
        transition: width 0.3s ease;
    }

    #progressText {
        font-size: 12px;
        font-weight: 500;
        font-family: var(--font-sans);
        color: var(--text-tertiary);
    }
</style>
