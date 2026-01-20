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
const MARKER_LABEL_OFFSET_Y = -4;
const MARKER_LABEL_POSITION = 'start';
const MARKER_LABEL_ALIGN = 'left';
const MARKER_LABEL_VERTICAL_ALIGN = 'bottom';
const MARKER_LABEL_ROTATE = 18;
const MARKER_DELETE_RADIUS = 10;
const MARKER_DELETE_OFFSET_X = 0;
const MARKER_DELETE_OFFSET_Y = 0;
const TOP_SLIDER_TOP = 16;
const TOP_SLIDER_HEIGHT = 22;
const TOP_SLIDER_GAP = 6;
const TOP_LABEL_GAP = 52;
const TOP_SLIDER_BOTTOM = TOP_SLIDER_TOP + TOP_SLIDER_HEIGHT + TOP_SLIDER_GAP;
const BOTTOM_SLIDER_BOTTOM = 18;
const BOTTOM_SLIDER_HEIGHT = 22;
const GRID_LEFT = 110;
const GRID_RIGHT = 62;
const GRID_BOTTOM = 124;
const MAIN_GRID_INDEX = 0;
const PINNED_GRID_INDEX = 1;
const LABEL_GRID_INDEX = 2;
const CURSOR_GRID_INDEX = 3;
const MAIN_AXIS_INDEX = 0;
const PINNED_AXIS_INDEX = 1;
const LABEL_AXIS_INDEX = 2;
const CURSOR_AXIS_INDEX = 3;
const SLIDER_AXIS_INDEX = 4;

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

function formatAxisLabel(value) {
  const date = new Date(value);
  if (Number.isNaN(date.getTime())) return '';

  return [
    `${pad2(date.getMonth() + 1)}-${pad2(date.getDate())}`,
    `${pad2(date.getHours())}:${pad2(date.getMinutes())}`
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
      pinnedCategory: null,
      scrollCategories: [],
      pinnedIntervals: [],
      scrollIntervals: [],
      activeMarkerId: null,
      suppressZrClick: false,
      dragStart: null,
      selectedCategoryIndex: null,
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
      this.pinnedCategory = categories.length ? categories[0] : null;
      this.scrollCategories = categories.slice(1);
      this.pinnedIntervals = intervals
        .filter((item) => item && item.value && item.value[0] === 0)
        .map((item) => ({
          ...item,
          value: [0, item.value[1], item.value[2], item.value[3]]
        }));
      this.scrollIntervals = intervals
        .filter((item) => item && item.value && item.value[0] > 0)
        .map((item) => ({
          ...item,
          value: [item.value[0] - 1, item.value[1], item.value[2], item.value[3]]
        }));
      this.selectedCategoryIndex = null;
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
      this.updateGridLayout();

      this.chart.on('datazoom', this.onDataZoom);
      this.chart.on('click', this.onChartClick);
      const zr = this.chart.getZr();
      if (zr) {
        zr.on('click', this.onZrClick);
        zr.on('mousewheel', this.onChartWheel);
        zr.on('mousedown', this.onZrMouseDown);
        zr.on('mousemove', this.onZrMouseMove);
        zr.on('mouseup', this.onZrMouseUp);
      }

      this.updateMarkerLines();
      this.updateSelectionLines();
      this.updateMarkerButtons();
      this.syncTopSlider();
    },
    teardownChart() {
      if (!this.chart) return;
      this.chart.off('datazoom', this.onDataZoom);
      this.chart.off('click', this.onChartClick);
      const zr = this.chart.getZr();
      if (zr) {
        zr.off('click', this.onZrClick);
        zr.off('mousewheel', this.onChartWheel);
        zr.off('mousedown', this.onZrMouseDown);
        zr.off('mousemove', this.onZrMouseMove);
        zr.off('mouseup', this.onZrMouseUp);
      }
      this.chart.dispose();
      this.chart = null;
    },
    resizeChart() {
      if (!this.chart) return;
      this.chart.resize();
      this.updateGridLayout();
      this.updateMarkerButtons();
    },
    getLayoutMetrics() {
      const sliderBottom = TOP_SLIDER_BOTTOM;
      const gridBottom = GRID_BOTTOM;
      const chartHeight = this.chart ? this.chart.getHeight() : 640;
      const totalCategories = Math.max(this.categories.length, 1);
      const labelHeight = Math.max(0, TOP_LABEL_GAP);
      const usableHeight = Math.max(
        0,
        chartHeight - gridBottom - sliderBottom - labelHeight
      );
      const rowHeight = totalCategories ? usableHeight / totalCategories : usableHeight;
      const pinnedHeight = rowHeight;
      const labelTop = sliderBottom;
      const pinnedTop = sliderBottom + labelHeight;
      const mainTop = pinnedTop + pinnedHeight;

      return {
        sliderBottom,
        gridBottom,
        labelTop,
        labelHeight,
        pinnedHeight,
        pinnedTop,
        mainTop
      };
    },
    updateGridLayout() {
      if (!this.chart) return;
      const layout = this.getLayoutMetrics();

      this.chart.setOption({
        grid: [
          {
            id: 'grid-main',
            top: layout.mainTop,
            bottom: layout.gridBottom,
            left: GRID_LEFT,
            right: GRID_RIGHT
          },
          {
            id: 'grid-pinned',
            top: layout.pinnedTop,
            height: layout.pinnedHeight,
            left: GRID_LEFT,
            right: GRID_RIGHT
          },
          {
            id: 'grid-label',
            top: layout.labelTop,
            height: layout.labelHeight,
            left: GRID_LEFT,
            right: GRID_RIGHT
          },
          {
            id: 'grid-cursor',
            top: layout.sliderBottom,
            bottom: layout.gridBottom,
            left: GRID_LEFT,
            right: GRID_RIGHT
          }
        ],
        dataZoom: [
          {
            id: 'dz_top',
            top: TOP_SLIDER_TOP,
            height: TOP_SLIDER_HEIGHT
          },
          {
            id: 'dz_bottom',
            bottom: BOTTOM_SLIDER_BOTTOM,
            height: BOTTOM_SLIDER_HEIGHT
          },
          {
            id: 'dz_y',
            top: layout.mainTop,
            bottom: layout.gridBottom
          }
        ]
      });
    },
    onChartClick(params) {
      this.suppressZrClick = true;
      const markerId = this.getMarkerIdFromEvent(params);
      if (markerId != null) {
        this.activeMarkerId =
          this.activeMarkerId === markerId ? null : markerId;
        this.updateMarkerButtons();
        return;
      }
      if (this.activeMarkerId != null) {
        this.clearActiveMarker();
      }
      if (!params || params.seriesType !== 'custom') return;
      if (!params.value || typeof params.value[0] !== 'number') return;

      let categoryIndex = null;
      if (params.seriesId === 'sy-bars-pinned') {
        categoryIndex = 0;
      } else if (params.seriesId === 'sy-bars-main') {
        categoryIndex = params.value[0] + 1;
      }

      if (categoryIndex == null) return;
      this.selectedCategoryIndex = categoryIndex;
      this.updateSelectionLines();
    },
    onZrClick() {
      if (this.suppressZrClick) {
        this.suppressZrClick = false;
        return;
      }
      this.clearActiveMarker();
    },
    onChartWheel() {
      this.clearActiveMarker();
    },
    onZrMouseDown(event) {
      if (this.activeMarkerId == null) return;
      if (!event) return;
      this.dragStart = { x: event.offsetX, y: event.offsetY };
    },
    onZrMouseMove(event) {
      if (!this.dragStart || this.activeMarkerId == null) return;
      if (!event) return;
      const dx = event.offsetX - this.dragStart.x;
      const dy = event.offsetY - this.dragStart.y;
      if (dx * dx + dy * dy >= 16) {
        this.dragStart = null;
        this.clearActiveMarker();
      }
    },
    onZrMouseUp() {
      this.dragStart = null;
    },
    clearActiveMarker() {
      if (this.activeMarkerId == null) return;
      this.activeMarkerId = null;
      this.updateMarkerButtons();
    },
    updateSelectionLines() {
      if (!this.chart) return;

      const selected = this.selectedCategoryIndex;
      const mainData = [];
      const pinnedData = [];

      if (typeof selected === 'number') {
        if (selected === 0) {
          pinnedData.push({ yAxis: 0 });
        } else if (selected > 0) {
          mainData.push({ yAxis: selected - 1 });
        }
      }

      this.chart.setOption({
        series: [
          {
            id: 'sy-selection-main',
            markLine: {
              data: mainData
            }
          },
          {
            id: 'sy-selection-pinned',
            markLine: {
              data: pinnedData
            }
          }
        ]
      });
    },
    getMarkerIdFromEvent(params) {
      if (!params || params.componentType !== 'markLine') return null;
      const data = params.data;
      return data && data.markerId != null ? data.markerId : null;
    },
    getClampedCursorRange() {
      const topMin = this.viewStart != null ? this.viewStart : this.domainStart;
      const topMax = this.viewEnd != null ? this.viewEnd : this.domainEnd;
      const cursorStart = Math.min(this.cursorStart, this.cursorEnd);
      const cursorEnd = Math.max(this.cursorStart, this.cursorEnd);
      const clampedStart = clamp(cursorStart, topMin, topMax);
      const clampedEnd = clamp(cursorEnd, topMin, topMax);

      return {
        topMin,
        topMax,
        start: clampedStart,
        end: clampedEnd
      };
    },
    buildOption() {
      const layout = this.getLayoutMetrics();
      const minuteInterval = 60 * 1000;
      const cursorRange = this.getClampedCursorRange();
      const topMin = cursorRange.topMin;
      const topMax = cursorRange.topMax;
      const topStart = cursorRange.start;
      const topEnd = cursorRange.end;

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
            xAxisIndex: SLIDER_AXIS_INDEX,
            filterMode: 'weakFilter',
            showDataShadow: false,
            brushSelect: false,
            top: TOP_SLIDER_TOP,
            height: TOP_SLIDER_HEIGHT,
            labelFormatter: '',
            startValue: topStart,
            endValue: topEnd
          },
          {
            id: 'dz_bottom',
            type: 'slider',
            xAxisIndex: [MAIN_AXIS_INDEX, PINNED_AXIS_INDEX],
            filterMode: 'weakFilter',
            showDataShadow: false,
            brushSelect: false,
            bottom: BOTTOM_SLIDER_BOTTOM,
            height: BOTTOM_SLIDER_HEIGHT,
            labelFormatter: '',
            startValue: this.viewStart,
            endValue: this.viewEnd
          },
          {
            id: 'dz_inside',
            type: 'inside',
            xAxisIndex: [MAIN_AXIS_INDEX, PINNED_AXIS_INDEX],
            filterMode: 'weakFilter'
          },
          {
            id: 'dz_y',
            type: 'slider',
            yAxisIndex: 0,
            filterMode: 'weakFilter',
            showDataShadow: false,
            right: 8,
            top: layout.mainTop,
            bottom: layout.gridBottom,
            width: 12
          }
        ],
        grid: [
          {
            id: 'grid-main',
            top: layout.mainTop,
            bottom: layout.gridBottom,
            left: GRID_LEFT,
            right: GRID_RIGHT,
            containLabel: false
          },
          {
            id: 'grid-pinned',
            top: layout.pinnedTop,
            height: layout.pinnedHeight,
            left: GRID_LEFT,
            right: GRID_RIGHT,
            containLabel: false
          },
          {
            id: 'grid-label',
            top: layout.labelTop,
            height: layout.labelHeight,
            left: GRID_LEFT,
            right: GRID_RIGHT,
            containLabel: false
          },
          {
            id: 'grid-cursor',
            top: layout.sliderBottom,
            bottom: layout.gridBottom,
            left: GRID_LEFT,
            right: GRID_RIGHT,
            containLabel: false
          }
        ],
        xAxis: [
          {
            id: 'x-main',
            gridIndex: MAIN_GRID_INDEX,
            type: 'time',
            min: this.domainStart,
            max: this.domainEnd,
            scale: true,
            minInterval: minuteInterval,
            splitNumber: 24,
            axisLine: {
              lineStyle: {
                color: '#cbd5e1'
              }
            },
            axisLabel: {
              color: '#475569',
              rotate: 90,
              margin: 6,
              inside: false,
              hideOverlap: true,
              interval: 0,
              formatter: (value) => formatAxisLabel(value)
            },
            splitLine: {
              show: true,
              lineStyle: {
                color: '#e2e8f0'
              }
            }
          },
          {
            id: 'x-pinned',
            gridIndex: PINNED_GRID_INDEX,
            type: 'time',
            min: this.domainStart,
            max: this.domainEnd,
            scale: true,
            minInterval: minuteInterval,
            splitNumber: 24,
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
          },
          {
            id: 'x-label',
            gridIndex: LABEL_GRID_INDEX,
            type: 'time',
            min: topMin,
            max: topMax,
            scale: true,
            minInterval: minuteInterval,
            splitNumber: 24,
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
          },
          {
            id: 'x-cursor',
            gridIndex: CURSOR_GRID_INDEX,
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
          },
          {
            id: 'x-slider',
            gridIndex: CURSOR_GRID_INDEX,
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
        yAxis: [
          {
            id: 'y-main',
            gridIndex: MAIN_GRID_INDEX,
            type: 'category',
            inverse: true,
            data: this.scrollCategories,
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
          {
            id: 'y-pinned',
            gridIndex: PINNED_GRID_INDEX,
            type: 'category',
            inverse: true,
            data: this.pinnedCategory ? [this.pinnedCategory] : [],
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
          {
            id: 'y-label',
            gridIndex: LABEL_GRID_INDEX,
            type: 'value',
            min: 0,
            max: 1,
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
          },
          {
            id: 'y-cursor',
            gridIndex: CURSOR_GRID_INDEX,
            type: 'value',
            min: 0,
            max: 1,
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
        series: [
          {
            id: 'sy-bars-main',
            type: 'custom',
            renderItem,
            xAxisIndex: MAIN_AXIS_INDEX,
            yAxisIndex: MAIN_AXIS_INDEX,
            itemStyle: {
              opacity: 0.85
            },
            encode: {
              x: [1, 2],
              y: 0
            },
            data: this.scrollIntervals
          },
          {
            id: 'sy-bars-pinned',
            type: 'custom',
            renderItem,
            xAxisIndex: PINNED_AXIS_INDEX,
            yAxisIndex: PINNED_AXIS_INDEX,
            itemStyle: {
              opacity: 0.85
            },
            encode: {
              x: [1, 2],
              y: 0
            },
            data: this.pinnedIntervals
          },
          {
            id: 'sy-selection-main',
            type: 'scatter',
            xAxisIndex: MAIN_AXIS_INDEX,
            yAxisIndex: MAIN_AXIS_INDEX,
            data: [],
            silent: true,
            markLine: {
              symbol: ['none', 'none'],
              z: 18,
              lineStyle: {
                width: 2,
                color: '#f97316',
                type: 'solid'
              },
              label: {
                show: false
              },
              data: []
            }
          },
          {
            id: 'sy-selection-pinned',
            type: 'scatter',
            xAxisIndex: PINNED_AXIS_INDEX,
            yAxisIndex: PINNED_AXIS_INDEX,
            data: [],
            silent: true,
            markLine: {
              symbol: ['none', 'none'],
              z: 18,
              lineStyle: {
                width: 2,
                color: '#f97316',
                type: 'solid'
              },
              label: {
                show: false
              },
              data: []
            }
          },
          {
            id: 'sy-markers-main',
            type: 'scatter',
            xAxisIndex: MAIN_AXIS_INDEX,
            yAxisIndex: MAIN_AXIS_INDEX,
            data: [],
            silent: false,
            emphasis: {
              disabled: true
            },
            markLine: {
              symbol: ['none', 'none'],
              precision: 0,
              z: 20,
              lineStyle: {
                width: 2,
                color: '#2563eb',
                type: 'dashed'
              },
              emphasis: {
                disabled: true
              },
              label: {
                show: false
              },
              data: []
            }
          },
          {
            id: 'sy-markers-pinned-lines',
            type: 'scatter',
            xAxisIndex: PINNED_AXIS_INDEX,
            yAxisIndex: PINNED_AXIS_INDEX,
            data: [],
            silent: true,
            emphasis: {
              disabled: true
            },
            markLine: {
              symbol: ['none', 'none'],
              precision: 0,
              z: 20,
              lineStyle: {
                width: 2,
                color: '#2563eb',
                type: 'dashed'
              },
              emphasis: {
                disabled: true
              },
              label: {
                show: false
              },
              data: []
            }
          },
          {
            id: 'sy-markers-labels',
            type: 'scatter',
            xAxisIndex: LABEL_AXIS_INDEX,
            yAxisIndex: LABEL_AXIS_INDEX,
            data: [],
            silent: false,
            emphasis: {
              disabled: true
            },
            markLine: {
              symbol: ['none', 'none'],
              precision: 0,
              z: 22,
              lineStyle: {
                width: 1,
                color: 'rgba(37, 99, 235, 0.02)',
                type: 'dashed'
              },
              emphasis: {
                disabled: true
              },
              label: {
                show: true,
                position: MARKER_LABEL_POSITION,
                align: MARKER_LABEL_ALIGN,
                verticalAlign: MARKER_LABEL_VERTICAL_ALIGN,
                distance: MARKER_LABEL_DISTANCE,
                offset: [0, MARKER_LABEL_OFFSET_Y],
                rotate: MARKER_LABEL_ROTATE,
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
              data: []
            }
          },
          {
            id: 'sy-cursors-overlay',
            type: 'scatter',
            xAxisIndex: CURSOR_AXIS_INDEX,
            yAxisIndex: CURSOR_AXIS_INDEX,
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
                { xAxis: topStart },
                { xAxis: topEnd }
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
        return { start: Math.round(start), end: Math.round(end) };
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
        return { start: Math.round(start), end: Math.round(end) };
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

        if (zoomId === 'dz_y') {
          return;
        }

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
    getGridRect(gridIndex = MAIN_GRID_INDEX) {
      if (!this.chart) return null;
      const model = this.chart.getModel && this.chart.getModel();
      const grid =
        model && model.getComponent && model.getComponent('grid', gridIndex);
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
      const gridOption = option && option.grid && option.grid[gridIndex];
      if (!gridOption) return null;

      const width = this.chart.getWidth();
      const height = this.chart.getHeight();
      const left = typeof gridOption.left === 'number' ? gridOption.left : 0;
      const right = typeof gridOption.right === 'number' ? gridOption.right : 0;
      const top = typeof gridOption.top === 'number' ? gridOption.top : 0;
      const bottom = typeof gridOption.bottom === 'number' ? gridOption.bottom : 0;
      const explicitWidth =
        typeof gridOption.width === 'number' ? gridOption.width : null;
      const explicitHeight =
        typeof gridOption.height === 'number' ? gridOption.height : null;

      return {
        x: left,
        y: top,
        width:
          explicitWidth != null
            ? explicitWidth
            : Math.max(0, width - left - right),
        height:
          explicitHeight != null
            ? explicitHeight
            : Math.max(0, height - top - bottom)
      };
    },
    getAxisPixelFromValue(value, xAxisIndex = MAIN_AXIS_INDEX) {
      if (!this.chart) return null;
      const pixel = this.chart.convertToPixel({ xAxisIndex }, value);
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
            id: 'sy-markers-main',
            markLine: {
              data: markerLines
            }
          },
          {
            id: 'sy-markers-pinned-lines',
            markLine: {
              data: markerLines
            }
          },
          {
            id: 'sy-markers-labels',
            markLine: {
              data: markerLines
            }
          }
        ]
      });
    },
    updateCursorLines() {
      if (!this.chart) return;
      const cursorRange = this.getClampedCursorRange();

      this.chart.setOption({
        series: [
          {
            id: 'sy-cursors-overlay',
            markLine: {
              data: [
                { xAxis: cursorRange.start },
                { xAxis: cursorRange.end }
              ]
            }
          }
        ]
      });
    },
    syncTopSlider() {
      if (!this.chart) return;

      const cursorRange = this.getClampedCursorRange();

      this.isSyncingTopSlider = true;
      this.chart.setOption({
        xAxis: [
          {
            id: 'x-label',
            min: cursorRange.topMin,
            max: cursorRange.topMax
          },
          {
            id: 'x-cursor',
            min: cursorRange.topMin,
            max: cursorRange.topMax
          },
          {
            id: 'x-slider',
            min: cursorRange.topMin,
            max: cursorRange.topMax
          }
        ],
        dataZoom: [
          {
            id: 'dz_top',
            startValue: cursorRange.start,
            endValue: cursorRange.end
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

      if (this.activeMarkerId == null) {
        this.markerButtons = [];
        return;
      }

      const markers = (this.markers || []).filter(
        (marker) =>
          marker &&
          marker.time != null &&
          marker.id === this.activeMarkerId
      );
      const mainGridRect = this.getGridRect(MAIN_GRID_INDEX);
      const pinnedGridRect = this.getGridRect(PINNED_GRID_INDEX);
      const buttons = [];

      if (!markers.length) {
        this.activeMarkerId = null;
        this.markerButtons = [];
        return;
      }

      const gridRects = [mainGridRect, pinnedGridRect].filter(
        (rect) => rect && rect.height > 0
      );
      const lineCenterY = gridRects.length
        ? (Math.min(...gridRects.map((rect) => rect.y)) +
            Math.max(...gridRects.map((rect) => rect.y + rect.height))) /
          2
        : null;

      if (markers.length && mainGridRect && lineCenterY != null) {
        const chartWidth = this.chart.getWidth();
        const chartHeight = this.chart.getHeight();
        const leftBound = MARKER_DELETE_RADIUS;
        const rightBound = chartWidth - MARKER_DELETE_RADIUS;
        const topBound = MARKER_DELETE_RADIUS;
        const bottomBound = chartHeight - MARKER_DELETE_RADIUS;

        markers.forEach((marker) => {
          const markerTime = toTimeValue(marker.time);
          if (markerTime == null) return;

          const lineX = this.getAxisPixelFromValue(markerTime, MAIN_AXIS_INDEX);
          if (lineX == null || Number.isNaN(lineX)) return;
          if (
            lineX < mainGridRect.x ||
            lineX > mainGridRect.x + mainGridRect.width
          ) {
            return;
          }

          const rawButtonX = lineX + MARKER_DELETE_OFFSET_X;
          const rawButtonY = lineCenterY + MARKER_DELETE_OFFSET_Y;
          const buttonX = clamp(rawButtonX, leftBound, rightBound);
          const buttonY = clamp(rawButtonY, topBound, bottomBound);

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
  user-select: none;
  -webkit-user-select: none;
  -ms-user-select: none;
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
