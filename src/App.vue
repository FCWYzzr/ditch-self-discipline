<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import Toast from './components/Toast.vue'
import notices from './assets/notice.json'
import dingSound from './assets/ding.wav'

// 状态定义
type Mode = 'focus' | 'break'

// 配置
const DEFAULT_FOCUS_TIME = 25 * 60 // 25分钟
const DEFAULT_BREAK_TIME = 5 * 60 // 5分钟

// 响应式状态
const mode = ref<Mode>('focus')
const timeLeft = ref(DEFAULT_FOCUS_TIME)
const isRunning = ref(false)
const focusTime = ref(DEFAULT_FOCUS_TIME)
const breakTime = ref(DEFAULT_BREAK_TIME)
const showToast = ref(false)
const toastMessage = ref('')
const audio = new Audio(dingSound)
audio.preload = "auto"

// 计时器引用
let timerInterval: number | null = null
let noticeInterval: number | null = null

// 计算属性
const totalTime = computed(() => mode.value === 'focus' ? focusTime.value : breakTime.value)
const progress = computed(() => ((totalTime.value - timeLeft.value) / totalTime.value) * 360)
const formattedTime = computed(() => {
  const minutes = Math.floor(timeLeft.value / 60)
  const seconds = timeLeft.value % 60
  return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`
})
const modeText = computed(() => mode.value === 'focus' ? '专注' : '休息')
const modeColor = computed(() => mode.value === 'focus' ? '#3B82F6' : '#10B981')

// 方法
const startTimer = () => {
  if (!isRunning.value) {
    isRunning.value = true
    timerInterval = window.setInterval(() => {
      if (timeLeft.value > 0) {
        timeLeft.value--
      } else {
        completeCycle()
      }
    }, 1000)
    
    // 如果是专注模式，设置5秒提示计时器
    if (mode.value === 'focus') {
      noticeInterval = window.setInterval(showRandomNotice, 30000)
    }
  }
}

const pauseTimer = () => {
  if (isRunning.value && timerInterval) {
    isRunning.value = false
    clearInterval(timerInterval)
    timerInterval = null
  }
  
  // 暂停时清除提示计时器
  if (noticeInterval) {
    clearInterval(noticeInterval)
    noticeInterval = null
  }
}

const resetTimer = () => {
  pauseTimer()
  timeLeft.value = mode.value === 'focus' ? focusTime.value : breakTime.value
}

const switchMode = () => {
  // 切换模式前清除提示计时器
  if (noticeInterval) {
    clearInterval(noticeInterval)
    noticeInterval = null
  }
  
  mode.value = mode.value === 'focus' ? 'break' : 'focus'
  resetTimer()
}

const completeCycle = () => {
  pauseTimer()
  showNotification(`${modeText.value}时间结束！`)
  switchMode()
}

const showNotification = (message: string) => {
  // 播放音效
  audio.currentTime = 0
  audio.play().catch(error => {
    console.error('Failed to play sound:', error)
  })
  
  // 显示提示
  toastMessage.value = message
  showToast.value = true
  setTimeout(() => {
    showToast.value = false
  }, 3000)
}

const showRandomNotice = () => {
  if (mode.value === 'focus') {
    const elapsedMinutes = Math.floor((focusTime.value - timeLeft.value) / 60)
    const randomIndex = Math.floor(Math.random() * notices.length)
    let notice = notices[randomIndex] || ''
    notice = notice.replace('[time]', elapsedMinutes.toString())
    showNotification(notice)
  }
}

const updateFocusTime = (minutes: number) => {
  focusTime.value = minutes * 60
  if (mode.value === 'focus' && !isRunning.value) {
    timeLeft.value = focusTime.value
  }
}

const updateBreakTime = (minutes: number) => {
  breakTime.value = minutes * 60
  if (mode.value === 'break' && !isRunning.value) {
    timeLeft.value = breakTime.value
  }
}

// 生命周期
onMounted(() => {
  // 从本地存储加载设置
  const savedFocusTime = localStorage.getItem('focusTime')
  const savedBreakTime = localStorage.getItem('breakTime')
  if (savedFocusTime) focusTime.value = parseInt(savedFocusTime)
  if (savedBreakTime) breakTime.value = parseInt(savedBreakTime)
  resetTimer()
})



onUnmounted(() => {
  if (timerInterval) {
    clearInterval(timerInterval)
  }
  if (noticeInterval) {
    clearInterval(noticeInterval)
  }
  // 保存设置到本地存储
  localStorage.setItem('focusTime', focusTime.value.toString())
  localStorage.setItem('breakTime', breakTime.value.toString())
  
  
})
</script>

<template>
  <div class="select-none fixed w-full h-full bg-gradient-to-br from-slate-50 to-slate-100 flex flex-col items-center justify-center">
    <!-- 计时器主体 -->
    <div class="relative w-[40vh] h-[40vh]">
      <!-- 圆形背景 -->
      <svg class="w-full h-full transform -rotate-90" viewBox="0 0 100 100">
        <!-- 背景圆环 -->
        <circle
          cx="50"
          cy="50"
          r="45"
          fill="none"
          stroke="#E5E7EB"
          stroke-width="10"
        />
        <!-- 进度圆环 -->
        <circle
          cx="50"
          cy="50"
          r="45"
          fill="none"
          :stroke="modeColor"
          stroke-width="10"
          stroke-linecap="round"
          :stroke-dasharray="2 * Math.PI * 45"
          :stroke-dashoffset="2 * Math.PI * 45 - (progress / 360) * (2 * Math.PI * 45)"
          class="transition-all duration-300 ease-in-out"
        />
      </svg>
      
      <!-- 计时器内容 -->
      <div class="absolute inset-0 flex flex-col items-center justify-center">
        <!-- 模式显示 -->
        <div class="text-lg font-semibold mb-2 text-gray-700">{{ modeText }}</div>
        
        <!-- 时间显示 -->
        <div class="text-6xl font-bold text-gray-900">{{ formattedTime }}</div>
        
        <!-- 控制按钮 -->
        <div class="flex space-x-4 mt-6">
          <button
            @click="isRunning ? pauseTimer() : startTimer()"
            class="px-6 py-2 rounded-full font-medium transition-all duration-200"
            :class="isRunning ? 'bg-amber-500 hover:bg-amber-600 text-white' : 'bg-blue-500 hover:bg-blue-600 text-white'"
          >
            {{ isRunning ? '暂停' : '开始' }}
          </button>
          <button
            @click="resetTimer"
            class="px-6 py-2 bg-gray-200 hover:bg-gray-300 rounded-full font-medium transition-all duration-200"
          >
            重置
          </button>
        </div>
      </div>
    </div>
    
    <!-- 模式切换 -->
    <div class="mt-8">
      <button
        @click="switchMode"
        class="px-8 py-3 bg-white rounded-full shadow-md hover:shadow-lg transition-all duration-200 font-medium text-gray-700"
      >
        切换到{{ mode === 'focus' ? '休息' : '专注' }}模式
      </button>
    </div>
    
    <!-- 时间设置 -->
    <div class="mt-12 grid grid-cols-2 gap-8">
      <!-- 专注时间设置 -->
      <div class="flex flex-col items-center">
        <div class="text-sm font-medium text-gray-600 mb-2">专注时间 (分钟)</div>
        <div class="flex items-center space-x-2">
          <button
            @click="updateFocusTime(Math.max(1, focusTime / 60 - 1))"
            class="w-8 h-8 rounded-full bg-gray-200 hover:bg-gray-300 flex items-center justify-center font-medium"
          >
            -
          </button>
          <div class="w-16 text-center text-xl font-medium">{{ focusTime / 60 }}</div>
          <button
            @click="updateFocusTime(focusTime / 60 + 1)"
            class="w-8 h-8 rounded-full bg-gray-200 hover:bg-gray-300 flex items-center justify-center font-medium"
          >
            +
          </button>
        </div>
      </div>
      
      <!-- 休息时间设置 -->
      <div class="flex flex-col items-center">
        <div class="text-sm font-medium text-gray-600 mb-2">休息时间 (分钟)</div>
        <div class="flex items-center space-x-2">
          <button
            @click="updateBreakTime(Math.max(1, breakTime / 60 - 1))"
            class="w-8 h-8 rounded-full bg-gray-200 hover:bg-gray-300 flex items-center justify-center font-medium"
          >
            -
          </button>
          <div class="w-16 text-center text-xl font-medium">{{ breakTime / 60 }}</div>
          <button
            @click="updateBreakTime(breakTime / 60 + 1)"
            class="w-8 h-8 rounded-full bg-gray-200 hover:bg-gray-300 flex items-center justify-center font-medium"
          >
            +
          </button>
        </div>
      </div>
    </div>
    
    <!-- 提示组件 -->
    <Toast :show="showToast" :message="toastMessage" />
  </div>
</template>
