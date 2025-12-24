<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useLayoutStore } from '@/stores/layout'
import { useColorMode } from '@vueuse/core'

// Store
const layoutStore = useLayoutStore()
const { screenshotMobileAdapt } = storeToRefs(layoutStore)

// Color Mode
const colorMode = useColorMode({
  emitAuto: true,
  initialValue: 'light',
})

const colorModeOptions = [
  { label: '跟随系统', value: 'auto' },
  { label: '浅色模式', value: 'light' },
  { label: '深色模式', value: 'dark' },
]
</script>

<template>
  <div class="space-y-6">
    <!-- 外观设置 -->
    <div>
      <h3 class="mb-3 flex items-center gap-2 text-sm font-semibold text-gray-900 dark:text-white">
        <UIcon name="i-heroicons-paint-brush" class="h-4 w-4 text-pink-500" />
        外观设置
      </h3>
      <div class="rounded-lg border border-gray-200 bg-gray-50 p-4 dark:border-gray-700 dark:bg-gray-800/50">
        <div class="flex items-center justify-between">
          <div class="flex-1 pr-4">
            <p class="text-sm font-medium text-gray-900 dark:text-white">主题模式</p>
          </div>
          <div class="w-64">
            <UTabs v-model="colorMode" :items="colorModeOptions"></UTabs>
          </div>
        </div>
      </div>
    </div>

    <!-- 截图设置 -->
    <div>
      <h3 class="mb-3 flex items-center gap-2 text-sm font-semibold text-gray-900 dark:text-white">
        <UIcon name="i-heroicons-camera" class="h-4 w-4 text-blue-500" />
        截图设置
      </h3>
      <div class="rounded-lg border border-gray-200 bg-gray-50 p-4 dark:border-gray-700 dark:bg-gray-800/50">
        <div class="flex items-center justify-between">
          <div class="flex-1 pr-4">
            <p class="text-sm font-medium text-gray-900 dark:text-white">移动端适配</p>
            <p class="text-xs text-gray-500 dark:text-gray-400">截图时自动缩放宽度，适合移动端查看</p>
          </div>
          <USwitch v-model="screenshotMobileAdapt" />
        </div>
      </div>
    </div>
  </div>
</template>
