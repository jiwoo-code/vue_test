<template>
  <div class="sy-chart-wrap">
    <div class="sy-chart" ref="chart"></div>
    <div class="sy-marker-overlay">
      <button
        v-for="item in markerButtons"
        :key="item.id"
        class="sy-marker-btn"
        type="button"
        :style="{ left: `${item.x}px`, top: `${item.y}px` }"
        @click="$emit('remove-marker', item.id)"
      >
        x
      </button>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';

const MARKER_LABEL_PADDING = [2, 6];
const MARKER_LABEL_FONT_SIZE = 12;
const MARKER_LABEL_FONT_FAMILY = 'sans-serif';
const MARKER_LABEL_BORDER_WIDTH = 1;
const MARKER_LABEL_DISTANCE = 6;
const MARKER_LABEL_POSITION = 'start';
const MARKER_LABEL_ALIGN = 'left';
const MARKER_LABEL_VERTICAL_ALIGN = 'middle';
const MARKER_DELETE_RADIUS = 10;
const MARKER_DELETE_GAP = 8;

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

function toTimeValue(value) {
  if (value == null) return null;
  if (typeof value === 'number') return value;
  if (echarts && echarts.number && typeof echarts.number.parseDate === 'function') {
    const parsed = echarts.number.parseDate(value);
    if (parsed && !Number.isNaN(parsed.getTime())) {
      return parsed.getTime();
    }
  }
  const time = new Date(value).getTime();
  return Number.isNaN(time) ? null : time;
}

function clamp(value, min, max) {
  return Math.min(max, Math.max(min, value));
}

function createRng(seed) {
  let state = seed >>> 0;
  return () => {
    state ^= state << 13;
    state ^= state >>> 17;
    state ^= state << 5;
    return (state >>> 0) / 4294967296;
  };
}

function renderItem(params, api) {
  const categoryIndex = api.value(0);
  const start = api.coord([api.value(1), categoryIndex]);
  const end = api.coord([api.value(2), categoryIndex]);
  const height = api.size([0, 1])[1] * 0.6;

  const rectShape = echarts.graphic.clipRectByRect(
    {
      x: start[0],
      y: start[1] - height / 2,
      width: end[0] - start[0],
      height
    },
    {
      x: params.coordSys.x,
      y: params.coordSys.y,
      width: params.coordSys.width,
      height: params.coordSys.height
    }
  );

  return (
    rectShape && {
      type: 'rect',
      transition: ['shape'],
      shape: rectShape,
      style: api.style()
    }
  );
}

export default {
  name: 'SyTimelineChart',
  props: {
    markers: {
      type: Array,
      default: () => []
    }
  },
  data() {
    return {
      chart: null,
      markerButtons: [],
      categories: [],
      intervals: [],
      domainStart: 0,
      domainEnd: 0,
      viewStart: 0,
      viewEnd: 0,
      cursorStart: 0,
      cursorEnd: 0,
      isSyncingTopSlider: false
    };
  },
  watch: {
    markers: {
      deep: true,
      handler() {
        this.updateMarkerLines();
        this.updateMarkerButtons();
      }
    }
  },
  mounted() {
    this.initData();
    this.initChart();
    window.addEventListener('resize', this.resizeChart, { passive: true });
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.resizeChart);
    this.teardownChart();
  },
  methods: {
    initData() {
      const rng = createRng(Date.now());
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
          const end = clamp(start + duration, startTime + 10 * 1000, endTime);

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

      this.categories = categories;
      this.intervals = intervals;
      this.domainStart = startTime;
      this.domainEnd = endTime;

      this.viewStart = startTime + span * 0.22;
      this.viewEnd = startTime + span * 0.42;
      this.cursorStart = startTime + span * 0.28;
      this.cursorEnd = startTime + span * 0.35;
    },
    initChart() {
      const el = this.$refs.chart;
      if (!el) return;

      this.chart = echarts.init(el);
      this.chart.setOption(this.buildOption(), { notMerge: true });

      this.chart.on('datazoom', this.onDataZoom);

      this.updateMarkerLines();
      this.updateMarkerButtons();
      this.syncTopSlider();
    },
    teardownChart() {
      if (!this.chart) return;
      this.chart.off('datazoom', this.onDataZoom);
      this.chart.dispose();
      this.chart = null;
    },
    resizeChart() {
      if (!this.chart) return;
      this.chart.resize();
      this.updateMarkerButtons();
    },
    buildOption() {
      const gridTop = 76;
      const gridBottom = 78;
      const gridLeft = 110;
      const gridRight = 42;
      const topMin = this.viewStart != null ? this.viewStart : this.domainStart;
      const topMax = this.viewEnd != null ? this.viewEnd : this.domainEnd;
      const cursorStart = Math.min(this.cursorStart, this.cursorEnd);
      const cursorEnd = Math.max(this.cursorStart, this.cursorEnd);
      const topStart = clamp(cursorStart, topMin, topMax);
      const topEnd = clamp(cursorEnd, topMin, topMax);

      return {
        backgroundColor: '#ffffff',
        animation: false,
        tooltip: {
          show: false,
          formatter: (params) => {
            if (!params || !params.value) return '';
            const start = formatDateTime(params.value[1]);
            const end = formatDateTime(params.value[2]);
            return [
              `${params.marker}${params.name}`,
              `${start} → ${end}`,
              `${Math.round(params.value[3] / 1000)}s`
            ].join('<br/>');
          }
        },
        dataZoom: [
          {
            id: 'dz_top',
            type: 'slider',
            xAxisIndex: 1,
            filterMode: 'weakFilter',
            showDataShadow: false,
            top: 16,
            height: 22,
            labelFormatter: '',
            startValue: topStart,
            endValue: topEnd
          },
          {
            id: 'dz_bottom',
            type: 'slider',
            xAxisIndex: 0,
            filterMode: 'weakFilter',
            showDataShadow: false,
            bottom: 18,
            height: 22,
            labelFormatter: '',
            startValue: this.viewStart,
            endValue: this.viewEnd
          },
          {
            id: 'dz_inside',
            type: 'inside',
            xAxisIndex: 0,
            filterMode: 'weakFilter'
          }
        ],
        grid: {
          top: gridTop,
          bottom: gridBottom,
          left: gridLeft,
          right: gridRight,
          containLabel: false
        },
        xAxis: [
          {
            id: 'x-main',
            type: 'time',
            min: this.domainStart,
            max: this.domainEnd,
            scale: true,
            axisLine: {
              lineStyle: {
                color: '#cbd5e1'
              }
            },
            axisLabel: {
              color: '#475569',
              formatter: (value) => formatDateTime(value)
            },
            splitLine: {
              show: true,
              lineStyle: {
                color: '#e2e8f0'
              }
            }
          },
          {
            id: 'x-cursor',
            type: 'time',
            min: topMin,
            max: topMax,
            scale: true,
            show: false,
            axisLine: {
              show: false
            },
            axisTick: {
              show: false
            },
            axisLabel: {
              show: false
            },
            splitLine: {
              show: false
            }
          }
        ],
        yAxis: {
          type: 'category',
          inverse: true,
          data: this.categories,
          axisTick: {
            show: false
          },
          axisLine: {
            show: false
          },
          axisLabel: {
            color: '#475569'
          }
        },
        series: [
          {
            id: 'sy-bars',
            type: 'custom',
            renderItem,
            itemStyle: {
              opacity: 0.85
            },
            encode: {
              x: [1, 2],
              y: 0
            },
            data: this.intervals
          },
          {
            id: 'sy-markers',
            type: 'scatter',
            data: [],
            silent: false,
            markLine: {
              symbol: ['none', 'none'],
              precision: 0,
              z: 20,
              lineStyle: {
                width: 2,
                color: '#2563eb',
                type: 'dashed'
              },
              label: {
                show: true,
                position: MARKER_LABEL_POSITION,
                align: MARKER_LABEL_ALIGN,
                verticalAlign: MARKER_LABEL_VERTICAL_ALIGN,
                distance: MARKER_LABEL_DISTANCE,
                color: '#1d4ed8',
                backgroundColor: 'rgba(37, 99, 235, 0.12)',
                borderColor: 'rgba(37, 99, 235, 0.35)',
                borderWidth: MARKER_LABEL_BORDER_WIDTH,
                padding: MARKER_LABEL_PADDING,
                borderRadius: 6,
                fontSize: MARKER_LABEL_FONT_SIZE,
                fontFamily: MARKER_LABEL_FONT_FAMILY,
                formatter: (params) => {
                  const x = params && params.data && params.data.xAxis;
                  return formatDateTime(x);
                }
              },
              emphasis: {
                lineStyle: {
                  width: 2.5
                }
              },
              data: []
            }
          },
          {
            id: 'sy-cursors',
            type: 'scatter',
            data: [],
            silent: true,
            markLine: {
              symbol: ['none', 'none'],
              z: 15,
              lineStyle: {
                width: 2,
                color: '#ef4444',
                type: 'solid'
              },
              label: {
                show: false
              },
              data: [
                { xAxis: this.cursorStart },
                { xAxis: this.cursorEnd }
              ]
            }
          }
        ]
      };
    },
    getDataZoomId(item) {
      if (!item) return null;
      if (item.dataZoomId) return item.dataZoomId;
      if (typeof item.dataZoomIndex === 'number' && this.chart) {
        const option = this.chart.getOption();
        const dataZoom = option && option.dataZoom;
        const dz = Array.isArray(dataZoom) ? dataZoom[item.dataZoomIndex] : null;
        return dz && dz.id ? dz.id : null;
      }
      return null;
    },
    getZoomFromEvent(event, options = {}) {
      const axisMin =
        options.axisMin != null ? options.axisMin : this.domainStart;
      const axisMax =
        options.axisMax != null ? options.axisMax : this.domainEnd;
      const dataZoomId = options.dataZoomId || 'dz_bottom';
      const fallbackStart =
        options.fallbackStart != null ? options.fallbackStart : this.viewStart;
      const fallbackEnd =
        options.fallbackEnd != null ? options.fallbackEnd : this.viewEnd;
      const batch = event && event.batch && event.batch[0];

      if (batch && (batch.startValue != null || batch.endValue != null)) {
        const start =
          batch.startValue != null ? batch.startValue : fallbackStart;
        const end = batch.endValue != null ? batch.endValue : fallbackEnd;
        return { start, end };
      }
      if (batch && (batch.start != null || batch.end != null)) {
        const span = axisMax - axisMin;
        const start =
          batch.start != null && span
            ? axisMin + (span * batch.start) / 100
            : fallbackStart;
        const end =
          batch.end != null && span
            ? axisMin + (span * batch.end) / 100
            : fallbackEnd;
        return { start, end };
      }

      if (!this.chart) {
        return { start: fallbackStart, end: fallbackEnd };
      }

      const option = this.chart.getOption();
      const dataZoom = option && option.dataZoom;
      const dz = Array.isArray(dataZoom)
        ? dataZoom.find((item) => item && item.id === dataZoomId)
        : null;
      if (dz && (dz.startValue != null || dz.endValue != null)) {
        const start =
          dz.startValue != null ? dz.startValue : fallbackStart;
        const end = dz.endValue != null ? dz.endValue : fallbackEnd;
        return { start, end };
      }
      if (dz && (dz.start != null || dz.end != null)) {
        const span = axisMax - axisMin;
        const start =
          dz.start != null && span
            ? axisMin + (span * dz.start) / 100
            : fallbackStart;
        const end =
          dz.end != null && span
            ? axisMin + (span * dz.end) / 100
            : fallbackEnd;
        return { start, end };
      }

      return { start: fallbackStart, end: fallbackEnd };
    },
    onDataZoom(event) {
      const batches =
        event && Array.isArray(event.batch) && event.batch.length
          ? event.batch
          : event
            ? [event]
            : [];
      if (!batches.length) return;

      let cursorUpdated = false;
      let viewUpdated = false;

      const topMin = this.viewStart != null ? this.viewStart : this.domainStart;
      const topMax = this.viewEnd != null ? this.viewEnd : this.domainEnd;

      batches.forEach((batch) => {
        if (!batch) return;
        const zoomId = this.getDataZoomId(batch) || this.getDataZoomId(event);

        if (zoomId === 'dz_top') {
          if (this.isSyncingTopSlider) return;

          const { start, end } = this.getZoomFromEvent(
            { batch: [batch] },
            {
              axisMin: topMin,
              axisMax: topMax,
              dataZoomId: 'dz_top',
              fallbackStart: this.cursorStart,
              fallbackEnd: this.cursorEnd
            }
          );

          if (start != null) {
            this.cursorStart = start;
          }
          if (end != null) {
            this.cursorEnd = end;
          }
          cursorUpdated = true;
          return;
        }

        const { start, end } = this.getZoomFromEvent(
          { batch: [batch] },
          {
            axisMin: this.domainStart,
            axisMax: this.domainEnd,
            dataZoomId: zoomId || 'dz_bottom',
            fallbackStart: this.viewStart,
            fallbackEnd: this.viewEnd
          }
        );
        this.viewStart = start;
        this.viewEnd = end;
        viewUpdated = true;
      });

      if (cursorUpdated) {
        this.updateCursorLines();
      }
      if (viewUpdated) {
        this.syncTopSlider();
      }
      if (viewUpdated || cursorUpdated) {
        this.updateMarkerButtons();
      }
    },
    getGridRect() {
      if (!this.chart) return null;
      const model = this.chart.getModel && this.chart.getModel();
      const grid = model && model.getComponent && model.getComponent('grid', 0);
      const coord = grid && grid.coordinateSystem;
      if (coord && coord.getRect) {
        const rect = coord.getRect();
        if (rect) {
          return {
            x: rect.x,
            y: rect.y,
            width: rect.width,
            height: rect.height
          };
        }
      }

      const option = this.chart.getOption();
      const gridOption = option && option.grid && option.grid[0];
      if (!gridOption) return null;

      const width = this.chart.getWidth();
      const height = this.chart.getHeight();
      const left = typeof gridOption.left === 'number' ? gridOption.left : 0;
      const right = typeof gridOption.right === 'number' ? gridOption.right : 0;
      const top = typeof gridOption.top === 'number' ? gridOption.top : 0;
      const bottom = typeof gridOption.bottom === 'number' ? gridOption.bottom : 0;

      return {
        x: left,
        y: top,
        width: Math.max(0, width - left - right),
        height: Math.max(0, height - top - bottom)
      };
    },
    getAxisPixelFromValue(value) {
      if (!this.chart) return null;
      const pixel = this.chart.convertToPixel({ xAxisIndex: 0 }, value);
      return Array.isArray(pixel) ? pixel[0] : pixel;
    },
    updateMarkerLines() {
      if (!this.chart) return;

      const markerLines = (this.markers || [])
        .filter((marker) => marker && marker.time != null)
        .map((marker) => ({
          xAxis: marker.time,
          markerId: marker.id
        }));

      this.chart.setOption({
        series: [
          {
            id: 'sy-markers',
            markLine: {
              data: markerLines
            }
          }
        ]
      });
    },
    updateCursorLines() {
      if (!this.chart) return;

      this.chart.setOption({
        series: [
          {
            id: 'sy-cursors',
            markLine: {
              data: [
                { xAxis: this.cursorStart },
                { xAxis: this.cursorEnd }
              ]
            }
          }
        ]
      });
    },
    syncTopSlider() {
      if (!this.chart) return;

      const topMin = this.viewStart != null ? this.viewStart : this.domainStart;
      const topMax = this.viewEnd != null ? this.viewEnd : this.domainEnd;
      const cursorStart = Math.min(this.cursorStart, this.cursorEnd);
      const cursorEnd = Math.max(this.cursorStart, this.cursorEnd);
      const topStart = clamp(cursorStart, topMin, topMax);
      const topEnd = clamp(cursorEnd, topMin, topMax);

      this.isSyncingTopSlider = true;
      this.chart.setOption({
        xAxis: [
          {
            id: 'x-cursor',
            min: topMin,
            max: topMax
          }
        ],
        dataZoom: [
          {
            id: 'dz_top',
            startValue: topStart,
            endValue: topEnd
          }
        ]
      });

      setTimeout(() => {
        this.isSyncingTopSlider = false;
      }, 0);
    },
    updateMarkerButtons() {
      if (!this.chart) {
        this.markerButtons = [];
        return;
      }

      const markers = (this.markers || []).filter(
        (marker) => marker && marker.time != null
      );
      const gridRect = this.getGridRect();
      const buttons = [];

      if (markers.length && gridRect) {
        const labelFont = `${MARKER_LABEL_FONT_SIZE}px ${MARKER_LABEL_FONT_FAMILY}`;
        const chartWidth = this.chart.getWidth();
        const chartHeight = this.chart.getHeight();
        const leftBound = MARKER_DELETE_RADIUS;
        const rightBound = chartWidth - MARKER_DELETE_RADIUS;
        const topBound = MARKER_DELETE_RADIUS;
        const bottomBound = chartHeight - MARKER_DELETE_RADIUS;
        let labelAnchorY = gridRect.y;
        const labelPosition = String(MARKER_LABEL_POSITION || '').toLowerCase();

        if (labelPosition.includes('end')) {
          labelAnchorY = gridRect.y + gridRect.height;
        } else if (labelPosition.includes('middle')) {
          labelAnchorY = gridRect.y + gridRect.height / 2;
        }

        markers.forEach((marker) => {
          const markerTime = toTimeValue(marker.time);
          if (markerTime == null) return;

          const lineX = this.getAxisPixelFromValue(markerTime);
          if (lineX == null || Number.isNaN(lineX)) return;
          if (lineX < gridRect.x || lineX > gridRect.x + gridRect.width) {
            return;
          }

          const labelText = formatDateTime(markerTime);
          const textRect = echarts.graphic.getTextRect(labelText, labelFont);
          const labelWidth =
            textRect.width +
            MARKER_LABEL_PADDING[1] * 2 +
            MARKER_LABEL_BORDER_WIDTH * 2;
          const labelHeight =
            textRect.height +
            MARKER_LABEL_PADDING[0] * 2 +
            MARKER_LABEL_BORDER_WIDTH * 2;
          let rawLabelLeft = lineX + MARKER_LABEL_DISTANCE;
          if (MARKER_LABEL_ALIGN === 'right') {
            rawLabelLeft = lineX - MARKER_LABEL_DISTANCE - labelWidth;
          } else if (MARKER_LABEL_ALIGN === 'center') {
            rawLabelLeft = lineX - labelWidth / 2;
          }
          const maxLabelLeft = Math.max(
            gridRect.x,
            gridRect.x + gridRect.width - labelWidth
          );
          const labelLeft = clamp(rawLabelLeft, gridRect.x, maxLabelLeft);
          let labelCenterY = labelAnchorY;
          if (MARKER_LABEL_VERTICAL_ALIGN === 'top') {
            labelCenterY = labelAnchorY + labelHeight / 2;
          } else if (MARKER_LABEL_VERTICAL_ALIGN === 'bottom') {
            labelCenterY = labelAnchorY - labelHeight / 2;
          }
          const labelTop = labelCenterY - labelHeight / 2;
          const labelCenterX = labelLeft + labelWidth / 2;
          const candidates = [
            {
              x:
                labelLeft +
                labelWidth +
                MARKER_DELETE_GAP +
                MARKER_DELETE_RADIUS,
              y: labelCenterY
            },
            {
              x: labelLeft - MARKER_DELETE_GAP - MARKER_DELETE_RADIUS,
              y: labelCenterY
            },
            {
              x: labelCenterX,
              y: labelTop - MARKER_DELETE_GAP - MARKER_DELETE_RADIUS
            },
            {
              x: labelCenterX,
              y: labelTop + labelHeight + MARKER_DELETE_GAP + MARKER_DELETE_RADIUS
            }
          ];
          const picked =
            candidates.find(
              (pos) =>
                pos.x >= leftBound &&
                pos.x <= rightBound &&
                pos.y >= topBound &&
                pos.y <= bottomBound
            ) || candidates[0];
          const buttonX = clamp(picked.x, leftBound, rightBound);
          const buttonY = clamp(picked.y, topBound, bottomBound);

          buttons.push({
            id: marker.id,
            x: buttonX,
            y: buttonY
          });
        });
      }

      this.markerButtons = buttons;
    }
  }
};
</script>

<style scoped>
.sy-chart-wrap {
  position: relative;
  width: 100%;
  height: 640px;
}

.sy-chart {
  width: 100%;
  height: 100%;
  border-radius: 14px;
  overflow: hidden;
  background: #ffffff;
  border: 1px solid rgba(15, 23, 42, 0.12);
}

.sy-marker-overlay {
  position: absolute;
  inset: 0;
  pointer-events: none;
  border-radius: 14px;
  overflow: hidden;
}

.sy-marker-btn {
  position: absolute;
  transform: translate(-50%, -50%);
  width: 20px;
  height: 20px;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.85);
  background: #1d4ed8;
  color: #ffffff;
  font: bold 12px sans-serif;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  cursor: pointer;
  pointer-events: auto;
}
</style>
