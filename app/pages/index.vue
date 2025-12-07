<template>
<div>
  <Toaster position="bottom-right" />
  <div v-if="!isHydrated" class="container mx-auto py-4 px-4 text-center text-muted-foreground">
    加载本地配置...
  </div>
  <div v-else class="container mx-auto py-4 px-4">
    <input type="file" ref="fileInput" accept=".json,application/json" class="hidden" @change="handleImportFile" />
  <div class="space-y-8">
    <!-- 标题区域 -->
    <div class="space-y-2">
      <h1 class="text-2xl md:text-3xl font-bold tracking-tight">OpenAI API 并发测试工具</h1>
      <p class="text-sm md:text-lg text-muted-foreground">
        测试多家兼容OpenAI API格式的API提供商，测量首字时延和每秒token数
      </p>
      <p class="text-xs md:text-md text-muted-foreground">
        数据安全：本工具所有配置数据均保存在浏览器本地，不会上传到任何服务器
      </p>
    </div>

    <!-- API配置列表 -->
    <div class="space-y-4">
      <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-3 md:gap-2">
        <h2 class="text-lg md:text-xl font-semibold">API配置列表</h2>
        <div class="flex items-center gap-2 flex-wrap">
          <Button @click="openAddDialog" variant="outline" size="sm" class="text-sm cursor-pointer">
            <Plus class="w-4 h-4 mr-1 md:mr-2" />
            添加配置
          </Button>
          <Button @click="triggerImport" variant="outline" size="sm" class="text-sm cursor-pointer ">
            <UploadCloud class="w-4 h-4 mr-1 md:mr-2" />
            导入配置
          </Button>
          <Button @click="exportConfigs" variant="outline" size="sm" class="text-sm cursor-pointer">
            <DownloadCloud class="w-4 h-4 mr-1 md:mr-2" />
            导出配置
          </Button>
        </div>
      </div>

      <div class="grid gap-4 grid-cols-1 md:grid-cols-2 lg:grid-cols-3">
        <Card v-for="api in apiConfigs" :key="api.name" class="api-card p-3 md:p-4 transition-all duration-300 border-primary/50 shadow-sm hover:shadow-lg hover:border-primary/80" @click="handleCardClick(api, $event)">
          <CardHeader class="p-0">
            <div class="flex items-start justify-between gap-2 mb-3">
              <CardTitle class="text-base md:text-lg font-medium truncate flex-1" :title="api.name">
                {{ api.name }}
              </CardTitle>
              <div class="flex items-center gap-1 shrink-0">
                <Switch class="cursor-pointer" :checked="api.enabled" :model-value="api.enabled" @update:checked="(val: boolean) => handleEnabledChange(api, val)" @update:modelValue="(val: boolean) => handleEnabledChange(api, val)" @change="(val: boolean) => handleEnabledChange(api, val)" @click="() => handleEnabledChange(api, !api.enabled)" />
              </div>
            </div>
          </CardHeader>
          <CardContent class="p-0 space-y-2 md:space-y-3">
            <div class="grid gap-1 md:gap-1.5 text-xs md:text-sm">
              <div class="flex justify-between gap-2">
                <span class="text-muted-foreground shrink-0">模型ID:</span>
                <span class="font-medium truncate" :title="api.model">{{ api.model }}</span>
              </div>
              <div class="flex justify-between gap-2">
                <span class="text-muted-foreground shrink-0">BaseURL:</span>
                <span class="text-muted-foreground font-medium truncate">
                  {{ api.base_url }}
                </span>
              </div>
            </div>

            <div class="flex items-center gap-1 md:gap-2 pt-2 md:pt-3 border-t">
              <Button variant="ghost" size="icon" class="h-7 md:h-8 w-7 md:w-8 cursor-pointer" @click="openEditDialog(api)" :title="'编辑此配置'">
                <Pencil class="w-3.5 md:w-4 h-3.5 md:h-4" />
              </Button>
              <Button variant="ghost" size="icon" class="h-7 md:h-8 w-7 md:w-8 cursor-pointer" @click="openDuplicateDialog(api)" :title="'复制此配置'">
                <Copy class="w-3.5 md:w-4 h-3.5 md:h-4" />
              </Button>
              <Button variant="ghost" size="icon" class="h-7 md:h-8 w-7 md:w-8 text-destructive hover:text-destructive cursor-pointer" @click="openDeleteDialog(api.name)" :title="'删除此配置'">
                <Trash2 class="w-3.5 md:w-4 h-3.5 md:h-4" />
              </Button>
            </div>
          </CardContent>
        </Card>
      </div>
    </div>

    <!-- 添加/编辑配置对话框 -->
    <Dialog :open="isDialogOpen" @update:open="isDialogOpen = $event">
      <DialogContent class="sm:max-w-[425px]">
        <DialogHeader>
          <DialogTitle>{{ isDuplicating ? '复制配置' : (isEditing ? '编辑配置' : '添加配置') }}</DialogTitle>
          <DialogDescription>
            配置API提供商的连接信息。Base URL通常以 /v1 结尾。
          </DialogDescription>
        </DialogHeader>
        <div class="grid gap-4 py-4">
          <div class="grid gap-2">
            <Label htmlFor="name">名称</Label>
            <Input id="name" v-model="configForm.name" placeholder="例如: OpenAI" />
          </div>
          <div class="grid gap-2">
            <Label htmlFor="base_url">Base URL</Label>
            <Input id="base_url" v-model="configForm.base_url" placeholder="https://api.openai.com/v1" />
          </div>
          <div class="grid gap-2">
            <Label htmlFor="api_key">API Key</Label>
            <div class="relative flex items-center">
              <Input 
                id="api_key" 
                :model-value="(isEditing || isDuplicating) ? (apiKeyVisible ? configForm.api_key : maskApiKey(configForm.api_key)) : configForm.api_key"
                @update:model-value="(val: any) => configForm.api_key = val"
                placeholder="sk-..." 
                :readonly="(isEditing || isDuplicating) && !apiKeyVisible"
              />
              <button
                v-if="isEditing || isDuplicating"
                type="button"
                @click="apiKeyVisible = !apiKeyVisible"
                class="absolute right-1 top-1/2 -translate-y-1/2 p-1 rounded bg-muted text-muted-foreground hover:text-foreground hover:bg-muted/80 transition-colors"
                :title="apiKeyVisible ? '隐藏API Key' : '显示API Key'"
              >
                <Eye v-if="apiKeyVisible" class="w-4 h-4" />
                <EyeOff v-else class="w-4 h-4" />
              </button>
            </div>
          </div>
          <div class="grid gap-2">
            <Label htmlFor="model">模型ID</Label>
            <Input id="model" v-model="configForm.model" placeholder="gpt-3.5-turbo" />
          </div>
          <div class="flex items-center justify-between">
            <Label htmlFor="enabled">启用</Label>
            <Switch v-model="configForm.enabled" />
          </div>
          <!-- BaseURL测试结果提示 -->
          <div v-if="testBaseURLStatus !== null" :class="['p-3 rounded-md text-sm', testBaseURLStatus.success ? 'bg-green-50 text-green-800' : 'bg-red-50 text-red-800']">
            <div class="font-medium">{{ testBaseURLStatus.message }}</div>
            <div v-if="testBaseURLStatus.details" class="text-xs mt-1 opacity-80">{{ testBaseURLStatus.details }}</div>
          </div>
        </div>
        <DialogFooter class="flex gap-2 pt-4">
          <Button variant="outline" @click="testBaseURL" :disabled="!configForm.base_url || !configForm.api_key || isTestingBaseURL" class="flex items-center gap-2">
            <Spinner v-if="isTestingBaseURL" />
            {{ isTestingBaseURL ? '测试中...' : '测试' }}
          </Button>
          <Button variant="outline" @click="isDialogOpen = false">取消</Button>
          <Button @click="saveConfig">保存</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>

    <!-- 删除确认对话框 -->
    <Dialog :open="deleteConfirmOpen" @update:open="deleteConfirmOpen = $event">
      <DialogContent class="sm:max-w-[425px]">
        <DialogHeader>
          <DialogTitle>删除配置</DialogTitle>
          <DialogDescription>
            确定要删除配置 <span class="font-semibold text-foreground">{{ deleteTargetName }}</span> 吗？此操作无法撤销。
          </DialogDescription>
        </DialogHeader>
        <DialogFooter class="flex gap-2 pt-4">
          <Button variant="outline" @click="cancelDeleteApi">取消</Button>
          <Button variant="destructive" @click="confirmDeleteApi">删除</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>

    <!-- 测试控制区域 -->
    <Card class="p-3 md:p-6">
      <CardHeader class="p-0 pb-3 md:pb-4">
        <CardTitle class="text-lg md:text-xl">测试控制</CardTitle>
        <CardDescription class="text-xs md:text-sm">
          点击开始按钮将并发测试所有启用的API提供商
        </CardDescription>
      </CardHeader>
      <CardContent class="p-0 space-y-3 md:space-y-4">
        <!-- 测试消息输入 -->
        <div class="space-y-2">
          <Label for="test-message" class="text-xs md:text-sm">测试消息</Label>
          <Textarea id="test-message" v-model="testMessage" placeholder="输入要发送的测试消息..." class="min-h-16 md:min-h-20 text-xs md:text-sm resize-none" />
        </div>

        <!-- 最大输出Token数 -->
        <div class="space-y-2">
          <Label for="max-tokens" class="text-xs md:text-sm">最大输出Token数</Label>
          <Input 
            id="max-tokens" 
            v-model.number="maxTokens" 
            type="number" 
            min="10" 
            max="2000" 
            placeholder="500" 
            class="w-full text-xs md:text-sm"
            @input="(e: Event) => {
              const value = parseInt((e.target as HTMLInputElement).value)
              if (value < 10) maxTokens = 10
              else if (value > 2000) maxTokens = 2000
            }"
          />
          <p class="text-xs text-muted-foreground">范围: 10 - 2000</p>
        </div>

        <!-- 控制按钮 -->
        <div class="flex gap-3">
          <Button @click="startTesting" :disabled="isTesting || !hasEnabledApis" class="flex-1 flex items-center justify-center gap-2 text-xs md:text-sm cursor-pointer">
            <Spinner v-if="isTesting" class="h-3.5 w-3.5 md:h-4 md:w-4" />
            {{ isTesting ? '测试中...' : '开始测试' }}
          </Button>
        </div>
      </CardContent>
    </Card>

    <!-- 测试结果 -->
    <Card class="p-3 md:p-6">
      <CardHeader class="p-0 pb-3 md:pb-4">
        <CardTitle class="text-lg md:text-xl">测试结果</CardTitle>
        <CardDescription class="text-xs md:text-sm">
          各API提供商的性能测试结果
        </CardDescription>
      </CardHeader>
      <CardContent class="p-0">
        <div class="overflow-x-auto">
          <!-- 测试结果的表格区域 -->
          <TestResultTable
            v-if="isTesting"
            :title="'当前测试（进行中）'"
            :results="testResults"
            :getResultBadgeVariant="getResultBadgeVariant"
            :getResultStatusText="getResultStatusText"
          />

          <div v-if="!testHistory || testHistory.length === 0" class="p-4 text-muted-foreground">暂无测试历史</div>

          <div>
            <TestResultTable
              v-for="run in testHistory"
              :key="run.runAt"
              :title="'测试时间：' + new Date(run.runAt).toLocaleString()"
              :results="run.results"
              :runAt="run.runAt"
              :getResultBadgeVariant="getResultBadgeVariant"
              :getResultStatusText="getResultStatusText"
            />
          </div>
        </div>
      </CardContent>
    </Card>
  </div>
  </div>
</div>
</template>

<script setup
  lang="ts">
  import { ref, computed, onMounted, watch } from 'vue'
  import { useStorage } from '@vueuse/core'
  import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '~/components/ui/card'
  import { Button } from '~/components/ui/button'
  import { Label } from '~/components/ui/label'
  import { Textarea } from '~/components/ui/textarea'
  import { Switch } from '~/components/ui/switch'
  import {
    Dialog,
    DialogContent,
    DialogDescription,
    DialogFooter,
    DialogHeader,
    DialogTitle,
  } from '~/components/ui/dialog'
  import { Input } from '~/components/ui/input'
  import { Plus, Pencil, Trash2, UploadCloud, DownloadCloud, Copy, Eye, EyeOff } from 'lucide-vue-next'
  import { Spinner } from '~/components/ui/spinner'
  import { Toaster } from '~/components/ui/sonner'
  import { toast } from 'vue-sonner'
  import { encode } from 'gpt-tokenizer'
  import TestResultTable from '~/components/TestResultTable.vue'

  // API配置类型
  interface ApiConfig {
    name: string
    base_url: string
    api_key: string
    model: string
    enabled: boolean
  }

  // 测试结果类型
  interface TestResult {
    name: string
    model: string
    status: 'pending' | 'testing' | 'success' | 'error' | 'disabled'
    timeToFirstToken: number | null
    tokensPerSecond: number | null
  }

  // 状态管理
  // 使用 localStorage 持久化配置和测试历史
  const apiConfigs = useStorage<ApiConfig[]>('apiConfigs', [] as ApiConfig[])
  const testResults = ref<TestResult[]>([])
  const testHistory = useStorage<{ runAt: string; results: TestResult[] }[]>('testHistory', [] as { runAt: string; results: TestResult[] }[])
  const isTesting = ref(false)
  const testMessage = ref('Explain how large language models work and their main application scenarios.')
  const maxTokens = ref(500) // 最大输出token数，默认500
  // 客户端 hydration 标志，确保先从 localStorage 恢复再显示页面内容
  const isHydrated = ref(false)
  // 隐藏的文件输入引用，用于导入 JSON 配置
  const fileInput = ref<HTMLInputElement | null>(null)

  // 对话框状态
  const isDialogOpen = ref(false)
  const isEditing = ref(false)
  const isDuplicating = ref(false)
  const editingOriginalName = ref('')
  const deleteConfirmOpen = ref(false)
  const deleteTargetName = ref('')
  const configForm = ref<ApiConfig>({
    name: '',
    base_url: '',
    api_key: '',
    model: '',
    enabled: true
  })
  const isTestingBaseURL = ref(false)
  const testBaseURLStatus = ref<{ success: boolean; message: string; details?: string } | null>(null)
  const apiKeyVisible = ref(false)

  // 加载API配置（从 localStorage）
  const loadApiConfigs = () => {
    // useStorage 会自动从 localStorage 恢复数据；若无数据则使用默认空数组
    initializeTestResults()
  }

  // 显示 Toast 提示
  const showToast = (type: 'success' | 'error', message: string) => {
    if (type === 'success') {
      toast.success(message)
    } else {
      toast.error(message)
    }
  }

  // 复制配置到剪贴板
  const copyConfigToClipboard = async (api: ApiConfig) => {
    const configText = `name: ${api.name}
base_url: ${api.base_url}
api_key: ${api.api_key}
model: ${api.model}`

    try {
      await navigator.clipboard.writeText(configText)
      showToast('success', `已复制配置 "${api.name}" 到剪贴板`)
    } catch (err) {
      console.error('复制失败:', err)
      showToast('error', '复制到剪贴板失败')
    }
  }

  // 处理卡片点击事件
  const handleCardClick = (api: ApiConfig, event: MouseEvent) => {
    // 检查是否点击在按钮或开关上
    const target = event.target as HTMLElement
    if (target.closest('button') || target.closest('[role="switch"]')) {
      return
    }
    // 执行复制操作
    copyConfigToClipboard(api)
  }

  // 保存所有配置到本地存储
  const saveAllConfigs = async () => {
    // useStorage 自动将 apiConfigs 变化持久化到 localStorage
    return true
  }

  // 初始化测试结果
  const initializeTestResults = () => {
    // 只显示启用的配置
    const enabledApis = apiConfigs.value.filter(api => api.enabled)
    testResults.value = enabledApis.map((api: ApiConfig) => ({
      name: api.name,
      model: api.model,
      status: 'pending',
      timeToFirstToken: null,
      tokensPerSecond: null,
    }))
  }

  // 监视 apiConfigs 的变化，确保在数据变化后更新测试结果和按钮状态
  // 注意：不要在服务端立即执行（避免 SSR 与客户端不一致），onMounted 会负责首次恢复
  watch(apiConfigs, () => {
    initializeTestResults()
  }, { deep: true })

  // 添加深层监听，确保 enabled 状态的任何变化都能实时同步
  watch(() => apiConfigs.value.map(api => api.enabled), () => {
    // 当任何配置的 enabled 状态改变时，实时更新 localStorage
    saveAllConfigs()
  }, { deep: true })

  // 重置表单
  const resetForm = () => {
    configForm.value = {
      name: '',
      base_url: '',
      api_key: '',
      model: '',
      enabled: true
    }
    editingOriginalName.value = ''
    isEditing.value = false
    isDuplicating.value = false
    testBaseURLStatus.value = null
    apiKeyVisible.value = false
  }

  // 测试BaseURL连接
  const testBaseURL = async () => {
    if (!configForm.value.base_url || !configForm.value.api_key) {
      testBaseURLStatus.value = {
        success: false,
        message: '请输入Base URL和API Key'
      }
      return
    }

    isTestingBaseURL.value = true
    testBaseURLStatus.value = null

    try {
      // 测试连接 - 使用models列表接口
      const response = await fetch(`${configForm.value.base_url}/models`, {
        method: 'GET',
        headers: {
          'Authorization': `Bearer ${configForm.value.api_key}`
        }
      })

      if (response.ok) {
        const data = await response.json()
        testBaseURLStatus.value = {
          success: true,
          message: '✓ BaseURL连接成功',
          details: `获取到 ${data.data?.length || 0} 个模型`
        }
      } else {
        const errorText = await response.text().catch(() => response.statusText)
        testBaseURLStatus.value = {
          success: false,
          message: '✗ BaseURL连接失败',
          details: `HTTP ${response.status}: ${errorText || response.statusText}`
        }
      }
    } catch (error) {
      testBaseURLStatus.value = {
        success: false,
        message: '✗ 连接错误',
        details: error instanceof Error ? error.message : '未知错误'
      }
    } finally {
      isTestingBaseURL.value = false
    }
  }

  // 打开添加对话框
  const openAddDialog = () => {
    resetForm()
    isDialogOpen.value = true
  }

  // 打开编辑对话框
  const openEditDialog = (api: ApiConfig) => {
    configForm.value = { ...api }
    editingOriginalName.value = api.name
    isEditing.value = true
    isDuplicating.value = false
    testBaseURLStatus.value = null
    apiKeyVisible.value = false
    isDialogOpen.value = true
  }

  // 复制配置
  const openDuplicateDialog = (api: ApiConfig) => {
    // 生成新的唯一名称
    let newName = `${api.name} (复制)`
    let counter = 1
    while (apiConfigs.value.some(config => config.name === newName)) {
      newName = `${api.name} (复制 ${counter})`
      counter++
    }
    
    // 填充表单
    configForm.value = {
      name: newName,
      base_url: api.base_url,
      api_key: api.api_key,
      model: api.model,
      enabled: api.enabled
    }
    editingOriginalName.value = ''
    isEditing.value = false
    isDuplicating.value = true
    testBaseURLStatus.value = null
    isDialogOpen.value = true
  }

  // 保存配置 (从对话框)
  const saveConfig = () => {
    if (!configForm.value.name || !configForm.value.base_url) return

    const newConfig: ApiConfig = { ...configForm.value }

    if (isEditing.value) {
      // 编辑现有配置
      const index = apiConfigs.value.findIndex(api => api.name === editingOriginalName.value)
      if (index !== -1) {
        // 使用 splice 确保 Vue 能够检测到数组变化
        apiConfigs.value.splice(index, 1, newConfig)
      }
    } else {
      // 添加新配置
      // 检查名称是否重复
      if (apiConfigs.value.some(api => api.name === newConfig.name)) {
        showToast('error', '配置名称已存在，请使用其他名称')
        return
      }
      apiConfigs.value.push(newConfig)
    }

    // useStorage 自动同步到 localStorage
    isDialogOpen.value = false
    initializeTestResults()
  }

  // 切换启用状态
  const handleEnabledChange = (api: ApiConfig, enabled: boolean) => {
    // 找到对应的配置并更新
    const index = apiConfigs.value.findIndex(item => item.name === api.name)
    if (index !== -1) {
      const original = apiConfigs.value[index]
      if (original) {
        // 使用 Object.assign 确保触发响应式更新
        const updatedConfig: ApiConfig = {
          name: original.name,
          base_url: original.base_url,
          api_key: original.api_key,
          model: original.model,
          enabled: enabled
        }
        // 使用 splice 确保 Vue 能够检测到数组变化
        apiConfigs.value.splice(index, 1, updatedConfig)
        // useStorage 自动同步到 localStorage
        initializeTestResults()
      }
    }
  }

  // 删除配置
  const openDeleteDialog = (name: string) => {
    deleteTargetName.value = name
    deleteConfirmOpen.value = true
  }

  // 确认删除配置
  const confirmDeleteApi = () => {
    const name = deleteTargetName.value
    const index = apiConfigs.value.findIndex(api => api.name === name)
    if (index !== -1) {
      // 使用 splice 确保 Vue 能够检测到数组变化
      apiConfigs.value.splice(index, 1)
      // useStorage 自动同步到 localStorage
      initializeTestResults()
    }
    deleteConfirmOpen.value = false
    deleteTargetName.value = ''
  }

  // 取消删除
  const cancelDeleteApi = () => {
    deleteConfirmOpen.value = false
    deleteTargetName.value = ''
  }

  // 计算属性
  const hasEnabledApis = computed(() => {
    return apiConfigs.value.some(api => api.enabled)
  })

  const getResultBadgeVariant = (status: string) => {
    switch (status) {
      case 'success': return 'default'
      case 'error': return 'destructive'
      case 'testing': return 'secondary'
      default: return 'outline'
    }
  }

  const getResultStatusText = (status: string) => {
    switch (status) {
      case 'success': return '成功'
      case 'error': return '失败'
      case 'testing': return '测试中'
      case 'pending': return '待测试'
      case 'disabled': return '已禁用'
      default: return status
    }
  }

  // API Key 加密函数：显示开头4字符***中间星号***结尾4字符
  const maskApiKey = (apiKey: string): string => {
    if (!apiKey) {
      return ''
    }
    if (apiKey.length <= 8) {
      return '*'.repeat(apiKey.length)
    }
    const start = apiKey.slice(0, 4)
    const end = apiKey.slice(-4)
    const middle = '*'.repeat(Math.max(3, apiKey.length - 8))
    return `${start}${middle}${end}`
  }

  // 测试单个API
  const testSingleApi = async (api: ApiConfig, resultIndex: number): Promise<TestResult> => {
    const result: TestResult = {
      name: api.name,
      model: api.model,
      status: 'testing',
      timeToFirstToken: null,
      tokensPerSecond: null,
    }

    try {
      const startTime = performance.now()
      let firstTokenTime: number | null = null

      if (!api.api_key) {
        throw new Error('API Key未配置')
      }

      const response = await fetch(`${api.base_url}/chat/completions`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${api.api_key}`
        },
        body: JSON.stringify({
          model: api.model,
          messages: [
            {
              role: 'user',
              content: testMessage.value
            }
          ],
          stream: true,
          stream_options: {
            include_usage: true
          },
          max_tokens: maxTokens.value
        })
      })

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }

      const reader = response.body?.getReader()
      if (!reader) {
        throw new Error('无法读取响应流')
      }

      const decoder = new TextDecoder()
      let fullResponse = ''
      let tokenCount = 0
      let apiUsageTokens: number | null = null // API返回的准确token数
      let firstTokenReceived = false
      let lastUpdateTime = 0
      const UPDATE_INTERVAL = 100 // 每100ms更新一次UI

      while (true) {
        const { done, value } = await reader.read()
        if (done) break

        const chunk = decoder.decode(value)
        const lines = chunk.split('\n').filter(line => line.trim() !== '')

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6)
            if (data === '[DONE]') continue

            try {
              const parsed = JSON.parse(data)

              // 优先方案1：检查API返回的usage信息（OpenAI会在流结束时返回）
              if (parsed.usage?.completion_tokens) {
                apiUsageTokens = parsed.usage.completion_tokens
              }

              if (!firstTokenReceived && parsed.choices?.[0]?.delta?.content) {
                firstTokenTime = performance.now() - startTime
                firstTokenReceived = true
                
                // 首字到达时立即更新显示
                if (resultIndex >= 0 && resultIndex < testResults.value.length) {
                  const item = testResults.value[resultIndex]
                  if (item) {
                    item.timeToFirstToken = firstTokenTime
                    item.status = 'success'
                  }
                }
              }

              if (parsed.choices?.[0]?.delta?.content) {
                fullResponse += parsed.choices[0].delta.content

                // 节流更新每秒Token数（避免过于频繁的UI更新）
                const currentTime = performance.now() - startTime - (firstTokenTime ?? 0)
                if (currentTime - lastUpdateTime >= UPDATE_INTERVAL) {
                  // 降级方案3：使用gpt-tokenizer本地计算当前已收到内容的token数
                  try {
                    tokenCount = encode(fullResponse).length
                  } catch (e) {
                    // 如果编码失败，使用简单的字符估算（1 token ≈ 4字符）
                    tokenCount = Math.ceil(fullResponse.length / 4)
                  }
                  
                  const currentTps = tokenCount > 0 ? (tokenCount / (currentTime / 1000)) : 0
                  if (resultIndex >= 0 && resultIndex < testResults.value.length) {
                    const item = testResults.value[resultIndex]
                    if (item) {
                      item.tokensPerSecond = currentTps
                    }
                  }
                  lastUpdateTime = currentTime
                }
              }
            } catch (e) {
              console.warn('解析流数据失败:', e)
            }
          }
        }
      }

      const endTime = performance.now()
      const totalTime = endTime - startTime

      // 最终token计数：优先使用API返回的准确值，否则使用本地计算
      let finalTokenCount = tokenCount
      if (apiUsageTokens !== null) {
        finalTokenCount = apiUsageTokens
      } else if (fullResponse) {
        // 流结束后再次精确计算
        try {
          finalTokenCount = encode(fullResponse).length
        } catch (e) {
          finalTokenCount = Math.ceil(fullResponse.length / 4)
        }
      }

      result.status = 'success'
      result.timeToFirstToken = firstTokenTime
      // 流式阶段推理速度：排除等待首token的时间，只计算流式生成阶段的速度
      const streamingTime = firstTokenTime !== null ? (totalTime - firstTokenTime) : totalTime
      result.tokensPerSecond = finalTokenCount > 0 && streamingTime > 0 ? (finalTokenCount / (streamingTime / 1000)) : 0

      // 最终更新UI中的结果
      if (resultIndex >= 0 && resultIndex < testResults.value.length) {
        testResults.value[resultIndex] = { ...result }
      }

    } catch (error) {
      result.status = 'error'
      // 实时更新错误状态
      if (resultIndex >= 0 && resultIndex < testResults.value.length) {
        testResults.value[resultIndex] = { ...result }
      }
    }

    return result
  }

  // 开始测试
  const startTesting = async () => {
    if (isTesting.value || !hasEnabledApis.value) return

    isTesting.value = true

    // 重置测试结果 - 仅包含启用的API
    const enabledApis = apiConfigs.value.filter(api => api.enabled)
    testResults.value = enabledApis.map(api => ({
      name: api.name,
      model: api.model,
      status: 'pending',
      timeToFirstToken: null,
      tokensPerSecond: null,
    }))

    // 并发测试所有启用的API
    const promises = enabledApis.map(async (api, apiIndex) => {
      // 更新状态为测试中
      if (apiIndex !== -1) {
        const item = testResults.value[apiIndex]
        if (item) {
          item.status = 'testing'
        }
      }

      // 执行测试（传递 resultIndex 以实时更新）
      await testSingleApi(api, apiIndex)
    })

    // 等待所有测试完成
    await Promise.allSettled(promises)
    // 将本次测试结果加入历史（按时间）
    try {
      testHistory.value.unshift({ runAt: new Date().toISOString(), results: testResults.value.map(r => ({ ...r })) })
    } catch (e) {
      console.warn('保存测试历史失败:', e)
    }

    isTesting.value = false
  }

  // 组件挂载时优先加载本地配置并标记 hydration 完成后显示页面
  onMounted(() => {
    loadApiConfigs()
    // 小延迟可确保 useStorage 在客户端完成恢复后再渲染（通常同步）
    // 这里不用长延迟，直接设置为已恢复
    isHydrated.value = true
  })

  // 触发选择本地 JSON 文件导入
  const triggerImport = () => {
    fileInput.value?.click()
  }

  // 处理文件导入
  const handleImportFile = async (e: Event) => {
    const files = (e.target as HTMLInputElement).files
    if (!files || files.length === 0) return
    const file = files[0]
    if (!file) return
    try {
      const text = await file.text()
      const parsed = JSON.parse(text)

      let newApis: any[] | undefined
      if (Array.isArray(parsed)) {
        newApis = parsed
      } else if (parsed && Array.isArray(parsed.apis)) {
        newApis = parsed.apis
      }

      if (!newApis) {
        showToast('error', '导入的 JSON 格式不正确，需为数组或包含 `apis` 字段的对象')
        return
      }

      // 增强的类型校验
      const valid = newApis.every(item => 
        item && 
        typeof item.name === 'string' && 
        typeof item.base_url === 'string' &&
        item.name.trim() !== '' &&
        item.base_url.trim() !== ''
      )
      if (!valid) {
        showToast('error', '导入的配置格式错误：每项必须包含有效的 name 和 base_url 字段')
        return
      }

      // 将导入的配置替换当前本地配置
      apiConfigs.value = newApis.map(a => ({
        name: a.name?.trim() || '',
        base_url: a.base_url?.trim() || '',
        api_key: typeof a.api_key === 'string' ? a.api_key.trim() : '',
        model: typeof a.model === 'string' ? a.model.trim() : '',
        enabled: typeof a.enabled === 'boolean' ? a.enabled : true
      }))

      initializeTestResults()
      showToast('success', '导入配置成功')
    } catch (err) {
      console.error('导入配置失败', err)
      showToast('error', '导入配置失败：文件不是有效的 JSON')
    } finally {
      // 清空 input，以便下次选择同一文件仍会触发 change
      if (fileInput.value) fileInput.value.value = ''
    }
  }

  // 导出当前配置（若无则导出仅包含参考 API 配置的模板 JSON）
  const exportConfigs = async () => {
    try {
      let payload: any
      if (apiConfigs.value && apiConfigs.value.length > 0) {
        payload = { apis: apiConfigs.value }
      } else {
        // 若本地没有配置，导出使用固定参考模板
        payload = {
          apis: [
            {
              name: 'OpenAI',
              base_url: 'https://api.openai.com/v1',
              api_key: 'sk-...',
              model: 'gpt-3.5-turbo',
              enabled: true
            },
            {
              name: 'DeepSeek',
              base_url: 'https://api.deepseek.com/v1',
              api_key: 'sk-...',
              model: 'deepseek-chat',
              enabled: true
            }
          ]
        }
      }

      const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = 'api_config_export.json'
      document.body.appendChild(a)
      a.click()
      a.remove()
      URL.revokeObjectURL(url)
      showToast('success', '导出配置成功')
    } catch (e) {
      console.error('导出失败', e)
      showToast('error', '导出配置失败')
    }
  }
</script>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
}

/* API 卡片悬浮效果 */
.api-card {
  border-radius: 8px;
  overflow: hidden;
}

.api-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.12), 0 8px 16px rgba(0, 0, 0, 0.08);
}

/* 平滑过渡 */
.api-card * {
  transition: color 0.3s ease, background-color 0.3s ease;
}
</style>
