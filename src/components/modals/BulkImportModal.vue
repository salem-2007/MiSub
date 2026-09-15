<script setup>
    import { ref, watch } from 'vue';
    import Modal from '../forms/Modal.vue';
    import GroupSelector from '../ui/GroupSelector.vue'; // Added
    import FileImport from './SubscriptionImport/FileImport.vue';
    import { useManualNodes } from '../../composables/useManualNodes.js';
    import { useDataStore } from '../../stores/useDataStore.js';
    import { useToastStore } from '../../stores/toast.js';
    import { useI18n } from '@/i18n/index.js';
    import { api } from '../../lib/http.js';
    import { generateNodeId } from '../../utils/id.js';
    import { readFilesAsText } from '../../utils/importFile.js';

    const { t } = useI18n();
    const { manualNodeGroups } = useManualNodes(useDataStore().markDirty);
    const toastStore = useToastStore();

    const props = defineProps({
        show: Boolean,
    });

    const emit = defineEmits(['update:show', 'import', 'files-imported']);

    const activeTab = ref('text'); // 'text' | 'file'
    const importText = ref('');
    const selectedGroup = ref('');
    const urlFocused = ref(false);

    const isProcessing = ref(false);
    const parseStatus = ref('');
    const errorMessage = ref('');
    const successMessage = ref('');

    watch(
        () => props.show,
        (visible) => {
            if (!visible) {
                importText.value = '';
                selectedGroup.value = '';
                activeTab.value = 'text';
                isProcessing.value = false;
                parseStatus.value = '';
                errorMessage.value = '';
                successMessage.value = '';
            }
        }
    );

    const handleConfirm = () => {
        if (activeTab.value !== 'text') return;
        emit('import', importText.value, selectedGroup.value); // group passed as second arg
        importText.value = '';
        selectedGroup.value = '';
        setTimeout(() => {
            emit('update:show', false);
        }, 1000);
    };

    /**
     * 文件导入：读取文件 → 后端解析 → 交回父级批量写入。
     */
    const handleFiles = async (fileList) => {
        isProcessing.value = true;
        errorMessage.value = '';
        successMessage.value = '';
        parseStatus.value = '正在读取文件...';

        try {
            const { text, fileCount } = await readFilesAsText(fileList);
            if (!text.trim()) {
                throw new Error('文件内容为空，未找到可导入的内容。');
            }

            parseStatus.value = `正在解析 ${fileCount} 个文件的内容...`;
            const parseResult = await api.post('/api/parse_subscription', { content: text });

            if (!parseResult.success) {
                throw new Error(parseResult.error || '解析文件失败');
            }

            const nodes = (parseResult.data?.nodes || []).map((node) => ({
                id: generateNodeId(),
                name: node.name || 'Unknown',
                url: node.url,
                enabled: true,
                protocol: node.protocol || 'unknown',
                source: 'import',
                group: selectedGroup.value || null,
            }));

            if (nodes.length === 0) {
                parseStatus.value = '';
                throw new Error(
                    '未能从文件中解析出任何有效节点。请确认文件包含受支持的节点链接、Clash/Surge 配置或 Base64 订阅内容。'
                );
            }

            const uniqueNodes = nodes.filter(
                (node, index, self) => index === self.findIndex((n) => n.url === node.url)
            );
            const duplicateCount = nodes.length - uniqueNodes.length;

            parseStatus.value = '';
            emit('files-imported', uniqueNodes, selectedGroup.value);

            const msg =
                `成功添加 ${uniqueNodes.length} 个节点` +
                (selectedGroup.value ? ` 到分组 "${selectedGroup.value}"` : '') +
                (duplicateCount > 0 ? `（去重 ${duplicateCount} 个重复节点）` : '');
            successMessage.value = msg;
            toastStore.showToast(msg, 'success');

            setTimeout(() => {
                emit('update:show', false);
            }, 2000);
        } catch (error) {
            console.error('批量文件导入失败:', error);
            parseStatus.value = '';
            errorMessage.value = error.message || '导入失败';
            toastStore.showToast(`导入失败: ${error.message}`, 'error');
        } finally {
            isProcessing.value = false;
        }
    };
</script>

<template>
    <Modal
        :show="show"
        @update:show="emit('update:show', $event)"
        @confirm="handleConfirm"
        size="2xl"
    >
        <template #title>
            <div class="flex items-center gap-3">
                <div class="p-2 misub-radius-lg bg-indigo-500/10">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6 text-indigo-500"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                        stroke-width="2"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10"
                        />
                    </svg>
                </div>
                <div>
                    <h3 class="text-lg font-bold text-gray-800 dark:text-white">
                        {{ t('bulkImport.title') }}
                    </h3>
                    <p class="mt-0.5 text-sm text-gray-500 dark:text-gray-400">
                        {{ t('bulkImport.description') }}
                    </p>
                </div>
            </div>
        </template>

        <template #footer>
            <button
                @click="emit('update:show', false)"
                class="px-4 py-2 bg-gray-200 hover:bg-gray-300 dark:bg-gray-700 dark:hover:bg-gray-600 text-gray-800 dark:text-gray-200 font-semibold text-sm misub-radius-lg transition-colors"
            >
                {{ t('actions.cancel') }}
            </button>
            <button
                v-if="activeTab === 'text'"
                @click="handleConfirm"
                class="px-4 py-2 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold text-sm misub-radius-lg transition-colors"
            >
                {{ t('actions.confirm') }}
            </button>
        </template>

        <template #body>
            <div class="space-y-6">
                <!-- 来源切换 -->
                <div class="grid grid-cols-2 gap-1 p-1 bg-gray-100 dark:bg-white/5 misub-radius-lg">
                    <button
                        type="button"
                        class="py-1.5 text-sm font-medium misub-radius-md transition-colors"
                        :class="
                            activeTab === 'text'
                                ? 'bg-white dark:bg-gray-800 text-primary-600 dark:text-primary-400 shadow-xs'
                                : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200'
                        "
                        @click="activeTab = 'text'"
                    >
                        粘贴文本
                    </button>
                    <button
                        type="button"
                        class="py-1.5 text-sm font-medium misub-radius-md transition-colors"
                        :class="
                            activeTab === 'file'
                                ? 'bg-white dark:bg-gray-800 text-primary-600 dark:text-primary-400 shadow-xs'
                                : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200'
                        "
                        @click="activeTab = 'file'"
                    >
                        上传文件
                    </button>
                </div>

                <!-- Group Selector -->
                <div class="relative">
                    <div class="flex flex-col gap-1.5">
                        <label
                            for="import-group"
                            class="text-xs font-semibold text-gray-500 dark:text-gray-400 mb-1.5 ml-1"
                        >
                            {{ t('bulkImport.groupLabel') }}
                        </label>
                        <GroupSelector
                            v-model="selectedGroup"
                            :groups="manualNodeGroups"
                            :placeholder="t('ui.groupSelector.placeholder')"
                        />
                    </div>
                </div>

                <div v-if="activeTab === 'text'" class="relative group">
                    <div
                        class="relative border misub-radius-lg transition-all duration-300 overflow-hidden bg-gray-50 dark:bg-black/20 border-gray-200 dark:border-white/10"
                        :class="[
                            urlFocused
                                ? 'ring-2 ring-primary-500/50 border-primary-500 dark:border-primary-500'
                                : 'hover:border-gray-300 dark:hover:border-white/20',
                        ]"
                    >
                        <div class="flex h-full">
                            <div
                                class="py-3 pl-3 flex items-start text-gray-400 group-focus-within:text-primary-500 transition-colors pointer-events-none"
                            >
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    class="h-5 w-5 mt-0.5"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                    stroke="currentColor"
                                    stroke-width="2"
                                >
                                    <path
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        d="M9 13h6m-3-3v6m5 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
                                    />
                                </svg>
                            </div>
                            <textarea
                                v-model="importText"
                                rows="8"
                                @focus="urlFocused = true"
                                @blur="urlFocused = false"
                                class="flex-1 w-full bg-transparent border-0 focus:ring-0 dark:text-white placeholder-gray-400 text-sm font-mono resize-none py-3 px-3 min-h-[160px]"
                                placeholder="http://...&#10;https://...&#10;vmess://...&#10;vless://...&#10;trojan://..."
                            ></textarea>
                        </div>
                        <!-- Focus Glow -->
                        <div
                            class="absolute inset-0 misub-radius-lg pointer-events-none transition-opacity duration-300 opacity-0 group-focus-within:opacity-100 ring-1 ring-primary-500/20"
                        ></div>
                    </div>
                    <p class="text-xs text-gray-500 dark:text-gray-400">
                        {{ t('bulkImport.helper') }}
                    </p>
                </div>

                <FileImport
                    v-else
                    :is-processing="isProcessing"
                    :parse-status="parseStatus"
                    :error-message="errorMessage"
                    :success-message="successMessage"
                    @files="handleFiles"
                />
            </div>
        </template>
    </Modal>
</template>
