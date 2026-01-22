<template>
  <div class="sy-shell">
    <section class="sy-panel">
      <header class="sy-header">
        <div>
          <p class="eyebrow">SY</p>
          <h2>{{ activeTab === 'main' ? 'ER Log Overview' : 'SY Timeline Detail' }}</h2>
          <p class="lede">
            {{ activeTab === 'main'
              ? 'ER log lots by equipment. Click a bar to open details.'
              : 'Timeline detail for the selected lot.' }}
          </p>
        </div>

        <nav class="sy-tabs">
          <button
            type="button"
            class="tab"
            :class="{ active: activeTab === 'main' }"
            @click="activeTab = 'main'"
          >
            Main
          </button>
          <button
            type="button"
            class="tab"
            :class="{ active: activeTab === 'detail' }"
            @click="activeTab = 'detail'"
          >
            Detail
          </button>
        </nav>
      </header>

      <MainTab
        v-if="activeTab === 'main'"
        :er-log-data="erLogData"
        @select-lot="openDetail"
      />
      <DetailTab
        v-else
        :detail-context="detailContext"
        :build-timeline-data="buildTimelineData"
      />
    </section>
  </div>
</template>

<script>
import MainTab from './MainTab.vue';
import DetailTab from './DetailTab.vue';
import { erLogDummyData, buildSyTimelineDummyData } from './syDummyData';

function parseDateTimeMs(value) {
  const match = /^(\d{4})-(\d{2})-(\d{2})\s+(\d{2}):(\d{2}):(\d{2})\.(\d{3})$/.exec(
    String(value || '').trim()
  );
  if (!match) return null;
  const [, year, month, day, hour, minute, second, ms] = match;
  const date = new Date(
    Number(year),
    Number(month) - 1,
    Number(day),
    Number(hour),
    Number(minute),
    Number(second),
    Number(ms)
  );
  return Number.isNaN(date.getTime()) ? null : date.getTime();
}

export default {
  name: 'SyMain',
  components: {
    MainTab,
    DetailTab
  },
  data() {
    return {
      activeTab: 'main',
      erLogData: erLogDummyData,
      detailContext: null
    };
  },
  methods: {
    openDetail(payload) {
      if (!payload || !payload.lot) return;
      this.detailContext = { ...payload };
      this.activeTab = 'detail';
    },
    buildTimelineData(detailContext = {}) {
      const lot = detailContext && detailContext.lot ? detailContext.lot : null;
      const clickedTime = this.toTimeValue(
        detailContext && detailContext.clickedTime
      );
      const lotStart = this.toTimeValue(lot && lot.lotStart);
      const lotFinish = this.toTimeValue(lot && lot.lotFinish);

      let startTime;
      let endTime;

      if (lotStart != null && lotFinish != null) {
        startTime = lotStart - 60 * 60 * 1000;
        endTime = lotFinish + 60 * 60 * 1000;
      } else {
        const baseDate = new Date();
        baseDate.setHours(0, 0, 0, 0);
        startTime = baseDate.getTime();
        endTime = startTime + 24 * 60 * 60 * 1000;
      }

      if (endTime < startTime) {
        endTime = startTime + 2 * 60 * 60 * 1000;
      }

      const span = endTime - startTime;
      const focusTime =
        clickedTime != null
          ? clickedTime
          : lotStart != null
            ? lotStart
            : startTime + span * 0.35;

      const viewStart = startTime;
      const viewEnd = endTime;

      const cursorSpan = 10 * 60 * 1000;
      let cursorStart = this.clamp(
        focusTime - cursorSpan / 2,
        startTime,
        endTime
      );
      let cursorEnd = this.clamp(
        focusTime + cursorSpan / 2,
        startTime,
        endTime
      );

      if (cursorEnd <= cursorStart) {
        cursorStart = viewStart;
        cursorEnd = viewEnd;
      }

      const { categories, intervals } = buildSyTimelineDummyData({
        lot,
        startTime,
        endTime,
        seed: this.seedFromContext(lot, focusTime)
      });

      return {
        categories,
        intervals,
        domainStart: startTime,
        domainEnd: endTime,
        viewStart,
        viewEnd,
        cursorStart,
        cursorEnd,
        selectedCategoryIndex: null
      };
    },
    seedFromContext(lot, focusTime) {
      const lotId = lot && lot.lotId ? lot.lotId : '';
      const eqpId = lot && lot.eqpId ? lot.eqpId : '';
      const seedSource = `${lotId}|${eqpId}|${focusTime || ''}`;
      let hash = 0;
      for (let i = 0; i < seedSource.length; i += 1) {
        hash = (hash << 5) - hash + seedSource.charCodeAt(i);
        hash |= 0;
      }
      return hash;
    },
    clamp(value, min, max) {
      return Math.min(max, Math.max(min, value));
    },
    toTimeValue(value) {
      if (value == null) return null;
      if (typeof value === 'number') return value;
      const parsed = parseDateTimeMs(value);
      if (parsed != null) return parsed;
      const time = new Date(value).getTime();
      return Number.isNaN(time) ? null : time;
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
  gap: 16px;
  align-items: flex-start;
  margin-bottom: 18px;
  flex-wrap: wrap;
}

.eyebrow {
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-weight: 700;
  margin: 0 0 4px;
  opacity: 0.85;
  color: #cbd5f5;
}

h2 {
  margin: 0 0 6px;
  color: #f8fafc;
}

.lede {
  margin: 0;
  opacity: 0.85;
  color: #cbd5f5;
}

.sy-tabs {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.tab {
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.06);
  color: #e2e8f0;
  border-radius: 999px;
  padding: 8px 14px;
  cursor: pointer;
  transition: background 0.2s ease, border-color 0.2s ease, opacity 0.2s ease;
}

.tab:hover:enabled {
  background: rgba(99, 102, 241, 0.18);
  border-color: rgba(99, 102, 241, 0.45);
}

.tab.active {
  background: rgba(14, 165, 233, 0.22);
  border-color: rgba(14, 165, 233, 0.55);
}

.tab:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}
</style>
