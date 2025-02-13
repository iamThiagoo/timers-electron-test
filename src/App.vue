<template>
  <div class="min-h-screen bg-gray-100 p-4">
    <div class="mx-auto max-w-[1600px] grid gap-8 lg:grid-cols-2">
      <div v-for="oven in ovens" :key="oven.id" class="rounded-lg bg-white p-4 shadow-lg">
        <h2 class="mb-4 text-center text-2xl font-bold text-gray-900">
          {{ oven.name }}
        </h2>
        <div class="grid gap-3">
          <div
            v-for="tray in oven.trays"
            :key="tray.id"
            class="grid gap-2 items-center rounded-lg border p-2"
          >
            <div class="flex justify-between">
              <div class="text-sm font-semibold text-gray-700">N°{{ tray.id }}</div>
              <div class="flex flex-col">
                <input
                  v-if="tray.isEditing"
                  type="time"
                  step="1"
                  v-model="tray.editTime"
                  @blur="handleTimeChange(oven.id, tray.id, tray.editTime)"
                  class="w-40 text-lg border rounded px-2 py-1"
                />
                <div
                  v-else
                  :class="{
                    'text-xl font-bold': true,
                    'text-red-500': tray.time === 0,
                    'text-green-500': tray.isRunning,
                    'text-gray-700': !tray.isRunning && tray.time > 0
                  }"
                >
                  {{ formatTime(tray.time) }}
                </div>
            </div>
          </div>
          <div class="flex justify-between text-xs text-gray-600">
            <span>{{ tray.startTime || "--:--" }}</span>
            <span>{{ tray.endTime || "--:--" }}</span>
          </div>
          <div 
            class="h-1 w-full mt-1 rounded-full overflow-hidden bg-gray-200"
          >
            <div
              :class="getProgressColor(tray.time, tray.initialTime)"
              :style="{ width: `${(tray.time / tray.initialTime) * 100}%` }"
              class="h-full transition-all duration-300 ease-in-out"
            ></div>
          </div>
            <div class="flex gap-1">
              <button
                @click="handleStart(oven.id, tray.id)"
                :disabled="tray.isRunning || tray.time === 0"
                class="h-8 w-8 flex items-center justify-center rounded-full bg-blue-500 text-white disabled:opacity-50"
              >
                <Timer class="h-4 w-4" />
              </button>
              <button
                @click="handlePause(oven.id, tray.id)"
                :disabled="!tray.isRunning"
                class="h-8 w-8 flex items-center justify-center rounded-full bg-yellow-500 text-white disabled:opacity-50"
              >
                <Pause class="h-4 w-4" />
              </button>
              <button
                @click="handleReset(oven.id, tray.id)"
                class="h-8 w-8 flex items-center justify-center rounded-full bg-red-500 text-white"
              >
                <RotateCcw class="h-4 w-4" />
              </button>
              <button
                @click="handleEdit(oven.id, tray.id)"
                :disabled="tray.isRunning"
                class="h-8 w-8 flex items-center justify-center rounded-full bg-gray-300 text-gray-700 disabled:opacity-50"
              >
                <Edit2 class="h-4 w-4" />
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>

import { ref, onMounted, onUnmounted, watch } from 'vue'
import { Timer, Pause, RotateCcw, Edit2 } from 'lucide-vue-next'
let interval
const STORAGE_KEY = "ovens_data";
const editTime = ref("");

const loadStoredData = () => {
  const stored = localStorage.getItem(STORAGE_KEY);
  if (stored) {
    return JSON.parse(stored);
  }
  return null;
};

const ovens = ref(loadStoredData() || [
  {
    id: 1,
    name: "FORNO 1",
    trays: Array.from({ length: 8 }, (_, i) => ({
      id: i + 1,
      time: 0.5 * 60,
      initialTime: 0.5 * 60,
      isRunning: false,
      startTime: null,
      endTime: null,
      isEditing: false,
    })),
  },
  {
    id: 2,
    name: "FORNO 2",
    trays: Array.from({ length: 8 }, (_, i) => ({
      id: i + 1,
      time: 0.5 * 60,
      initialTime: 0.5 * 60,
      isRunning: false,
      startTime: null,
      endTime: null,
      isEditing: false,
    })),
  },
]);

onMounted(() => {
  interval = setInterval(() => {
    ovens.value = ovens.value.map(oven => ({
      ...oven,
      trays: oven.trays.map(tray => ({
        ...tray,
        time: tray.isRunning && tray.time > 0 ? tray.time - 1 : tray.time,
      })),
    }))
  }, 1000)
})

onUnmounted(() => {
  clearInterval(interval)
})

watch(ovens, (newOvens) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(newOvens));
}, { deep: true });

const formatTime = (seconds) => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const remainingSeconds = seconds % 60;

  return `${hours.toString().padStart(2, "0")}:${minutes.toString().padStart(2, "0")}:${remainingSeconds.toString().padStart(2, "0")}`;
};

const getCurrentTime = () => {
  return new Date().toLocaleTimeString("pt-BR", { hour: "2-digit", minute: "2-digit" })
}

const getEstimatedEndTime = (startTime, durationInSeconds) => {
  const [hours, minutes] = startTime.split(":").map(Number)
  const endTime = new Date()
  endTime.setHours(hours)
  endTime.setMinutes(minutes + Math.ceil(durationInSeconds / 60))
  return endTime.toLocaleTimeString("pt-BR", { hour: "2-digit", minute: "2-digit" })
}

const handleStart = (ovenId, trayId) => {
  const currentTime = getCurrentTime()
  ovens.value = ovens.value.map(oven =>
    oven.id === ovenId
      ? {
          ...oven,
          trays: oven.trays.map(tray =>
            tray.id === trayId
              ? {
                  ...tray,
                  isRunning: true,
                  startTime: currentTime,
                  endTime: getEstimatedEndTime(currentTime, tray.time),
                  isEditing: false,
                }
              : tray
          ),
        }
      : oven
  )
}

const handlePause = (ovenId, trayId) => {
  ovens.value = ovens.value.map(oven =>
    oven.id === ovenId
      ? {
          ...oven,
          trays: oven.trays.map(tray =>
            tray.id === trayId ? { ...tray, isRunning: false } : tray
          ),
        }
      : oven
  )
}

const handleReset = (ovenId, trayId) => {
  ovens.value = ovens.value.map(oven =>
    oven.id === ovenId
      ? {
          ...oven,
          trays: oven.trays.map(tray =>
            tray.id === trayId
              ? {
                  ...tray,
                  time: tray.initialTime,
                  isRunning: false,
                  startTime: null,
                  endTime: null,
                }
              : tray
          ),
        }
      : oven
  )
}
const handleEdit = (ovenId, trayId) => {
  console.log(ovenId, trayId);
  ovens.value = ovens.value.map(oven => ({
    ...oven,
    trays: oven.trays.map(tray => ({
      ...tray,
      isEditing: tray.id === trayId && oven.id === ovenId,
      editTime: tray.id === trayId && oven.id === ovenId ? formatTime(tray.time) : undefined
    }))
  }));
};

const handleTimeChange = (ovenId, trayId, newTime) => {
  if (!newTime) return;

  const [hours, minutes, seconds] = newTime.split(':').map(Number);
  const totalSeconds = (hours * 3600) + (minutes * 60) + (seconds || 0);

  ovens.value = ovens.value.map(oven => ({
    ...oven,
    trays: oven.trays.map(tray => ({
      ...tray,
      time: tray.id === trayId ? totalSeconds : tray.time,
      initialTime: tray.id === trayId ? totalSeconds : tray.initialTime,
      isEditing: false,
      editTime: undefined
    }))
  }));
};


const getProgressColor = (current, total) => {
  const percentage = (current / total) * 100
  if (percentage > 80) return "bg-emerald-500"
  if (percentage > 60) return "bg-green-400"
  if (percentage > 40) return "bg-yellow-500"
  if (percentage > 20) return "bg-red-400"
  return "bg-red-800"
}

</script>