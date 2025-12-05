<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { useChatStore } from '@/stores/chat'
import type { PromptPreset } from '@/types/chat'

// Props
const props = defineProps<{
  open: boolean
  mode: 'add' | 'edit'
  preset: PromptPreset | null
  defaultChatType: 'group' | 'private'
}>()

// Emits
const emit = defineEmits<{
  'update:open': [value: boolean]
  saved: []
}>()

// Store
const chatStore = useChatStore()

// 表单数据
const formData = ref({
  name: '',
  chatType: 'group' as 'group' | 'private',
  roleDefinition: '',
  responseRules: '',
})

// 计算属性
const isBuiltIn = computed(() => props.preset?.isBuiltIn ?? false)
const isEditMode = computed(() => props.mode === 'edit')
const modalTitle = computed(() => {
  if (isBuiltIn.value) return '查看预设'
  return isEditMode.value ? '编辑预设' : '添加预设'
})

const canSave = computed(() => {
  return formData.value.name.trim() && formData.value.roleDefinition.trim() && formData.value.responseRules.trim()
})

// 获取默认角色定义
function getDefaultRoleDefinition(chatType: 'group' | 'private'): string {
  if (chatType === 'private') {
    return `你是一个专业的私聊记录分析助手。
你的任务是帮助用户理解和分析他们的私聊记录数据。

注意：这是一个私聊对话，只有两个人参与。你的分析应该关注：
- 两人之间的对话互动
- 谁更主动、谁回复更多
- 对话的主题和内容变化
- 不要使用"群"这个词，使用"对话"或"聊天"`
  }
  return `你是一个专业的群聊记录分析助手。
你的任务是帮助用户理解和分析他们的群聊记录数据。`
}

// 获取默认回答要求
function getDefaultResponseRules(chatType: 'group' | 'private'): string {
  if (chatType === 'private') {
    return `1. 基于工具返回的数据回答，不要编造信息
2. 如果数据不足以回答问题，请说明
3. 回答要简洁明了，使用 Markdown 格式
4. 可以引用具体的发言作为证据
5. 关注两人之间的互动模式和对话特点`
  }
  return `1. 基于工具返回的数据回答，不要编造信息
2. 如果数据不足以回答问题，请说明
3. 回答要简洁明了，使用 Markdown 格式
4. 可以引用具体的发言作为证据
5. 对于统计数据，可以适当总结趋势和特点`
}

// 监听打开状态，初始化表单
watch(
  () => props.open,
  (newVal) => {
    if (newVal) {
      if (props.preset) {
        // 编辑模式：加载现有预设
        formData.value = {
          name: props.preset.name,
          chatType: props.preset.chatType,
          roleDefinition: props.preset.roleDefinition,
          responseRules: props.preset.responseRules,
        }
      } else {
        // 添加模式：重置为默认（根据当前选中的聊天类型）
        formData.value = {
          name: '',
          chatType: props.defaultChatType,
          roleDefinition: getDefaultRoleDefinition(props.defaultChatType),
          responseRules: getDefaultResponseRules(props.defaultChatType),
        }
      }
    }
  }
)

// 方法
function closeModal() {
  emit('update:open', false)
}

function handleSave() {
  if (!canSave.value || isBuiltIn.value) return

  if (isEditMode.value && props.preset) {
    // 更新现有预设
    chatStore.updatePromptPreset(props.preset.id, {
      name: formData.value.name.trim(),
      chatType: formData.value.chatType,
      roleDefinition: formData.value.roleDefinition.trim(),
      responseRules: formData.value.responseRules.trim(),
    })
  } else {
    // 添加新预设
    chatStore.addPromptPreset({
      name: formData.value.name.trim(),
      chatType: formData.value.chatType,
      roleDefinition: formData.value.roleDefinition.trim(),
      responseRules: formData.value.responseRules.trim(),
    })
  }

  emit('saved')
  closeModal()
}

// 预览完整提示词（始终展示）

// 获取锁定部分的提示词（与 Agent 中的一致）
function getLockedPromptSection(chatType: string): string {
  const now = new Date()
  const currentDate = now.toLocaleDateString('zh-CN', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    weekday: 'long',
  })

  const isPrivate = chatType === 'private'
  const chatTypeDesc = isPrivate ? '私聊记录' : '群聊记录'

  const memberNote = isPrivate
    ? `成员查询策略：
- 私聊只有两个人，可以直接获取成员列表
- 当用户提到"对方"、"他/她"时，通过 get_group_members 获取另一方信息`
    : `成员查询策略：
- 当用户提到特定群成员（如"张三说过什么"、"小明的发言"等）时，应先调用 get_group_members 获取成员列表
- 群成员有三种名称：accountName（QQ原始昵称）、groupNickname（群昵称）、aliases（用户自定义别名）
- 通过 get_group_members 的 search 参数可以模糊搜索这三种名称
- 找到成员后，使用其 id 字段作为 search_messages 的 sender_id 参数来获取该成员的发言`

  return `当前日期是 ${currentDate}。

你可以使用以下工具来获取${chatTypeDesc}数据：

1. search_messages - 根据关键词搜索聊天记录，可指定 year/month 筛选时间段，也可指定 sender_id 筛选特定成员的发言
2. get_recent_messages - 获取指定时间段的聊天消息，可指定 year 和 month
3. get_member_stats - 获取成员活跃度统计
4. get_time_stats - 获取时间分布统计
5. get_group_members - 获取成员列表，包括 id、QQ号、账号名称、昵称、别名和消息统计
6. get_member_name_history - 获取成员的昵称变更历史，需要先通过 get_group_members 获取成员 ID
7. get_conversation_between - 获取两个成员之间的对话记录，需要先通过 get_group_members 获取两人的成员 ID

${memberNote}

时间处理要求：
- 如果用户提到"X月"但没有指定年份，默认使用当前年份（${now.getFullYear()}年）
- 如果当前月份还没到用户提到的月份，则使用去年
- 例如：现在是${now.getFullYear()}年${now.getMonth() + 1}月，用户问"10月的聊天"应该查询${now.getMonth() + 1 >= 10 ? now.getFullYear() : now.getFullYear() - 1}年10月

根据用户的问题，选择合适的工具获取数据，然后基于数据给出回答。`
}

const previewContent = computed(() => {
  const chatType = formData.value.chatType

  // 获取锁定的系统部分
  const lockedSection = getLockedPromptSection(chatType)

  // 组合完整提示词（与 Agent 中的 buildSystemPrompt 保持一致）
  return `${formData.value.roleDefinition}

${lockedSection}

回答要求：
${formData.value.responseRules}`
})
</script>

<template>
  <UModal :open="open" @update:open="emit('update:open', $event)" :ui="{ content: 'md:w-full max-w-2xl' }">
    <template #content>
      <div class="p-6">
        <!-- Header -->
        <div class="mb-4 flex items-center justify-between">
          <h2 class="text-lg font-semibold text-gray-900 dark:text-white">{{ modalTitle }}</h2>
          <UButton icon="i-heroicons-x-mark" variant="ghost" size="sm" @click="closeModal" />
        </div>

        <!-- 内置预设提示 -->
        <UAlert v-if="isBuiltIn" color="info" variant="outline" icon="i-heroicons-information-circle" class="mb-4">
          <template #title>
            <span class="text-sm">内置预设不可编辑，你可以复制后修改。</span>
          </template>
        </UAlert>

        <!-- 表单 -->
        <div class="max-h-[500px] space-y-4 overflow-y-auto pr-1">
          <!-- 预设名称 -->
          <div>
            <label class="mb-1.5 block text-sm font-medium text-gray-700 dark:text-gray-300">预设名称</label>
            <UInput v-model="formData.name" placeholder="为预设起个名字" :disabled="isBuiltIn" />
          </div>

          <!-- 适用类型（只读显示） -->
          <div class="flex items-center gap-2 text-sm text-gray-600 dark:text-gray-400">
            <UIcon :name="formData.chatType === 'group' ? 'i-heroicons-chat-bubble-left-right' : 'i-heroicons-user'" class="h-4 w-4" />
            <span>适用于{{ formData.chatType === 'group' ? '群聊' : '私聊' }}</span>
          </div>

          <!-- 角色定义 -->
          <div>
            <label class="mb-1.5 block text-sm font-medium text-gray-700 dark:text-gray-300">角色定义</label>
            <UTextarea
              v-model="formData.roleDefinition"
              :rows="5"
              placeholder="定义 AI 助手的角色和任务..."
              :disabled="isBuiltIn"
              class="font-mono text-sm"
            />
          </div>

          <!-- 回答要求 -->
          <div>
            <label class="mb-1.5 block text-sm font-medium text-gray-700 dark:text-gray-300">
              回答要求
              <span class="font-normal text-gray-500">（指导 AI 如何回答）</span>
            </label>
            <UTextarea
              v-model="formData.responseRules"
              :rows="5"
              placeholder="定义 AI 回答的格式和要求..."
              :disabled="isBuiltIn"
              class="font-mono text-sm"
            />
          </div>

          <!-- 完整提示词预览 -->
          <div>
            <label class="mb-1.5 flex items-center gap-2 text-sm font-medium text-gray-700 dark:text-gray-300">
              <UIcon name="i-heroicons-eye" class="h-4 w-4 text-violet-500" />
              完整提示词预览
            </label>
            <div class="rounded-lg border border-gray-200 bg-gray-50 p-4 dark:border-gray-700 dark:bg-gray-800/50">
              <pre class="max-h-48 overflow-y-auto whitespace-pre-wrap text-xs text-gray-700 dark:text-gray-300">{{ previewContent }}</pre>
            </div>
          </div>
        </div>

        <!-- Footer -->
        <div class="mt-6 flex justify-end gap-2">
          <UButton variant="ghost" @click="closeModal">取消</UButton>
          <UButton v-if="!isBuiltIn" color="primary" :disabled="!canSave" @click="handleSave">
            {{ isEditMode ? '保存修改' : '添加预设' }}
          </UButton>
        </div>
      </div>
    </template>
  </UModal>
</template>

