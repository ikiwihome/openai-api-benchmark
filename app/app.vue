<template>
<div class="min-h-screen">
  <div class="fixed top-4 right-4 z-50 flex items-center gap-2">
    <Button variant="ghost" size="icon" @click="toggleDarkMode" class="no-hover" :title="colorMode.value === 'dark' ? '切换到浅色模式' : '切换到深色模式'">
      <Sun v-if="colorMode.value !== 'dark'" class="h-5 w-5" />
      <Moon v-else class="h-5 w-5" />
    </Button>
    <Button variant="ghost" size="icon" @click="openGithub" class="no-hover" :title="'在 GitHub 上打开项目'">
      <Icon icon="radix-icons:github-logo" class="h-5 w-5" />
    </Button>
  </div>
  <NuxtPage />
</div>
</template>

<script setup>
import { Icon } from '@iconify/vue'
import { Sun, Moon } from 'lucide-vue-next'
const colorMode = useColorMode()

// 使用 colorMode 统一管理主题状态
function toggleDarkMode() {
  colorMode.preference = colorMode.value === 'dark' ? 'light' : 'dark'
}

// 打开 GitHub 项目
function openGithub() {
  window.open('https://github.com/ikiwihome/openai-api-benchmark', '_blank')
}

onMounted(() => {
  // colorMode 模块会自动处理系统主题偏好
  // 如果需要默认浅色主题，可以在 nuxt.config.ts 中配置
  if (!colorMode.preference) {
    colorMode.preference = 'light'
  }
})
</script>

<style scoped>
/* 样式已内联到模板中的 flex 容器 */
</style>