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

      <SyTimelineChart
        :markers="markers"
        :timeline-data="timelineData"
        @remove-marker="removeMarker"
      />
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
      error: '',
      timelineData: {}
    };
  },
  created() {
    this.timelineData = this.buildTimelineData();
  },
  methods: {
    buildTimelineData() {
      const rng = this.createRng(Date.now());
      const baseDate = new Date();
      baseDate.setHours(0, 0, 0, 0);

      const startTime = baseDate.getTime();
      const span = 24 * 60 * 60 * 1000;
      const endTime = startTime + span;

      const categories = [
        'category19',
        'category17',
        'category15',
        'category13',
        'category11',
        'category9',
        'category7',
        'category5',
        'category3',
        'category1',
        'categoryC',
        'categoryA'
      ];

      const types = [
        { name: 'Type A', color: '#7b9ce1' },
        { name: 'Type B', color: '#bd6d6c' },
        { name: 'Type C', color: '#75d874' },
        { name: 'Type D', color: '#e0bc78' },
        { name: 'Type E', color: '#dc77dc' },
        { name: 'Type F', color: '#72b362' }
      ];

      const intervals = [];
      let intervalId = 1;

      categories.forEach((category, categoryIndex) => {
        const count = 18 + Math.floor(rng() * 10);
        for (let index = 0; index < count; index += 1) {
          const typeItem = types[Math.floor(rng() * types.length)];
          const start = startTime + Math.floor(rng() * (span - 60 * 1000));
          const duration = 2 * 60 * 1000 + Math.floor(rng() * 90 * 60 * 1000);
          const end = this.clamp(start + duration, startTime + 10 * 1000, endTime);

          intervals.push({
            id: intervalId,
            name: typeItem.name,
            value: [categoryIndex, start, end, end - start],
            itemStyle: {
              color: typeItem.color
            }
          });

          intervalId += 1;
        }
      });

      return {
        categories,
        intervals,
        domainStart: startTime,
        domainEnd: endTime,
        viewStart: startTime + span * 0.22,
        viewEnd: startTime + span * 0.42,
        cursorStart: startTime + span * 0.28,
        cursorEnd: startTime + span * 0.35
      };
    },
    createRng(seed) {
      let state = seed >>> 0;
      return () => {
        state ^= state << 13;
        state ^= state >>> 17;
        state ^= state << 5;
        return (state >>> 0) / 4294967296;
      };
    },
    clamp(value, min, max) {
      return Math.min(max, Math.max(min, value));
    },
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
