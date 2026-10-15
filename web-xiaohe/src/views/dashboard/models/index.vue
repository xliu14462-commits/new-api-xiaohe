<template>
  <div v-loading="loading" class="flex flex-col gap-4">
    <!-- 页头 -->
    <div class="flex flex-wrap items-center justify-between gap-3">
      <div>
        <h2 class="text-lg font-semibold text-g-900">{{ t('dashboard.modelAnalytics.title') }}</h2>
        <p class="mt-1 text-sm text-g-500">{{ currentRangeLabel }}</p>
      </div>
      <ElButton @click="openFilterDialog" class="!border-[#e8ecf0] !bg-white !text-g-700 hover:!border-theme hover:!text-theme">
        <ArtSvgIcon class="mr-1.5" icon="ri:filter-3-line" />
        {{ t('dashboard.modelAnalytics.filter') }}
      </ElButton>
    </div>

    <!-- 统计卡片：白底 + 浅灰边框 + 圆形柔和图标 -->
    <div class="grid grid-cols-2 overflow-hidden rounded-xl border border-[#e8ecf0] bg-white lg:grid-cols-5">
      <div
        v-for="(item, idx) in statCards"
        :key="item.title"
        class="border-[#e8ecf0] p-5"
        :class="[
          'border-b',
          idx < 4 ? 'lg:border-r lg:border-b-0' : 'lg:border-b-0',
          idx >= 2 ? 'max-lg:border-b-0' : '',
          idx % 2 === 0 ? 'max-lg:border-r' : ''
        ]"
      >
        <div class="flex items-center gap-2.5">
          <span
            class="flex h-8 w-8 items-center justify-center rounded-full"
            :class="item.iconBadge"
          >
            <ArtSvgIcon :icon="item.icon" class="text-sm" />
          </span>
          <span class="text-sm text-g-600">{{ item.title }}</span>
        </div>
        <div class="mt-3 font-mono text-2xl font-semibold tabular-nums text-g-900">
          {{ item.value }}
        </div>
        <p class="mt-1 text-xs text-g-400">{{ item.description }}</p>
      </div>
    </div>

    <!-- 额度分布图表卡片：圆角边框 + 胶囊 Tab -->
    <div class="overflow-hidden rounded-xl border border-[#e8ecf0] bg-white">
      <div class="flex w-full flex-col gap-1.5 border-b border-[#e8ecf0] px-4 py-3 sm:flex-row sm:items-center sm:justify-between">
        <div class="flex items-center gap-2">
          <span class="flex h-6 w-6 items-center justify-center rounded-md bg-theme/10 text-theme">
            <ArtSvgIcon icon="ri:pie-chart-line" class="text-xs" />
          </span>
          <span class="text-sm font-semibold text-g-900">
            {{ t('dashboard.modelAnalytics.quotaDistribution') }}
          </span>
          <span class="text-xs text-g-400">
            {{ t('dashboard.modelAnalytics.total') }}: {{ totalQuotaLabel }}
          </span>
        </div>
        <div class="bg-muted/60 inline-flex h-8 w-fit items-center rounded-lg border p-0.5">
          <button
            v-for="opt in [
              { value: 'bar', label: t('dashboard.modelAnalytics.barChart') },
              { value: 'area', label: t('dashboard.modelAnalytics.areaChart') }
            ]"
            :key="opt.value"
            @click="quotaChartMode = opt.value as 'bar' | 'area'"
            class="shrink-0 rounded-md px-3 py-1 text-xs font-medium transition-colors"
            :class="quotaChartMode === opt.value
              ? 'bg-background text-foreground shadow-sm'
              : 'text-g-500 hover:text-g-900'"
          >
            {{ opt.label }}
          </button>
        </div>
      </div>
      <div class="p-2">
        <ElEmpty v-if="!loading && !hasData" :description="t('dashboard.modelAnalytics.noData')" />
        <ArtBarChart
          v-else-if="quotaChartMode === 'bar'"
          :data="chartData.quotaByModel"
          :x-axis-data="chartData.timeLabels"
          height="20rem"
          :loading="loading"
          value-prefix="$"
          :show-legend="true"
          legend-position="top"
          :stack="true"
        />
        <ArtLineChart
          v-else
          :data="chartData.quotaByModel"
          :x-axis-data="chartData.timeLabels"
          height="20rem"
          :loading="loading"
          value-prefix="$"
          show-area-color
          show-legend
        />
      </div>
    </div>

    <!-- 模型调用分析图表卡片：圆角边框 + 胶囊 Tab -->
    <div class="overflow-hidden rounded-xl border border-[#e8ecf0] bg-white">
      <div class="flex w-full flex-col gap-1.5 border-b border-[#e8ecf0] px-4 py-3 sm:flex-row sm:items-center sm:justify-between">
        <div class="flex items-center gap-2">
          <span class="flex h-6 w-6 items-center justify-center rounded-md bg-theme/10 text-theme">
            <ArtSvgIcon icon="ri:line-chart-line" class="text-xs" />
          </span>
          <span class="text-sm font-semibold text-g-900">
            {{ t('dashboard.modelAnalytics.title') }}
          </span>
          <span class="text-xs text-g-400">
            {{ t('dashboard.modelAnalytics.total') }}:
            {{ formatMetric(summary.totalCount, locale) }}
          </span>
        </div>
        <div class="bg-muted/60 inline-flex h-8 w-fit items-center rounded-lg border p-0.5">
          <button
            v-for="tab in [
              { value: 'trend', label: t('dashboard.modelAnalytics.callTrend') },
              { value: 'distribution', label: t('dashboard.modelAnalytics.callDistribution') },
              { value: 'ranking', label: t('dashboard.modelAnalytics.callRanking') }
            ]"
            :key="tab.value"
            @click="analyticsTab = tab.value as any"
            class="shrink-0 rounded-md px-3 py-1 text-xs font-medium transition-colors"
            :class="analyticsTab === tab.value
              ? 'bg-background text-foreground shadow-sm'
              : 'text-g-500 hover:text-g-900'"
          >
            {{ tab.label }}
          </button>
        </div>
      </div>
      <div class="p-2">
        <ElEmpty v-if="!loading && !hasData" :description="t('dashboard.modelAnalytics.noData')" />
        <ArtLineChart
          v-else-if="analyticsTab === 'trend'"
          :data="chartData.callTrendSeries"
          :x-axis-data="chartData.timeLabels"
          height="20rem"
          :loading="loading"
          show-legend
          show-area-color
        />
        <ArtRingChart
          v-else-if="analyticsTab === 'distribution'"
          :data="chartData.callDistribution"
          height="20rem"
          :loading="loading"
          show-legend
          legend-position="left"
          :show-label="true"
          :radius="['35%', '60%']"
        />
        <ArtBarChart
          v-else
          :data="chartData.callRankingValues"
          :x-axis-data="chartData.callRankingLabels"
          height="20rem"
          :loading="loading"
          show-legend
          legend-position="bottom"
        />
      </div>
    </div>

    <!-- 筛选弹窗 -->
    <ElDialog
      v-model="filterDialogVisible"
      width="32rem"
      class="model-filter-dialog"
    >
      <template #header>
        <div class="flex items-start justify-between pr-8">
          <div>
            <h3 class="text-base font-semibold text-g-900">{{ t('dashboard.modelAnalytics.filterTitle') }}</h3>
            <p class="mt-0.5 text-xs text-g-400">按时间范围和用户筛选模型分析视图</p>
          </div>
        </div>
      </template>

      <div class="flex flex-col gap-4">
        <!-- 快速时间范围 -->
        <div>
          <div class="mb-2 flex items-center gap-1.5 text-sm font-medium text-g-700">
            <ArtSvgIcon icon="ri:calendar-line" class="text-xs" />
            快速范围
          </div>
          <div class="grid grid-cols-4 gap-2">
            <button
              v-for="preset in rangePresets"
              :key="preset.days"
              @click="selectQuickRange(preset.days)"
              class="h-9 rounded-lg border text-sm font-medium transition-all"
              :class="selectedPresetDays === preset.days
                ? 'border-blue-500 bg-blue-500 text-white ring-2 ring-blue-200'
                : 'border-[#e8ecf0] bg-white text-g-600 hover:border-g-300'"
            >
              {{ t(preset.label) }}
            </button>
          </div>
        </div>

        <!-- 分隔线 -->
        <div class="relative">
          <div class="absolute inset-0 flex items-center">
            <span class="w-full border-t border-[#e8ecf0]" />
          </div>
          <div class="relative flex justify-center">
            <span class="bg-white px-3 text-xs text-g-400">自定义时间范围</span>
          </div>
        </div>

        <!-- 起始/结束时间 -->
        <div>
          <label class="mb-1.5 block text-sm text-g-700">起始时间</label>
          <ElDatePicker
            :model-value="draftRange[0]"
            @update:model-value="(val: Date) => { draftRange[0] = val; selectedPresetDays = null }"
            class="w-full"
            type="datetime"
            placeholder="选择起始时间"
            :clearable="false"
          />
        </div>
        <div>
          <label class="mb-1.5 block text-sm text-g-700">结束时间</label>
          <ElDatePicker
            :model-value="draftRange[1]"
            @update:model-value="(val: Date) => { draftRange[1] = val; selectedPresetDays = null }"
            class="w-full"
            type="datetime"
            placeholder="选择结束时间"
            :clearable="false"
          />
        </div>

        <!-- 分隔线 -->
        <div class="relative">
          <div class="absolute inset-0 flex items-center">
            <span class="w-full border-t border-[#e8ecf0]" />
          </div>
          <div class="relative flex justify-center">
            <span class="bg-white px-3 text-xs text-g-400">图表设置</span>
          </div>
        </div>

        <!-- 时间粒度 -->
        <div>
          <label class="mb-1.5 block text-sm text-g-700">时间粒度</label>
          <ElSelect v-model="draftGranularity" style="width: 10rem">
            <ElOption :label="t('dashboard.modelAnalytics.hour')" value="hour" />
            <ElOption :label="t('dashboard.modelAnalytics.day')" value="day" />
            <ElOption :label="t('dashboard.modelAnalytics.week')" value="week" />
          </ElSelect>
        </div>
      </div>

      <template #footer>
        <div class="flex justify-end gap-2">
          <ElButton @click="resetFilters" class="!rounded-lg !border-[#e8ecf0] !bg-white">
            <ArtSvgIcon class="mr-1" icon="ri:reset-left-line" />
            {{ t('dashboard.modelAnalytics.reset') }}
          </ElButton>
          <ElButton type="primary" @click="applyFilters" class="!rounded-lg">
            <ArtSvgIcon class="mr-1" icon="ri:search-line" />
            应用筛选器
          </ElButton>
        </div>
      </template>
    </ElDialog>
  </div>
</template>

<script setup lang="ts">
  import { fetchSystemStatus } from '@/api/auth'
  import { fetchUserQuotaData, type QuotaDataItem } from '@/api/dashboard'
  import { useUserStore } from '@/store/modules/user'
  import { formatTokenMetric } from '@/utils/number'
  import { formatQuotaUsd, resolveQuotaPerUnit } from '@/utils/quota'
  import { ElMessage } from 'element-plus'
  import { useI18n } from 'vue-i18n'
  import {
    buildAnalyticsChartData,
    createRecentFilters,
    formatMetric,
    summarizeAnalytics,
    type AnalyticsFilters,
    type TimeGranularity
  } from './model-analytics'

  defineOptions({ name: 'ModelAnalytics' })

  const { t, locale } = useI18n()
  const userStore = useUserStore()
  const loading = ref(false)
  const filterDialogVisible = ref(false)
  const quotaData = ref<QuotaDataItem[]>([])
  const quotaPerUnit = ref(500000)
  const filters = ref<AnalyticsFilters>(createRecentFilters(1))
  const draftRange = ref<[Date, Date]>([
    new Date(filters.value.startTimestamp * 1000),
    new Date(filters.value.endTimestamp * 1000)
  ])
  const draftGranularity = ref<TimeGranularity>('hour')
  const selectedPresetDays = ref<number | null>(1)
  const quotaChartMode = ref<'bar' | 'area'>('bar')
  const analyticsTab = ref<'trend' | 'distribution' | 'ranking'>('trend')
  const rangePresets = [
    { days: 1, label: 'dashboard.modelAnalytics.oneDay' },
    { days: 7, label: 'dashboard.modelAnalytics.sevenDays' },
    { days: 14, label: 'dashboard.modelAnalytics.fourteenDays' },
    { days: 29, label: 'dashboard.modelAnalytics.twentyNineDays' }
  ] as const

  const isAdministrator = computed(() =>
    (userStore.info.roles || []).some((role) => role === 'R_SUPER' || role === 'R_ADMIN')
  )
  const summary = computed(() => summarizeAnalytics(quotaData.value, filters.value))
  const chartData = computed(() =>
    buildAnalyticsChartData(
      quotaData.value,
      filters.value.granularity,
      locale.value,
      quotaPerUnit.value
    )
  )
  const hasData = computed(() => quotaData.value.length > 0)
  const totalQuotaLabel = computed(() =>
    formatQuotaUsd(summary.value.totalQuota, quotaPerUnit.value)
  )
  const currentRangeLabel = computed(() => {
    const formatter = new Intl.DateTimeFormat(locale.value, {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit'
    })
    return `${formatter.format(filters.value.startTimestamp * 1000)} - ${formatter.format(filters.value.endTimestamp * 1000)}`
  })
  const statCards = computed(() => [
    {
      title: t('dashboard.modelAnalytics.totalCount'),
      value: formatMetric(summary.value.totalCount, locale.value),
      description: t('dashboard.modelAnalytics.totalCountHint'),
      icon: 'ri:hashtag',
      iconBadge: 'bg-blue-50 text-blue-500'
    },
    {
      title: t('dashboard.modelAnalytics.totalQuota'),
      value: totalQuotaLabel.value,
      description: t('dashboard.modelAnalytics.totalQuotaHint'),
      icon: 'ri:coins-line',
      iconBadge: 'bg-emerald-50 text-emerald-500'
    },
    {
      title: t('dashboard.modelAnalytics.totalTokens'),
      value: formatTokenMetric(summary.value.totalTokens, locale.value),
      description: t('dashboard.modelAnalytics.totalTokensHint'),
      icon: 'ri:stack-line',
      iconBadge: 'bg-violet-50 text-violet-500'
    },
    {
      title: t('dashboard.modelAnalytics.averageRpm'),
      value: formatMetric(summary.value.averageRpm, locale.value, 3),
      description: t('dashboard.modelAnalytics.averageRpmHint'),
      icon: 'ri:speed-up-line',
      iconBadge: 'bg-amber-50 text-amber-500'
    },
    {
      title: t('dashboard.modelAnalytics.averageTpm'),
      value: formatTokenMetric(summary.value.averageTpm, locale.value),
      description: t('dashboard.modelAnalytics.averageTpmHint'),
      icon: 'ri:flashlight-line',
      iconBadge: 'bg-rose-50 text-rose-500'
    }
  ])

  /**
   * 加载当前筛选范围内的模型调用数据
   * @returns 无返回值
   */
  const loadAnalytics = async (): Promise<void> => {
    loading.value = true
    try {
      const [data, status] = await Promise.all([
        fetchUserQuotaData(
          {
            start_timestamp: filters.value.startTimestamp,
            end_timestamp: filters.value.endTimestamp,
            default_time: filters.value.granularity
          },
          isAdministrator.value
        ),
        fetchSystemStatus()
      ])
      quotaData.value = data
      quotaPerUnit.value = resolveQuotaPerUnit(status.quota_per_unit)
    } catch {
      quotaData.value = []
      ElMessage.error(t('dashboard.modelAnalytics.loadFailed'))
    } finally {
      loading.value = false
    }
  }

  /**
   * 打开筛选弹窗并同步当前生效条件
   * @returns 无返回值
   */
  const openFilterDialog = (): void => {
    draftRange.value = [
      new Date(filters.value.startTimestamp * 1000),
      new Date(filters.value.endTimestamp * 1000)
    ]
    draftGranularity.value = filters.value.granularity
    filterDialogVisible.value = true
  }

  /**
   * 选择快捷时间范围并匹配推荐聚合粒度
   * @param days 最近天数
   */
  const selectQuickRange = (days: number): void => {
    const recentFilters = createRecentFilters(days)
    draftRange.value = [
      new Date(recentFilters.startTimestamp * 1000),
      new Date(recentFilters.endTimestamp * 1000)
    ]
    draftGranularity.value = recentFilters.granularity
    selectedPresetDays.value = days
  }

  /**
   * 应用弹窗中的筛选条件并重新加载数据
   */
  const applyFilters = (): void => {
    filters.value = {
      startTimestamp: Math.floor(draftRange.value[0].getTime() / 1000),
      endTimestamp: Math.floor(draftRange.value[1].getTime() / 1000),
      granularity: draftGranularity.value
    }
    filterDialogVisible.value = false
    void loadAnalytics()
  }

  /**
   * 恢复默认最近一天筛选并重新加载数据
   */
  const resetFilters = (): void => {
    filters.value = createRecentFilters(1)
    selectedPresetDays.value = 1
    filterDialogVisible.value = false
    void loadAnalytics()
  }

  onMounted(loadAnalytics)
</script>

<style scoped>
  :deep(.model-filter-dialog) {
    border-radius: 12px;
    overflow: hidden;
  }
  :deep(.model-filter-dialog .el-dialog__header) {
    padding: 16px 20px 12px;
    margin-right: 0;
    border-bottom: 1px solid #e8ecf0;
  }
  :deep(.model-filter-dialog .el-dialog__title) {
    font-size: 15px;
    font-weight: 600;
    color: #1a1a2e;
  }
  :deep(.model-filter-dialog .el-dialog__body) {
    padding: 16px 20px;
    color: #4d5875;
    font-size: 13px;
  }
  :deep(.model-filter-dialog .el-dialog__footer) {
    padding: 12px 20px 16px;
    border-top: 1px solid #e8ecf0;
  }
  :deep(.model-filter-dialog .el-button) {
    border-radius: 8px;
    font-weight: 500;
  }
</style>
