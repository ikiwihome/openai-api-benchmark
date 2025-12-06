<template>
  <div>
    <div class="mb-6">
      <div v-if="title || description" class="flex items-center justify-between mb-2">
        <div>
          <h4 v-if="title" :class="['text-md pt-6', title.includes('当前测试') ? 'font-bold' : 'font-medium']">{{ title }}</h4>
          <div v-if="description" class="text-sm text-muted-foreground">{{ description }}</div>
        </div>
        <slot name="extra" />
      </div>
      <!-- 使用 table-layout: fixed 强制固定列宽，避免内容长度不同导致列不对齐 -->
      <div class="w-full overflow-x-auto">
        <Table class="table-fixed" style="table-layout: fixed;">
          <TableHeader>
            <TableRow>
              <TableHead class="w-24 sm:w-32 px-2 py-2 text-sm sm:text-md">提供商</TableHead>
              <TableHead class="hidden sm:table-cell w-48 px-2 py-2 text-sm sm:text-md">模型</TableHead>
              <TableHead class="hidden md:table-cell w-20 px-2 py-2 text-center text-sm sm:text-md">状态</TableHead>
              <TableHead class="w-28 sm:w-32 px-2 py-2 text-right text-sm sm:text-md">首字时延</TableHead>
              <TableHead class="w-28 sm:w-32 px-2 py-2 text-right text-sm sm:text-md">每秒Token数</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            <TableRow v-for="result in results" :key="result.name + (runAt || '')">
              <TableCell class="w-24 sm:w-32 px-2 py-2 font-medium text-sm sm:text-base">{{ result.name }}</TableCell>
              <TableCell class="hidden sm:table-cell w-48 px-2 py-2 text-sm sm:text-md">{{ result.model }}</TableCell>
              <TableCell class="hidden md:table-cell w-20 px-2 py-2">
                <div class="flex items-center justify-center">
                  <Badge 
                    :variant="getResultBadgeVariant(result.status)"
                    :class="getStatusBadgeClass(result.status)"
                    class="text-sm"
                  >
                    {{ getResultStatusText(result.status) }}
                  </Badge>
                </div>
              </TableCell>
              <TableCell class="w-28 sm:w-32 px-2 py-2 text-right text-sm sm:text-md">
                <span v-if="result.timeToFirstToken != null && typeof result.timeToFirstToken === 'number'">
                  {{ result.timeToFirstToken.toFixed(2) }} ms
                </span>
                <span v-else class="text-muted-foreground">-</span>
              </TableCell>
              <TableCell class="w-28 sm:w-32 px-2 py-2 text-right text-sm sm:text-md">
                <span v-if="result.tokensPerSecond != null && typeof result.tokensPerSecond === 'number'">
                  {{ result.tokensPerSecond.toFixed(2) }}
                </span>
                <span v-else class="text-muted-foreground">-</span>
              </TableCell>
            </TableRow>
          </TableBody>
        </Table>
      </div>
    </div>
    <!-- 分割线：如果有 runAt 属性说明是历史表格，添加下边框作为分割线 -->
    <div v-if="runAt" class="border-b border-gray-200"></div>
  </div>
</template>

<script setup lang="ts">
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '~/components/ui/table'
import { Badge } from '~/components/ui/badge'

const props = defineProps<{
  title?: string
  description?: string
  results: Array<{
    name: string
    model: string
    status: string
    timeToFirstToken: number | null
    tokensPerSecond: number | null
  }>
  runAt?: string // 用于历史表格唯一 key
  getResultBadgeVariant: (status: string) => "default" | "secondary" | "destructive" | "outline" | null | undefined
  getResultStatusText: (status: string) => string
}>()

// 根据状态返回对应的 Badge 样式类
const getStatusBadgeClass = (status: string) => {
  switch (status) {
    case 'success':
      return 'bg-green-100 text-green-800 hover:bg-green-200'
    case 'error':
      return 'bg-red-100 text-red-800 hover:bg-red-200'
    case 'testing':
      return 'bg-blue-100 text-blue-800 hover:bg-blue-200'
    default:
      return ''
  }
}
</script>
