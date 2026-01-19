<template>
  <div class="sy-shell">
    <section class="sy-panel">
      <header class="sy-header">
        <div>
          <p class="eyebrow">SY</p>
          <h2>Timeline (ECharts)</h2>
          <p class="lede">상단 슬라이더(dataZoom)와 세로 기준선을 사용합니다.</p>
        </div>
      </header>

      <div class="toolbar">
        <label class="field">
          기준선 시간 (yyyy-MM-dd HH:mm:ss)
          <input v-model="markerInput" type="text" placeholder="yyyy-MM-dd HH:mm:ss" />
        </label>

        <button type="button" class="btn" @click="addMarker">점선 추가</button>
        <button type="button" class="btn ghost" :disabled="markers.length === 0" @click="clearMarkers">
          모두 삭제
        </button>
      </div>

      <p v-if="error" class="error">{{ error }}</p>

      <SyTimelineChart :markers="markers" @remove-marker="removeMarker" />
    </section>
  </div>
</template>

<script>
import SyTimelineChart from './SyTimelineChart.vue';

function pad2(value) {
  return String(value).padStart(2, '0');
}

function formatDateTime(value) {
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return '';

  return [
    `${date.getFullYear()}-${pad2(date.getMonth() + 1)}-${pad2(date.getDate())}`,
    `${pad2(date.getHours())}:${pad2(date.getMinutes())}:${pad2(date.getSeconds())}`
  ].join(' ');
}

function parseDateTime(text) {
  const normalized = String(text || '').trim().replace('T', ' ');
  const match = /^(\d{4})-(\d{2})-(\d{2})\s+(\d{2}):(\d{2})(?::(\d{2}))?$/.exec(normalized);
  if (!match) return null;

  const [, year, month, day, hour, minute, second = '0'] = match;
  const date = new Date(
    Number(year),
    Number(month) - 1,
    Number(day),
    Number(hour),
    Number(minute),
    Number(second)
  );

  if (Number.isNaN(date.getTime())) return null;
  return date.getTime();
}

export default {
  name: 'SyPage',
  components: {
    SyTimelineChart
  },
  data() {
    return {
      markers: [],
      markerInput: formatDateTime(Date.now()),
      error: ''
    };
  },
  methods: {
    addMarker() {
      this.error = '';

      const time = parseDateTime(this.markerInput);
      if (time == null) {
        this.error = '시간 형식은 yyyy-MM-dd HH:mm:ss 입니다.';
        return;
      }

      const id = `marker-${Date.now()}-${Math.random().toString(16).slice(2)}`;
      const next = [...this.markers, { id, time }];
      next.sort((a, b) => a.time - b.time);
      this.markers = next;
    },
    removeMarker(markerId) {
      this.markers = this.markers.filter((item) => item.id !== markerId);
    },
    clearMarkers() {
      this.markers = [];
    }
  }
};
</script>

<style scoped>
.sy-shell {
  display: flex;
  justify-content: center;
}

.sy-panel {
  background: #0b1224;
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 18px;
  padding: 24px 20px;
  width: 100%;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.4);
}

.sy-header {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  align-items: flex-start;
  margin-bottom: 16px;
}

h2 {
  margin: 0 0 6px;
}

.lede {
  margin: 0;
  opacity: 0.85;
}

.toolbar {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  align-items: flex-end;
  margin-bottom: 10px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 13px;
  min-width: 280px;
  flex: 1;
}

input[type='text'] {
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.06);
  color: #e2e8f0;
  padding: 8px 10px;
  width: 100%;
  box-sizing: border-box;
}

.btn {
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.08);
  color: #e2e8f0;
  border-radius: 10px;
  padding: 9px 12px;
  cursor: pointer;
  transition: background 0.2s ease, border-color 0.2s ease, opacity 0.2s ease;
}

.btn:hover:enabled {
  background: rgba(99, 102, 241, 0.2);
  border-color: rgba(99, 102, 241, 0.5);
}

.btn.ghost {
  border-color: rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.04);
}

.btn:disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

.error {
  margin: 0 0 12px;
  color: #fca5a5;
  font-size: 13px;
}
</style>
