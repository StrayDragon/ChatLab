<script setup lang="ts">
import { useI18n } from 'vue-i18n'
import { useLayoutStore } from '@/stores/layout'
import SidebarButton from './SidebarButton.vue'

const { t } = useI18n()
const layoutStore = useLayoutStore()

// 注意：帮助和反馈功能已迁移到首页 Footer (HomeFooter.vue)
// 如需恢复，请参考 git 历史记录
</script>

<template>
  <div class="px-4 py-2 dark:border-gray-800 space-y-2 mb-2">
    <!-- 设置 -->
    <SidebarButton
      icon="i-heroicons-cog-6-tooth"
      :title="t('sidebar.footer.settings')"
      @click="layoutStore.showSettingModal = true"
    />
  </div>
</template>
