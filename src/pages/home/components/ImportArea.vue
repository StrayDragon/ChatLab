<script setup lang="ts">
import { FileDropZone } from '@/components/UI'
import { storeToRefs } from 'pinia'
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import { useI18n } from 'vue-i18n'
import { useSessionStore, type BatchFileInfo } from '@/stores/session'

const { t } = useI18n()
const sessionStore = useSessionStore()
const { isImporting, importProgress, isBatchImporting, batchFiles, batchImportResult } = storeToRefs(sessionStore)

const importError = ref<string | null>(null)
const diagnosisSuggestion = ref<string | null>(null)
const hasImportLog = ref(false)

const router = useRouter()

// 计算是否正在导入（单文件或批量）
const isAnyImporting = computed(() => isImporting.value || isBatchImporting.value)

// 计算批量导入进度
const batchProgress = computed(() => {
  if (!isBatchImporting.value || batchFiles.value.length === 0) return null

  const completed = batchFiles.value.filter(
    (f) => f.status === 'success' || f.status === 'failed' || f.status === 'cancelled'
  ).length
  const current = batchFiles.value.findIndex((f) => f.status === 'importing')

  return {
    completed,
    total: batchFiles.value.length,
    currentIndex: current,
  }
})

/**
 * Translate error key to localized message
 * Error keys follow format: 'error.{error_name}'
 * Example: 'error.unrecognized_format' -> t('home.import.errors.unrecognized_format')
 */
function translateError(error: string): string {
  if (error.startsWith('error.')) {
    const key = `home.import.errors.${error.slice(6)}` // Remove 'error.' prefix
    const translated = t(key)
    return translated !== key ? translated : error
  }
  // Unknown error format, return as-is
  return error
}

// 根据会话类型导航到对应页面
async function navigateToSession(sessionId: string) {
  const session = await window.chatApi.getSession(sessionId)
  if (session) {
    const routeName = session.type === 'private' ? 'private-chat' : 'group-chat'
    router.push({ name: routeName, params: { id: sessionId } })
  }
}

// 检查是否有导入日志
async function checkImportLog() {
  const result = await window.cacheApi.getLatestImportLog()
  hasImportLog.value = result.success && !!result.path
}

// 处理文件选择（点击选择）- 支持多选
async function handleClickImport() {
  importError.value = null
  diagnosisSuggestion.value = null
  hasImportLog.value = false

  // 使用系统对话框选择多个文件
  const result = await window.api.dialog.showOpenDialog({
    title: t('home.import.selectFiles'),
    properties: ['openFile', 'multiSelections'],
    filters: [
      { name: t('home.import.chatRecords'), extensions: ['json', 'jsonl', 'txt'] },
      { name: t('home.import.allFiles'), extensions: ['*'] },
    ],
  })

  if (result.canceled || result.filePaths.length === 0) {
    return
  }

  await processFilePaths(result.filePaths)
}

// 处理文件拖拽 - 支持多选
async function handleFileDrop({ paths }: { files: File[]; paths: string[] }) {
  if (paths.length === 0) {
    importError.value = t('home.import.cannotReadPath')
    return
  }

  importError.value = null
  diagnosisSuggestion.value = null
  hasImportLog.value = false

  await processFilePaths(paths)
}

// 统一处理文件路径（单文件或多文件）
async function processFilePaths(paths: string[]) {
  if (paths.length === 1) {
    // 单文件导入 - 使用原有逻辑
    const result = await sessionStore.importFileFromPath(paths[0])
    if (!result.success && result.error) {
      importError.value = translateError(result.error)
      if (result.diagnosisSuggestion) {
        diagnosisSuggestion.value = result.diagnosisSuggestion
      }
      await checkImportLog()
    } else if (result.success && sessionStore.currentSessionId) {
      await navigateToSession(sessionStore.currentSessionId)
    }
  } else {
    // 多文件批量导入
    await sessionStore.importFilesFromPaths(paths)
    // 批量导入完成后不自动跳转，显示结果摘要
  }
}

// 取消批量导入
function handleCancelBatchImport() {
  sessionStore.cancelBatchImport()
}

// 关闭结果摘要
function handleCloseResult() {
  sessionStore.clearBatchImportResult()
}

// 跳转到指定会话
async function handleGoToSession(sessionId: string) {
  sessionStore.clearBatchImportResult()
  await navigateToSession(sessionId)
}

function openTutorial() {
  const { locale } = useI18n()
  const langPath = locale.value === 'zh-CN' ? '/cn' : ''
  window.open(`https://chatlab.fun${langPath}/usage/how-to-export.html?utm_source=app`, '_blank')
}

// 打开最新的导入日志文件
async function openLatestImportLog() {
  const result = await window.cacheApi.getLatestImportLog()
  if (result.success && result.path) {
    await window.cacheApi.showInFolder(result.path)
  } else {
    // 没有日志文件时，打开日志目录
    await window.cacheApi.openDir('logs')
  }
}

function getProgressText(): string {
  if (!importProgress.value) return ''
  switch (importProgress.value.stage) {
    case 'detecting':
      return t('home.import.progress.detecting')
    case 'reading':
      return t('home.import.progress.reading')
    case 'parsing':
      return t('home.import.progress.parsing')
    case 'saving':
      return t('home.import.progress.saving')
    case 'done':
      return t('home.import.progress.done')
    case 'error':
      return t('home.import.progress.error')
    default:
      return ''
  }
}

function getProgressDetail(): string {
  if (!importProgress.value) return ''
  const { messagesProcessed, totalBytes, bytesRead } = importProgress.value

  if (messagesProcessed && messagesProcessed > 0) {
    return t('home.import.processed', { count: messagesProcessed.toLocaleString() })
  }

  if (totalBytes && bytesRead) {
    const percent = Math.round((bytesRead / totalBytes) * 100)
    const mbRead = (bytesRead / 1024 / 1024).toFixed(1)
    const mbTotal = (totalBytes / 1024 / 1024).toFixed(1)
    return `${mbRead} MB / ${mbTotal} MB (${percent}%)`
  }

  return importProgress.value.message || ''
}

// 获取文件状态图标
function getFileStatusIcon(file: BatchFileInfo): string {
  switch (file.status) {
    case 'pending':
      return 'i-heroicons-clock'
    case 'importing':
      return 'i-heroicons-arrow-path'
    case 'success':
      return 'i-heroicons-check-circle'
    case 'failed':
      return 'i-heroicons-x-circle'
    case 'cancelled':
      return 'i-heroicons-minus-circle'
    default:
      return 'i-heroicons-question-mark-circle'
  }
}

// 获取文件状态颜色类名
function getFileStatusClass(file: BatchFileInfo): string {
  switch (file.status) {
    case 'pending':
      return 'text-gray-400'
    case 'importing':
      return 'text-pink-500 animate-spin'
    case 'success':
      return 'text-green-500'
    case 'failed':
      return 'text-red-500'
    case 'cancelled':
      return 'text-gray-400'
    default:
      return 'text-gray-400'
  }
}

// 获取文件进度描述
function getFileProgressText(file: BatchFileInfo): string {
  if (file.status === 'pending') return t('home.import.batch.waiting')
  if (file.status === 'cancelled') return t('home.import.batch.skipped')
  if (file.status === 'success') return t('home.import.batch.success')
  if (file.status === 'failed') return translateError(file.error || 'error.import_failed')

  // importing 状态
  if (file.progress) {
    switch (file.progress.stage) {
      case 'detecting':
        return t('home.import.progress.detecting')
      case 'reading':
        return t('home.import.progress.reading')
      case 'parsing':
        if (file.progress.messagesProcessed) {
          return t('home.import.processed', { count: file.progress.messagesProcessed.toLocaleString() })
        }
        return t('home.import.progress.parsing')
      case 'saving':
        return t('home.import.progress.saving')
      default:
        return ''
    }
  }
  return ''
}
</script>

<template>
  <div class="flex flex-col items-center space-y-6">
    <!-- 批量导入进度（导入中） -->
    <div
      v-if="isBatchImporting && batchFiles.length > 0"
      class="w-full max-w-4xl rounded-3xl border border-gray-200/50 bg-gray-100/50 px-8 py-8 backdrop-blur-md dark:border-white/10 dark:bg-gray-800/40"
    >
      <!-- 标题和进度 -->
      <div class="mb-6 flex items-center justify-between">
        <div class="flex items-center gap-3">
          <div class="flex h-10 w-10 items-center justify-center rounded-xl bg-pink-50 dark:bg-pink-500/10">
            <UIcon name="i-heroicons-arrow-path" class="h-5 w-5 animate-spin text-pink-600 dark:text-pink-400" />
          </div>
          <div>
            <p class="text-lg font-semibold text-gray-900 dark:text-white">
              {{ t('home.import.batch.importing') }}
            </p>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{
                t('home.import.batch.progressCount', {
                  current: (batchProgress?.completed || 0) + 1,
                  total: batchProgress?.total || 0,
                })
              }}
            </p>
          </div>
        </div>
        <UButton color="error" variant="soft" size="sm" icon="i-heroicons-stop-circle" @click="handleCancelBatchImport">
          {{ t('home.import.batch.cancel') }}
        </UButton>
      </div>

      <!-- 文件列表 -->
      <div class="max-h-80 space-y-2 overflow-y-auto">
        <div
          v-for="(file, index) in batchFiles"
          :key="file.path"
          class="flex items-center gap-3 rounded-lg px-3 py-2 transition-colors"
          :class="{
            'bg-pink-50/50 dark:bg-pink-500/5': file.status === 'importing',
          }"
        >
          <UIcon :name="getFileStatusIcon(file)" class="h-5 w-5 shrink-0" :class="getFileStatusClass(file)" />
          <div class="min-w-0 flex-1">
            <p class="truncate text-sm font-medium text-gray-900 dark:text-white">
              {{ file.name }}
            </p>
            <p class="text-xs text-gray-500 dark:text-gray-400">
              {{ getFileProgressText(file) }}
            </p>
          </div>
          <span class="text-xs text-gray-400">{{ index + 1 }}/{{ batchFiles.length }}</span>
        </div>
      </div>
    </div>

    <!-- 批量导入结果摘要 -->
    <div
      v-else-if="batchImportResult"
      class="w-full max-w-4xl rounded-3xl border border-gray-200/50 bg-gray-100/50 px-8 py-8 backdrop-blur-md dark:border-white/10 dark:bg-gray-800/40"
    >
      <!-- 标题 -->
      <div class="mb-6 flex items-center justify-between">
        <div class="flex items-center gap-3">
          <div
            class="flex h-10 w-10 items-center justify-center rounded-xl"
            :class="
              batchImportResult.failed === 0 ? 'bg-green-50 dark:bg-green-500/10' : 'bg-amber-50 dark:bg-amber-500/10'
            "
          >
            <UIcon
              :name="batchImportResult.failed === 0 ? 'i-heroicons-check-circle' : 'i-heroicons-exclamation-triangle'"
              class="h-5 w-5"
              :class="
                batchImportResult.failed === 0
                  ? 'text-green-600 dark:text-green-400'
                  : 'text-amber-600 dark:text-amber-400'
              "
            />
          </div>
          <div>
            <p class="text-lg font-semibold text-gray-900 dark:text-white">
              {{ t('home.import.batch.completed') }}
            </p>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{
                t('home.import.batch.summary', {
                  success: batchImportResult.success,
                  failed: batchImportResult.failed,
                  cancelled: batchImportResult.cancelled,
                })
              }}
            </p>
          </div>
        </div>
        <UButton color="neutral" variant="ghost" size="sm" icon="i-heroicons-x-mark" @click="handleCloseResult" />
      </div>

      <!-- 文件列表 -->
      <div class="max-h-80 space-y-2 overflow-y-auto">
        <div
          v-for="file in batchImportResult.files"
          :key="file.path"
          class="flex items-center gap-3 rounded-lg px-3 py-2"
        >
          <UIcon :name="getFileStatusIcon(file)" class="h-5 w-5 shrink-0" :class="getFileStatusClass(file)" />
          <div class="min-w-0 flex-1">
            <p class="truncate text-sm font-medium text-gray-900 dark:text-white">
              {{ file.name }}
            </p>
            <p v-if="file.status === 'failed'" class="text-xs text-red-500">
              {{ translateError(file.error || 'error.import_failed') }}
            </p>
            <p v-else-if="file.status === 'cancelled'" class="text-xs text-gray-500">
              {{ t('home.import.batch.skipped') }}
            </p>
          </div>
          <UButton
            v-if="file.status === 'success' && file.sessionId"
            size="xs"
            variant="soft"
            @click="handleGoToSession(file.sessionId)"
          >
            {{ t('home.import.batch.view') }}
          </UButton>
        </div>
      </div>
    </div>

    <!-- 默认导入区域（非批量状态） -->
    <FileDropZone
      v-else
      :accept="['.json', '.jsonl', '.txt']"
      :disabled="isAnyImporting"
      :multiple="true"
      class="w-full max-w-4xl"
      @files="handleFileDrop"
    >
      <template #default="{ isDragOver }">
        <div
          class="group relative flex w-full cursor-pointer flex-col items-center justify-center rounded-3xl border border-gray-200/50 bg-gray-100/50 px-8 py-10 backdrop-blur-md transition-all duration-300 hover:border-pink-500/30 hover:bg-gray-100/80 hover:shadow-2xl hover:shadow-pink-500/10 focus:outline-none focus:ring-4 focus:ring-pink-500/20 sm:px-12 sm:py-14 dark:border-white/10 dark:bg-gray-800/40 dark:hover:border-pink-500/30 dark:hover:bg-gray-800/60"
          :class="{
            'border-pink-500/50 bg-pink-50/50 dark:border-pink-400/50 dark:bg-pink-500/10':
              isDragOver && !isAnyImporting,
            'cursor-not-allowed opacity-70': isAnyImporting,
          }"
          @click="!isAnyImporting && handleClickImport()"
        >
          <!-- Icon -->
          <div
            class="mb-6 flex h-16 w-16 items-center justify-center rounded-2xl bg-pink-50 transition-transform duration-300 group-hover:scale-105 dark:bg-pink-500/10"
            :class="{ 'scale-105': isDragOver && !isAnyImporting, 'animate-pulse': isImporting }"
          >
            <UIcon
              v-if="!isImporting"
              name="i-heroicons-arrow-up-tray"
              class="h-8 w-8 text-pink-600 transition-transform duration-200 group-hover:-translate-y-1 dark:text-pink-400"
            />
            <UIcon v-else name="i-heroicons-arrow-path" class="h-8 w-8 animate-spin text-pink-600 dark:text-pink-400" />
          </div>

          <!-- Text -->
          <div class="w-full min-w-80 text-center">
            <template v-if="isImporting && importProgress">
              <!-- 导入中显示进度 -->
              <p class="mb-4 text-lg font-semibold text-gray-900 dark:text-white">{{ getProgressText() }}</p>
              <div class="mx-auto w-full max-w-md">
                <UProgress v-model="importProgress.progress" size="md" />
              </div>
              <p class="mt-3 text-sm text-gray-500 dark:text-gray-400">
                {{ getProgressDetail() }}
              </p>
            </template>
            <template v-else>
              <!-- 默认状态 -->
              <p class="text-lg font-semibold text-gray-900 dark:text-white">
                {{ isDragOver ? t('home.import.dropHint') : t('home.import.clickHint') }}
              </p>
              <p class="mt-2 text-sm text-gray-500 dark:text-gray-400">
                {{ t('home.import.multipleHint') }}
              </p>
            </template>
          </div>
        </div>
      </template>
    </FileDropZone>

    <!-- Error Message -->
    <div
      v-if="importError"
      class="flex max-w-lg flex-col items-center gap-3 rounded-lg bg-red-50 px-4 py-4 dark:bg-red-900/20"
    >
      <div class="flex items-center gap-2 text-sm text-red-600 dark:text-red-400">
        <UIcon name="i-heroicons-exclamation-circle" class="h-5 w-5 shrink-0" />
        <span>{{ importError }}</span>
      </div>
      <!-- 诊断建议（如果有） -->
      <div
        v-if="diagnosisSuggestion"
        class="w-full rounded-md bg-amber-50 px-3 py-2 text-sm text-amber-800 dark:bg-amber-900/30 dark:text-amber-200"
      >
        <div class="flex items-start gap-2">
          <UIcon name="i-heroicons-light-bulb" class="mt-0.5 h-4 w-4 shrink-0" />
          <span>{{ diagnosisSuggestion }}</span>
        </div>
      </div>
      <UButton v-if="hasImportLog" size="xs" @click="openLatestImportLog">{{ t('home.import.viewLog') }}</UButton>
    </div>

    <UButton @click="openTutorial" trailing-icon="i-heroicons-chevron-right-20-solid">
      {{ t('home.import.tutorial') }}
    </UButton>
  </div>
</template>
