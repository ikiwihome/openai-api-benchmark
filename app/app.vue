<template>
<div class="min-h-screen">
  <div class="theme-toggle">
    <Button variant="ghost" size="icon" @click="toggleDarkMode" class="no-hover">
      <Sun v-if="colorMode.value !== 'dark'" class="h-5 w-5" />
      <Moon v-else class="h-5 w-5" />
    </Button>
  </div>
  <NuxtPage />
</div>
</template>

<script setup>
import { Sun, Moon } from 'lucide-vue-next'
const colorMode = useColorMode()

// 使用 colorMode 统一管理主题状态
function toggleDarkMode() {
  colorMode.preference = colorMode.value === 'dark' ? 'light' : 'dark'
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
.theme-toggle {
  position: relative;
  z-index: 10;
}
</style>