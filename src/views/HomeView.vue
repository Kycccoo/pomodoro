<template>
  <v-container>
    <v-row class="main mt-5">
      <v-col cols="10">
        <table id="tablex">
          <tr v-for="minute in 25" :key="minute">
            <td
  v-for="sec in 60"
  :key="sec"
  :ref="`cell-${minute}-${sec}`"
  :class="{ 'timm': true }"
  :style="tdStyles[(25 - minute) * 60 + sec - 1]"
>
  <!-- {{ timers(sec) }} -->
</td>
          </tr>
        </table>
      </v-col>
      <v-col cols="2" class="text-area">
        <v-col cols="12">
          <h1>待辦事項</h1>
        </v-col>
        <v-col cols="12">
          <h1 style="text-decoration:underline;">{{ currentText }}</h1>
        </v-col>
        <v-col cols="12">
          <h1>剩餘時間</h1>
        </v-col>
        <v-col cols="12">
          <h1 style="text-decoration:underline;">{{ currentTime }}</h1>
        </v-col>
      </v-col>
      <v-col cols="12" class="button-area">
        <v-btn
          variant="text"
          icon="mdi-play"
          :disabled="status === STATUS.COUNTING || (currentItem.length === 0 && items.length === 0)"
          @click="startTimer"
          color="#6e6e73"
          border="1px solid black"
        ></v-btn>
        <v-btn
          variant="text"
          icon="mdi-pause"
          :disabled="status !== STATUS.COUNTING"
          @click="pauseTimer"
          color="#6e6e73"
        ></v-btn>
        <v-btn
          variant="text"
          icon="mdi-skip-next"
          :disabled="currentItem.length === 0"
          @click="finishTimer"
          color="#6e6e73"
        ></v-btn>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { useListStore } from '@/store/list'
import { useSettingsStore } from '@/store/settings'
import { storeToRefs } from 'pinia'
import { ref, computed } from 'vue'
import { useWebNotification } from '@vueuse/core'

const list = useListStore()
const { currentItem, items, timeleft } = storeToRefs(list)
const { setCurrentItem, countdown, setFinishedItem } = list

const settings = useSettingsStore()
const { selectedAlarmFile, notify } = storeToRefs(settings)

const timersArray = ref(Array.from({ length: 26 }, () => Array(60).fill(false)))

// 0 = 停止
// 1 = 倒數中
// 2 = 暫停
const STATUS = {
  STOP: 0,
  COUNTING: 1,
  PAUSE: 2
}

const status = ref(STATUS.STOP)
let timer = 0

const startTimer = () => {
  if (status.value === STATUS.STOP && items.value.length > 0) {
    setCurrentItem()
  }

  status.value = STATUS.COUNTING

  timer = setInterval(() => {
    countdown()

    // 設置當前秒數對應的 td 元素的狀態為 true
    const minute = Math.floor(timeleft.value / 60)
    const sec = timeleft.value % 60

    timersArray.value[minute][sec] = true

    if (timeleft.value === 0) {
      finishTimer()
    }
  }, 1000)
}

const tdStyles = computed(() => {
  const styles = []
  for (let minute = 0; minute < 26; minute++) {
    for (let sec = 0; sec < 60; sec++) {
      styles.push({
        'background-color': timersArray.value[minute][sec] ? '#fff' : '#216e39'
      })
    }
  }
  return styles
})

const pauseTimer = () => {
  status.value = STATUS.PAUSE
  clearInterval(timer)
}

const finishTimer = () => {
  clearInterval(timer)
  status.value = STATUS.STOP

  const audio = new Audio()
  audio.src = selectedAlarmFile.value
  audio.play()

  if (notify.value) {
    const { show, isSupported } = useWebNotification({
      title: '事項完成',
      body: currentItem.value,
      icon: new URL('@/assets/tomato.png', import.meta.url).href
    })
    if (isSupported.value) {
      show()
    }
  }

  setFinishedItem()

  // 重置 timersArray
  timersArray.value = Array(26).fill().map(() => Array(60).fill(false))

  if (items.value.length > 0) {
    startTimer()
  }
}

const currentText = computed(() => {
  if (currentItem.value.length > 0) {
    return currentItem.value
  } else if (items.value.length > 0) {
    return '點擊開始'
  } else {
    return '沒有事項'
  }
})

const currentTime = computed(() => {
  const m = Math.floor(timeleft.value / 60).toString().padStart(2, '0')
  const s = (timeleft.value % 60).toString().padStart(2, '0')
  return m + ':' + s
})

</script>

<style scoped>
body {
  margin: 0;
  padding: 0;
}
.main {
  text-align: center;
  height: 80vh;
  display: flex;
  align-items: center;
  align-content: space-between;
}
.text-area {
  align-items: center;
}
.text-area .v-col {
  color: #6e6e73;
  display: flex;
  justify-content: end;
  flex-wrap: wrap;
}
.v-btn {
  font-size: 3rem;
  margin: 0 5rem;
}
table {
  border-collapse: separate;
  width: 100%;
  height: 70vh;
}
td {
  border: 1px solid #ddd;
  width: 15px;
  height: 15px;
  box-sizing: border-box;
  border-radius: 20%;
  font-family: "Black Ops One";
}
</style>
