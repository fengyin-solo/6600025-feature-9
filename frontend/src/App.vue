<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { useCanBusStore } from './store/canbus';
import FrameTable from './components/FrameTable.vue';
import SignalChart from './components/SignalChart.vue';

const store = useCanBusStore();
const showExportMenu = ref(false);
const exportMenuRef = ref<HTMLDivElement | null>(null);

function handleClickOutside(event: MouseEvent) {
  if (exportMenuRef.value && !exportMenuRef.value.contains(event.target as Node)) {
    showExportMenu.value = false;
  }
}

onMounted(() => {
  document.addEventListener('click', handleClickOutside);
});

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
});

function handleLoadDbc() {
  store.loadMockDbc();
  alert(`已加载 DBC 定义: ${store.dbcMessages.size} 条消息`);
}

function handleExport(useFiltered: boolean = false, selectedSignalsOnly: boolean = false) {
  if (selectedSignalsOnly && store.selectedSignals.size === 0) {
    alert('请先在信号趋势图中选择要导出的信号（点击图例）');
    showExportMenu.value = false;
    return;
  }

  const csv = store.exportFrames({ useFiltered, selectedSignalsOnly });
  const blob = new Blob([csv], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  
  let suffix = 'all';
  if (useFiltered && selectedSignalsOnly) {
    suffix = 'filtered_selected';
  } else if (useFiltered) {
    suffix = 'filtered';
  } else if (selectedSignalsOnly) {
    suffix = 'selected';
  }
  
  a.href = url;
  a.download = `can_frames_${suffix}_${Date.now()}.csv`;
  a.click();
  URL.revokeObjectURL(url);
  showExportMenu.value = false;
}
</script>

<template>
  <div class="h-screen flex flex-col bg-gray-900 text-gray-100 overflow-hidden">
    <!-- Header -->
    <header class="flex items-center justify-between px-6 py-3 bg-gray-800 border-b border-gray-700 shrink-0">
      <div class="flex items-center gap-3">
        <div class="w-8 h-8 bg-cyan-600 rounded-lg flex items-center justify-center">
          <svg class="w-5 h-5 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 3v2m6-2v2M9 19v2m6-2v2M5 9H3m2 6H3m18-6h-2m2 6h-2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z" />
          </svg>
        </div>
        <h1 class="text-lg font-bold text-gray-100">CAN 总线数据帧解析与诊断仪</h1>
      </div>

      <div class="flex items-center gap-2">
        <button
          @click="handleLoadDbc"
          class="px-3 py-1.5 bg-gray-700 hover:bg-gray-600 text-gray-200 text-sm rounded transition-colors border border-gray-600"
        >
          加载DBC
        </button>
        <button
          @click="store.isCapturing ? store.stopCapture() : store.startCapture()"
          class="px-3 py-1.5 text-sm rounded transition-colors font-medium"
          :class="store.isCapturing
            ? 'bg-red-600 hover:bg-red-700 text-white'
            : 'bg-green-600 hover:bg-green-700 text-white'"
        >
          {{ store.isCapturing ? '停止捕获' : '开始捕获' }}
        </button>
        <button
          @click="store.clearFrames()"
          class="px-3 py-1.5 bg-gray-700 hover:bg-gray-600 text-gray-200 text-sm rounded transition-colors border border-gray-600"
        >
          清除
        </button>
        <div class="relative" ref="exportMenuRef">
          <button
            @click="showExportMenu = !showExportMenu"
            class="px-3 py-1.5 bg-gray-700 hover:bg-gray-600 text-gray-200 text-sm rounded transition-colors border border-gray-600 flex items-center gap-1"
          >
            导出CSV
            <svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
            </svg>
          </button>
          <div
            v-if="showExportMenu"
            class="absolute right-0 top-full mt-1 w-60 bg-gray-800 border border-gray-600 rounded shadow-lg z-50 py-1"
          >
            <button
              @click="handleExport(false, false)"
              class="w-full text-left px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 transition-colors"
            >
              <div class="font-medium">导出全部帧</div>
              <div class="text-xs text-gray-400">导出所有 {{ store.frames.length }} 条帧数据</div>
            </button>
            <button
              @click="handleExport(true, false)"
              class="w-full text-left px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 transition-colors"
            >
              <div class="font-medium">导出筛选结果</div>
              <div class="text-xs text-gray-400">仅导出当前筛选的 {{ store.filteredFrames.length }} 条帧</div>
            </button>
            <button
              @click="handleExport(false, true)"
              class="w-full text-left px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 transition-colors"
              :class="{ 'opacity-50 cursor-not-allowed': store.selectedSignals.size === 0 }"
            >
              <div class="font-medium">导出选中信号</div>
              <div class="text-xs text-gray-400">
                {{ store.selectedSignals.size > 0 ? `已选 ${store.selectedSignals.size} 个信号，每列为单独字段` : '请先点击图例选择信号' }}
              </div>
            </button>
            <button
              @click="handleExport(true, true)"
              class="w-full text-left px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 transition-colors border-t border-gray-700"
              :class="{ 'opacity-50 cursor-not-allowed': store.selectedSignals.size === 0 }"
            >
              <div class="font-medium">筛选结果 + 选中信号</div>
              <div class="text-xs text-gray-400">
                {{ store.selectedSignals.size > 0 ? `筛选帧中导出 ${store.selectedSignals.size} 个信号` : '请先点击图例选择信号' }}
              </div>
            </button>
          </div>
        </div>
      </div>
    </header>

    <!-- Main Area -->
    <main class="flex-1 flex overflow-hidden">
      <!-- Left Panel: Frame Table (60%) -->
      <div class="w-3/5 border-r border-gray-700 flex flex-col overflow-hidden">
        <FrameTable />
      </div>

      <!-- Right Panel: Signal Chart (40%) -->
      <div class="w-2/5 flex flex-col overflow-hidden">
        <SignalChart />
      </div>
    </main>

    <!-- Status Bar -->
    <footer class="flex items-center justify-between px-6 py-1.5 bg-gray-800 border-t border-gray-700 text-xs shrink-0">
      <div class="flex items-center gap-4 text-gray-500">
        <span>
          <span :class="store.isCapturing ? 'text-green-400' : 'text-gray-500'">
            ● {{ store.isCapturing ? '捕获中' : '已停止' }}
          </span>
        </span>
        <span>DBC消息: {{ store.dbcMessages.size }}</span>
      </div>
      <div class="flex items-center gap-4 text-gray-500">
        <span>帧数: {{ store.busStats.totalFrames }}</span>
        <span>RX: {{ store.busStats.rxCount }}</span>
        <span>TX: {{ store.busStats.txCount }}</span>
        <span>负载: {{ store.busLoadPercent }}%</span>
      </div>
    </footer>
  </div>
</template>
