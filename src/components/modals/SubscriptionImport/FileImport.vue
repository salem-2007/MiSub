<script setup>
    /**
     * 文件上传导入：拖拽 / 点击选择本地文件，读取文本后走后端 /api/parse_subscription。
     * 纯 UI 组件，解析与入库由父组件处理。
     */
    import { ref, computed } from 'vue';
    import { IMPORT_FILE_ACCEPT, readFilesAsText } from '../../../utils/importFile.js';

    const props = defineProps({
        isProcessing: {
            type: Boolean,
            default: false,
        },
        parseStatus: {
            type: String,
            default: '',
        },
        errorMessage: {
            type: String,
            default: '',
        },
        successMessage: {
            type: String,
            default: '',
        },
    });

    const emit = defineEmits(['files']);

    const isDragging = ref(false);
    const fileInput = ref(null);

    const acceptAttr = IMPORT_FILE_ACCEPT;

    const statusText = computed(() => props.parseStatus || '');

    function openPicker() {
        if (props.isProcessing) return;
        fileInput.value?.click();
    }

    function onInputChange(event) {
        const files = event.target.files;
        if (files && files.length) emit('files', files);
        // 允许重复选择同一文件
        event.target.value = '';
    }

    function onDrop(event) {
        isDragging.value = false;
        if (props.isProcessing) return;
        const files = event.dataTransfer?.files;
        if (files && files.length) emit('files', files);
    }

    function onDragOver() {
        if (!props.isProcessing) isDragging.value = true;
    }

    function onDragLeave() {
        isDragging.value = false;
    }

    defineExpose({ readFilesAsText });
</script>

<template>
    <div class="space-y-4">
        <div
            class="relative border-2 border-dashed misub-radius-lg p-6 text-center transition-all duration-200 cursor-pointer"
            :class="[
                isDragging
                    ? 'border-primary-500 bg-primary-500/5'
                    : 'border-gray-300 dark:border-gray-600 hover:border-primary-400 dark:hover:border-primary-500',
                isProcessing ? 'opacity-60 pointer-events-none' : '',
            ]"
            @click="openPicker"
            @drop.prevent="onDrop"
            @dragover.prevent="onDragOver"
            @dragleave.prevent="onDragLeave"
        >
            <input
                ref="fileInput"
                type="file"
                multiple
                class="hidden"
                :accept="acceptAttr"
                @change="onInputChange"
            />
            <div class="flex flex-col items-center gap-2">
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    class="h-9 w-9 text-primary-500"
                    fill="none"
                    viewBox="0 0 24 24"
                    stroke="currentColor"
                    stroke-width="1.6"
                >
                    <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        d="M12 16.5V9.75m0 0l3 3m-3-3l-3 3M6.75 19.5a4.5 4.5 0 01-1.41-8.775 5.25 5.25 0 0110.233-2.33 3 3 0 013.758 3.848A3.752 3.752 0 0118 19.5H6.75z"
                    />
                </svg>
                <p class="text-sm font-medium text-gray-700 dark:text-gray-200">
                    点击选择文件，或将文件拖拽到此处
                </p>
                <p class="text-xs text-gray-500 dark:text-gray-400">
                    支持 .txt / .yaml / .yml / .json / .conf，可多选；自动识别其中的节点
                </p>
            </div>
        </div>

        <div
            class="bg-blue-50 dark:bg-blue-900/20 border border-blue-200 dark:border-blue-800 misub-radius-md p-3"
        >
            <h4 class="text-sm font-medium text-blue-800 dark:text-blue-200 mb-2">
                自动识别的文件内容：
            </h4>
            <ul class="text-xs text-blue-700 dark:text-blue-300 space-y-1">
                <li>• <strong>纯文本节点</strong>：每行一个 ss:// vmess:// vless:// trojan:// 等链接</li>
                <li>• <strong>Base64 订阅</strong>：v2rayN / Shadowrocket 等导出的 .txt</li>
                <li>• <strong>Clash / Mihomo 配置</strong>：YAML 中的 proxies 列表</li>
                <li>• <strong>Surge / Loon / QuantumultX</strong>：代理段配置</li>
                <li>• <strong>JSON</strong>：Clash Meta 配置或节点数组</li>
            </ul>
        </div>

        <div
            v-if="statusText"
            class="flex items-center gap-2 text-sm text-gray-600 dark:text-gray-300"
        >
            <svg class="h-4 w-4 animate-spin text-primary-500" viewBox="0 0 24 24" fill="none">
                <circle
                    class="opacity-25"
                    cx="12"
                    cy="12"
                    r="10"
                    stroke="currentColor"
                    stroke-width="4"
                ></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v4a4 4 0 00-4 4H4z"></path>
            </svg>
            <span>{{ statusText }}</span>
        </div>

        <p
            v-if="errorMessage"
            class="text-sm text-red-600 dark:text-red-400 bg-red-50 dark:bg-red-900/20 misub-radius-md p-2"
        >
            {{ errorMessage }}
        </p>
        <p
            v-if="successMessage"
            class="text-sm text-green-600 dark:text-green-400 bg-green-50 dark:bg-green-900/20 misub-radius-md p-2"
        >
            {{ successMessage }}
        </p>
    </div>
</template>
