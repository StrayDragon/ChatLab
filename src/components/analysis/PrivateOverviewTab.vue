<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import type { AnalysisSession, MemberActivity, HourlyActivity, MessageType, WeekdayActivity } from '@/types/chat'
import { getMessageTypeName } from '@/types/chat'
import { DoughnutChart, BarChart } from '@/components/charts'
import type { DoughnutChartData, BarChartData } from '@/components/charts'
import { SectionCard, StatCard } from '@/components/UI'

const props = defineProps<{
  session: AnalysisSession
  memberActivity: MemberActivity[]
  messageTypes: Array<{ type: MessageType; count: number }>
  hourlyActivity: HourlyActivity[]
  timeRange: { start: number; end: number } | null
  selectedYear: number | null
  filteredMessageCount: number
  filteredMemberCount: number
  timeFilter?: { startTs?: number; endTs?: number }
}>()

// 时间跨度
const durationDays = computed(() => {
  if (props.selectedYear) {
    const isLeapYear =
      (props.selectedYear % 4 === 0 && props.selectedYear % 100 !== 0) || props.selectedYear % 400 === 0
    return isLeapYear ? 366 : 365
  }
  if (!props.timeRange) return 0
  return Math.ceil((props.timeRange.end - props.timeRange.start) / 86400)
})

// 显示的消息数
const displayMessageCount = computed(() => {
  return props.selectedYear ? props.filteredMessageCount : props.session.messageCount
})

// 消息类型图表数据
const typeChartData = computed<DoughnutChartData>(() => {
  return {
    labels: props.messageTypes.map((t) => getMessageTypeName(t.type)),
    values: props.messageTypes.map((t) => t.count),
  }
})

// 双方消息对比数据
const memberComparisonData = computed(() => {
  if (props.memberActivity.length !== 2) return null

  const sorted = [...props.memberActivity].sort((a, b) => b.messageCount - a.messageCount)
  const total = sorted[0].messageCount + sorted[1].messageCount

  return {
    member1: {
      name: sorted[0].name,
      count: sorted[0].messageCount,
      percentage: total > 0 ? Math.round((sorted[0].messageCount / total) * 100) : 0,
    },
    member2: {
      name: sorted[1].name,
      count: sorted[1].messageCount,
      percentage: total > 0 ? Math.round((sorted[1].messageCount / total) * 100) : 0,
    },
    total,
  }
})

// 双方对比图表数据
const comparisonChartData = computed<DoughnutChartData>(() => {
  if (!memberComparisonData.value) {
    return { labels: [], values: [] }
  }
  return {
    labels: [memberComparisonData.value.member1.name, memberComparisonData.value.member2.name],
    values: [memberComparisonData.value.member1.count, memberComparisonData.value.member2.count],
  }
})

// 最活跃时段
const peakHour = computed(() => {
  if (!props.hourlyActivity.length) return null
  const peak = props.hourlyActivity.reduce(
    (max, h) => (h.messageCount > max.messageCount ? h : max),
    props.hourlyActivity[0]
  )
  return peak
})

// 图片消息数量
const imageCount = computed(() => {
  const imageType = props.messageTypes.find((t) => t.type === 1)
  return imageType?.count || 0
})

// 日均消息数
const dailyAvgMessages = computed(() => {
  if (durationDays.value === 0) return 0
  return Math.round(displayMessageCount.value / durationDays.value)
})

// 星期活跃度数据
const weekdayActivity = ref<WeekdayActivity[]>([])
const isLoadingWeekday = ref(false)

// 星期名称映射（周一开始）
const weekdayNames = ['周一', '周二', '周三', '周四', '周五', '周六', '周日']

// 加载星期活跃度数据
async function loadWeekdayActivity() {
  if (!props.session.id) return
  isLoadingWeekday.value = true
  try {
    weekdayActivity.value = await window.chatApi.getWeekdayActivity(props.session.id, props.timeFilter)
  } catch (error) {
    console.error('加载星期活跃度失败:', error)
  } finally {
    isLoadingWeekday.value = false
  }
}

// 监听 session.id 和 timeFilter 变化
watch(
  () => [props.session.id, props.timeFilter],
  () => {
    loadWeekdayActivity()
  },
  { immediate: true, deep: true }
)

// 24小时分布图数据
const hourlyChartData = computed<BarChartData>(() => {
  return {
    labels: props.hourlyActivity.map((h) => `${h.hour}:00`),
    values: props.hourlyActivity.map((h) => h.messageCount),
  }
})

// 星期分布图数据
const weekdayChartData = computed<BarChartData>(() => {
  return {
    labels: weekdayActivity.value.map((w) => weekdayNames[w.weekday - 1]),
    values: weekdayActivity.value.map((w) => w.messageCount),
  }
})

// 最活跃星期
const peakWeekday = computed(() => {
  if (!weekdayActivity.value.length) return null
  return weekdayActivity.value.reduce(
    (max, w) => (w.messageCount > max.messageCount ? w : max),
    weekdayActivity.value[0]
  )
})

// 时段占比计算
const totalMessages = computed(() => props.hourlyActivity.reduce((sum, h) => sum + h.messageCount, 0))

const lateNightRatio = computed(() => {
  const lateNight = props.hourlyActivity
    .filter((h) => h.hour >= 0 && h.hour < 6)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((lateNight / totalMessages.value) * 100) : 0
})

const morningRatio = computed(() => {
  const morning = props.hourlyActivity
    .filter((h) => h.hour >= 6 && h.hour < 12)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((morning / totalMessages.value) * 100) : 0
})

const afternoonRatio = computed(() => {
  const afternoon = props.hourlyActivity
    .filter((h) => h.hour >= 12 && h.hour < 18)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((afternoon / totalMessages.value) * 100) : 0
})

const eveningRatio = computed(() => {
  const evening = props.hourlyActivity
    .filter((h) => h.hour >= 18 && h.hour < 24)
    .reduce((sum, h) => sum + h.messageCount, 0)
  return totalMessages.value > 0 ? Math.round((evening / totalMessages.value) * 100) : 0
})

const weekdayVsWeekend = computed(() => {
  if (!weekdayActivity.value.length) return { weekday: 0, weekend: 0 }
  const weekdaySum = weekdayActivity.value
    .filter((w) => w.weekday >= 1 && w.weekday <= 5)
    .reduce((sum, w) => sum + w.messageCount, 0)
  const weekendSum = weekdayActivity.value
    .filter((w) => w.weekday >= 6 && w.weekday <= 7)
    .reduce((sum, w) => sum + w.messageCount, 0)
  const total = weekdaySum + weekendSum
  return {
    weekday: total > 0 ? Math.round((weekdaySum / total) * 100) : 0,
    weekend: total > 0 ? Math.round((weekendSum / total) * 100) : 0,
  }
})
</script>

<template>
  <div class="space-y-6 p-6">
    <!-- 私聊身份卡 -->
    <div class="relative overflow-hidden rounded-3xl bg-pink-500 p-8 text-white shadow-xl">
      <div class="relative">
        <div>
          <div class="flex items-center gap-3">
            <h2 class="text-3xl font-black tracking-tight">{{ session.name }}</h2>
            <span class="rounded-full bg-white/20 px-3 py-1 text-xs font-medium backdrop-blur-md">
              {{ session.platform.toUpperCase() }}
            </span>
          </div>
          <p class="mt-2 text-lg text-white/90 font-medium">
            私聊 ·
            <span class="opacity-80">数据分析报告</span>
          </p>
        </div>

        <div class="mt-8 grid grid-cols-3 gap-6">
          <div class="rounded-2xl bg-white/10 px-6 py-4 backdrop-blur-md">
            <p class="text-3xl font-black tracking-tight">{{ displayMessageCount.toLocaleString() }}</p>
            <p class="mt-1 text-sm font-medium text-white/70">{{ selectedYear ? '筛选消息' : '消息总数' }}</p>
          </div>
          <div class="rounded-2xl bg-white/10 px-6 py-4 backdrop-blur-md">
            <p class="text-3xl font-black tracking-tight">{{ durationDays }}</p>
            <p class="mt-1 text-sm font-medium text-white/70">跨度天数</p>
          </div>
          <div class="rounded-2xl bg-white/10 px-6 py-4 backdrop-blur-md">
            <p class="text-3xl font-black tracking-tight">{{ dailyAvgMessages }}</p>
            <p class="mt-1 text-sm font-medium text-white/70">日均消息</p>
          </div>
        </div>
      </div>
    </div>

    <!-- 双方消息对比 -->
    <SectionCard v-if="memberComparisonData" title="消息占比" :show-divider="false">
      <div class="p-5">
        <div class="flex items-center gap-8">
          <!-- 左侧成员 -->
          <div class="flex-1 text-center">
            <div
              class="mx-auto flex h-16 w-16 items-center justify-center rounded-full bg-pink-100 dark:bg-pink-900/30"
            >
              <span class="text-2xl font-bold text-pink-600 dark:text-pink-400">
                {{ memberComparisonData.member1.name.charAt(0) }}
              </span>
            </div>
            <p class="mt-2 text-sm font-medium text-gray-900 dark:text-white">
              {{ memberComparisonData.member1.name }}
            </p>
            <p class="text-2xl font-black text-pink-600 dark:text-pink-400">
              {{ memberComparisonData.member1.percentage }}%
            </p>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{ memberComparisonData.member1.count.toLocaleString() }} 条
            </p>
          </div>

          <!-- 中间对比条 -->
          <div class="flex-1">
            <div class="relative h-8 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
              <div
                class="absolute left-0 top-0 h-full rounded-l-full bg-pink-500 transition-all"
                :style="{ width: `${memberComparisonData.member1.percentage}%` }"
              />
              <div
                class="absolute right-0 top-0 h-full rounded-r-full bg-blue-500 transition-all"
                :style="{ width: `${memberComparisonData.member2.percentage}%` }"
              />
            </div>
            <div class="mt-2 flex justify-between text-xs text-gray-500">
              <span>{{ memberComparisonData.member1.percentage }}%</span>
              <span>{{ memberComparisonData.member2.percentage }}%</span>
            </div>
          </div>

          <!-- 右侧成员 -->
          <div class="flex-1 text-center">
            <div
              class="mx-auto flex h-16 w-16 items-center justify-center rounded-full bg-blue-100 dark:bg-blue-900/30"
            >
              <span class="text-2xl font-bold text-blue-600 dark:text-blue-400">
                {{ memberComparisonData.member2.name.charAt(0) }}
              </span>
            </div>
            <p class="mt-2 text-sm font-medium text-gray-900 dark:text-white">
              {{ memberComparisonData.member2.name }}
            </p>
            <p class="text-2xl font-black text-blue-600 dark:text-blue-400">
              {{ memberComparisonData.member2.percentage }}%
            </p>
            <p class="text-sm text-gray-500 dark:text-gray-400">
              {{ memberComparisonData.member2.count.toLocaleString() }} 条
            </p>
          </div>
        </div>
      </div>
    </SectionCard>

    <!-- 关键指标卡片 -->
    <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
      <!-- 日均消息 -->
      <StatCard label="日均消息" :value="`${dailyAvgMessages} 条`" icon="📊" icon-bg="blue">
        <template #subtext>
          <span class="text-sm text-gray-500">共 {{ durationDays }} 天</span>
        </template>
      </StatCard>

      <!-- 图片/表情 -->
      <StatCard label="图片消息" :value="`${imageCount} 张`" icon="📸" icon-bg="pink">
        <template #subtext>
          <span class="text-sm text-gray-500">最活跃时段:</span>
          <span class="font-semibold text-pink-500">{{ peakHour?.hour || 0 }}:00</span>
        </template>
      </StatCard>

      <!-- 最活跃星期 -->
      <StatCard
        label="最活跃星期"
        :value="peakWeekday ? weekdayNames[peakWeekday.weekday - 1] : '-'"
        icon="📅"
        icon-bg="amber"
      >
        <template #subtext>
          <span class="text-sm text-gray-500">{{ peakWeekday?.messageCount ?? 0 }} 条消息</span>
        </template>
      </StatCard>

      <!-- 周末活跃度 -->
      <StatCard label="周末活跃度" :value="`${weekdayVsWeekend.weekend}%`" icon="🏖️" icon-bg="green">
        <template #subtext>
          <span class="text-sm text-gray-500">周末消息占比</span>
        </template>
      </StatCard>
    </div>

    <!-- 图表区域：消息类型 & 双方占比 -->
    <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
      <!-- 消息类型分布 -->
      <SectionCard title="消息类型分布" :show-divider="false">
        <div class="p-5">
          <DoughnutChart :data="typeChartData" :height="256" />
        </div>
      </SectionCard>

      <!-- 双方消息占比饼图 -->
      <SectionCard v-if="memberComparisonData" title="双方消息占比" :show-divider="false">
        <div class="p-5">
          <DoughnutChart :data="comparisonChartData" :height="256" />
        </div>
      </SectionCard>
    </div>

    <!-- 24小时 & 星期分布 -->
    <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
      <!-- 24小时分布 -->
      <SectionCard title="24小时活跃分布" :show-divider="false">
        <div class="p-5">
          <BarChart
            :data="hourlyChartData"
            :height="256"
            :x-label-filter="(_, index) => (index % 3 === 0 ? `${index}:00` : '')"
          />

          <div class="mt-6 grid grid-cols-4 gap-2">
            <div class="text-center">
              <div class="text-xs text-gray-500 dark:text-gray-400">凌晨</div>
              <div class="mt-1 text-base font-semibold text-gray-900 dark:text-white">{{ lateNightRatio }}%</div>
              <div class="mx-auto mt-1 h-1 w-full max-w-12 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
                <div class="h-full rounded-full bg-pink-300 transition-all" :style="{ width: `${lateNightRatio}%` }" />
              </div>
            </div>
            <div class="text-center">
              <div class="text-xs text-gray-500 dark:text-gray-400">上午</div>
              <div class="mt-1 text-base font-semibold text-gray-900 dark:text-white">{{ morningRatio }}%</div>
              <div class="mx-auto mt-1 h-1 w-full max-w-12 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
                <div class="h-full rounded-full bg-pink-400 transition-all" :style="{ width: `${morningRatio}%` }" />
              </div>
            </div>
            <div class="text-center">
              <div class="text-xs text-gray-500 dark:text-gray-400">下午</div>
              <div class="mt-1 text-base font-semibold text-gray-900 dark:text-white">{{ afternoonRatio }}%</div>
              <div class="mx-auto mt-1 h-1 w-full max-w-12 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
                <div class="h-full rounded-full bg-pink-500 transition-all" :style="{ width: `${afternoonRatio}%` }" />
              </div>
            </div>
            <div class="text-center">
              <div class="text-xs text-gray-500 dark:text-gray-400">晚上</div>
              <div class="mt-1 text-base font-semibold text-gray-900 dark:text-white">{{ eveningRatio }}%</div>
              <div class="mx-auto mt-1 h-1 w-full max-w-12 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
                <div class="h-full rounded-full bg-pink-600 transition-all" :style="{ width: `${eveningRatio}%` }" />
              </div>
            </div>
          </div>
        </div>
      </SectionCard>

      <!-- 星期分布 -->
      <SectionCard title="星期活跃分布" :show-divider="false">
        <div class="p-5">
          <div v-if="isLoadingWeekday" class="flex h-64 items-center justify-center">
            <UIcon name="i-heroicons-arrow-path" class="h-6 w-6 animate-spin text-pink-500" />
          </div>
          <BarChart v-else :data="weekdayChartData" :height="256" />

          <div class="mt-6 grid grid-cols-2 gap-4">
            <div class="text-center">
              <div class="text-xs text-gray-500 dark:text-gray-400">工作日</div>
              <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">
                {{ weekdayVsWeekend.weekday }}%
              </div>
              <div class="mx-auto mt-2 h-1 w-full max-w-24 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
                <div
                  class="h-full rounded-full bg-pink-500 transition-all"
                  :style="{ width: `${weekdayVsWeekend.weekday}%` }"
                />
              </div>
            </div>
            <div class="text-center">
              <div class="text-xs text-gray-500 dark:text-gray-400">周末</div>
              <div class="mt-1 text-lg font-semibold text-gray-900 dark:text-white">
                {{ weekdayVsWeekend.weekend }}%
              </div>
              <div class="mx-auto mt-2 h-1 w-full max-w-24 overflow-hidden rounded-full bg-gray-100 dark:bg-gray-800">
                <div
                  class="h-full rounded-full bg-blue-500 transition-all"
                  :style="{ width: `${weekdayVsWeekend.weekend}%` }"
                />
              </div>
            </div>
          </div>
        </div>
      </SectionCard>
    </div>
  </div>
</template>
