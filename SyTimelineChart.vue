<template>
  <div class="sy-chart-wrap">
    <v-chart
      ref="chart"
      class="sy-chart"
      :option="chartOption"
      :autoresize="true"
      @datazoom="onDataZoom"
      @click="onChartClick"
      @mouseover="onChartMouseOver"
      @mouseout="onChartMouseOut"
      @zr:mousewheel="onChartWheel"
      @zr:mousemove="onZrMouseMove"
      @globalout="onZrGlobalOut"
    />
    <div class="sy-marker-overlay">
      <button
        v-for="item in markerButtons"
        :key="item.id"
        class="sy-marker-btn"
        type="button"
        :style="{ left: `${item.x}px`, top: `${item.y}px` }"
        @mouseenter="onMarkerButtonEnter(item.id)"
        @mouseleave="onMarkerButtonLeave"
        @click="$emit('remove-marker', item.id)"
      >
        x
      </button>
    </div>
  </div>
</template>

<script>
import { use, graphic, number } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { CustomChart, ScatterChart } from 'echarts/charts';
import {
  GridComponent,
  DataZoomComponent,
  MarkLineComponent,
  TooltipComponent
} from 'echarts/components';
import VChart from 'vue-echarts';

use([
  CanvasRenderer,
  CustomChart,
  ScatterChart,
  GridComponent,
  DataZoomComponent,
  MarkLineComponent,
  TooltipComponent
]);

const CHART_CONFIG = Object.freeze({
  markerLabel: {
    padding: [2, 6],
    fontSize: 12,
    fontFamily: 'sans-serif',
    borderWidth: 1,
    distance: 6,
    offsetY: -4,
    position: 'start',
    align: 'left',
    verticalAlign: 'bottom',
    rotate: 18,
    color: '#1d4ed8',
    backgroundColor: '#dbeafe',
    borderColor: 'rgba(37, 99, 235, 0.35)'
  },
  markerDelete: {
    radius: 10,
    offsetX: 0,
    offsetY: 0
  },
  markerHoverThreshold: 10,
  markerLabelRightRatio: 0.7,
  markerLabelThemes: {
    event: {
      textColor: '#1d4ed8',
      backgroundColor: '#dbeafe',
      borderColor: '#2563eb'
    },
    warning: {
      textColor: '#92400e',
      backgroundColor: '#fef3c7',
      borderColor: '#f59e0b'
    },
    error: {
      textColor: '#b91c1c',
      backgroundColor: '#fee2e2',
      borderColor: '#ef4444'
    }
  },
  slider: {
    top: 16,
    height: 22,
    gap: 6,
    labelGap: 52,
    bottom: 18,
    bottomHeight: 22
  },
  grid: {
    left: 110,
    right: 62,
    bottom: 124
  },
  index: {
    grid: {
      main: 0,
      pinned: 1,
      label: 2,
      cursor: 3
    },
    axis: {
      main: 0,
      pinned: 1,
      label: 2,
      cursor: 3
    },
    sliderAxis: 4
  },
  axisStyle: {
    labelColor: '#475569',
    lineColor: '#cbd5e1',
    splitLineColor: '#e2e8f0'
  },
  lineStyle: {
    selection: { width: 2, color: '#f97316', type: 'solid' },
    marker: { width: 2, color: '#2563eb', type: 'dashed' },
    markerLabel: {
      width: 1,
      color: 'rgba(37, 99, 235, 0.02)',
      type: 'dashed'
    },
    cursor: { width: 2, color: '#ef4444', type: 'solid' }
  }
});

export default {
  name: 'SyTimelineChart',
  components: {
    'v-chart': VChart
  },
  props: {
    markers: {
      type: Array,
      default: () => []
    },
    timelineData: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {
      chart: null,
      chartOption: {
        animation: false,
        series: []
      },
      isBuildingOption: false, // 옵션 계산 중 chart.getOption() 재호출 방지용 플래그
      markerButtons: [],
      categories: [],
      pinnedCategory: null,
      scrollCategories: [],
      pinnedIntervals: [],
      scrollIntervals: [],
      hoveredMarkerId: null,
      hoverClearTimer: null,
      isHoveringMarkerLine: false,
      isHoveringMarkerLabel: false,
      isHoveringMarkerButton: false,
      selectedCategoryIndex: null,
      domainStart: 0,
      domainEnd: 0,
      viewStart: 0,
      viewEnd: 0,
      cursorStart: 0,
      cursorEnd: 0,
      isSyncingTopSlider: false,
      layoutCache: null
    };
  },
  watch: {
    markers: {
      deep: true,
      handler() {
        this.updateMarkerLines();
        this.updateMarkerButtons();
      }
    },
    timelineData: {
      deep: true,
      immediate: true,
      handler(value) {
        this.setData(value);
        this.refreshChart();
      }
    }
  },
  mounted() {
    window.addEventListener('resize', this.resizeChart, { passive: true });
    this.$nextTick(() => {
      this.getChartInstance();
      this.refreshChart();
    });
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.resizeChart);
    this.teardownChart();
  },
  methods: {
    // =====================
    // 차트 라이프사이클
    // =====================
    getChartInstance() {
      if (this.chart && typeof this.chart.getModel === 'function') {
        return this.chart;
      }
      const ref = this.$refs.chart;
      const candidate = ref && ref.chart ? (ref.chart.value || ref.chart) : null;
      if (candidate && typeof candidate.getModel === 'function') {
        this.chart = candidate;
        return candidate;
      }
      return null;
    },
    onChartReady(chart) {
      this.chart = chart;
      this.refreshChart();
    },
    teardownChart() {
      if (!this.chart) return;
      this.cancelHoverClear();
      this.chart = null;
    },
    resizeChart() {
      const chart = this.getChartInstance();
      if (!chart) return;
      chart.resize();
      this.updateGridLayout();
      this.updateMarkerButtons();
    },
    // =====================
    // 데이터 갱신
    // =====================
    setData(payload = {}) {
      const categories = Array.isArray(payload.categories) ? payload.categories : [];
      const intervals = Array.isArray(payload.intervals) ? payload.intervals : [];
      const resolvedDomain = this.resolveDomain(payload, intervals);
      const span = resolvedDomain.end - resolvedDomain.start;

      this.categories = categories;
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
      this.selectedCategoryIndex =
        typeof payload.selectedCategoryIndex === 'number'
          ? payload.selectedCategoryIndex
          : null;
      this.domainStart = resolvedDomain.start;
      this.domainEnd = resolvedDomain.end;

      // 초기 화면에 보이는 시간창을 전체 범위의 일부(22%~42%)로 설정
      const defaultViewStart =
        span > 0 ? resolvedDomain.start + span * 0.22 : resolvedDomain.start;
      const defaultViewEnd =
        span > 0 ? resolvedDomain.start + span * 0.42 : resolvedDomain.end;
      // 그 시간창 안에서 커서(선) 기본 위치를 더 좁은 구간(28%~35%)로 설정
      const defaultCursorStart =
        span > 0 ? resolvedDomain.start + span * 0.28 : resolvedDomain.start;
      const defaultCursorEnd =
        span > 0 ? resolvedDomain.start + span * 0.35 : resolvedDomain.end;

      this.viewStart = this.normalizeTime(payload.viewStart, defaultViewStart);
      this.viewEnd = this.normalizeTime(payload.viewEnd, defaultViewEnd);
      this.cursorStart = this.normalizeTime(
        payload.cursorStart,
        defaultCursorStart
      );
      this.cursorEnd = this.normalizeTime(payload.cursorEnd, defaultCursorEnd);
    },
    refreshChart() {
      const chart = this.getChartInstance();
      this.isBuildingOption = true;
      let option;
      try {
        option = this.buildBaseOption();
      } finally {
        this.isBuildingOption = false;
      }
      this.chartOption = option;
      if (!chart) return;
      this.applyOptionPatch(option, {
        notMerge: true,
        lazyUpdate: false
      });
      this.updateGridLayout();
      this.updateMarkerLines();
      this.updateSelectionLines();
      this.updateMarkerButtons();
      this.syncTopSlider();
    },
    // =====================
    // 데이터 준비
    // =====================
    resolveDomain(payload, intervals) {
      const providedStart = this.toTimeValue(payload.domainStart);
      const providedEnd = this.toTimeValue(payload.domainEnd);
      const intervalDomain = this.getIntervalDomain(intervals);

      const start =
        providedStart != null ? providedStart : intervalDomain.start;
      const end = providedEnd != null ? providedEnd : intervalDomain.end;

      return {
        start,
        end: end < start ? start : end
      };
    },
    getIntervalDomain(intervals) {
      let min = null;
      let max = null;

      intervals.forEach((item) => {
        if (!item || !item.value) return;
        const start = this.toTimeValue(item.value[1]);
        const end = this.toTimeValue(item.value[2]);
        if (start != null) {
          min = min == null ? start : Math.min(min, start);
        }
        if (end != null) {
          max = max == null ? end : Math.max(max, end);
        }
      });

      if (min == null || max == null) {
        return { start: 0, end: 0 };
      }

      return { start: min, end: max };
    },
    normalizeTime(value, fallback) {
      const parsed = this.toTimeValue(value);
      return parsed != null ? parsed : fallback;
    },
    clamp(value, min, max) {
      return Math.min(max, Math.max(min, value));
    },
    getClampedCursorRange() {
      const topMin = this.viewStart != null ? this.viewStart : this.domainStart;
      const topMax = this.viewEnd != null ? this.viewEnd : this.domainEnd;
      const cursorStart = Math.min(this.cursorStart, this.cursorEnd);
      const cursorEnd = Math.max(this.cursorStart, this.cursorEnd);
      const clampedStart = this.clamp(cursorStart, topMin, topMax);
      const clampedEnd = this.clamp(cursorEnd, topMin, topMax);

      return {
        topMin,
        topMax,
        start: clampedStart,
        end: clampedEnd
      };
    },
    // =====================
    // 차트 옵션
    // =====================
    // 슬라이더/라벨/핀/메인 영역 세로 레이아웃 계산
    getLayoutMetrics() {
      const { grid, slider } = CHART_CONFIG;
      const sliderBottom = slider.top + slider.height + slider.gap;
      const gridBottom = grid.bottom;
      const chartHeight = this.chart ? this.chart.getHeight() : 640;
      const labelHeight = Math.max(0, slider.labelGap);
      const usableHeight = Math.max(
        0,
        chartHeight - gridBottom - sliderBottom - labelHeight
      );

      const hasPinned = Boolean(this.pinnedCategory);
      const visibleMain = Math.max(this.getVisibleMainCount(), 1);
      const totalVisible = visibleMain + (hasPinned ? 1 : 0);
      const rowHeight = totalVisible ? usableHeight / totalVisible : usableHeight;
      const pinnedHeight = hasPinned ? rowHeight : 0;
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
    getDataZoomLayout(layout) {
      const { slider } = CHART_CONFIG;
      return {
        top: {
          top: slider.top,
          height: slider.height
        },
        bottom: {
          bottom: slider.bottom,
          height: slider.bottomHeight
        },
        y: {
          top: layout.mainTop,
          bottom: layout.gridBottom
        }
      };
    },
    buildBaseOption() {
      const layout = this.getLayoutMetrics();
      const cursorRange = this.getClampedCursorRange();

      return {
        backgroundColor: '#ffffff',
        animation: false,
        tooltip: {
          trigger: 'item',
          triggerOn: 'mousemove',
          confine: false,
          appendToBody: true,
          className: 'sy-timeline-tooltip',
          backgroundColor: '#1f2937',
          borderWidth: 0,
          padding: 8,
          textStyle: {
            color: '#ffffff',
            fontSize: 12
          },
          extraCssText: 'box-shadow: 0 10px 20px rgba(15,23,42,0.2); pointer-events: none;',
          formatter: (params) => this.getTimelineTooltipHtml(params)
        },
        dataZoom: this.buildDataZoomOptions(layout, cursorRange),
        grid: this.buildGridOptions(layout),
        xAxis: this.buildXAxisOptions(cursorRange),
        yAxis: this.buildYAxisOptions(),
        series: this.buildSeriesOptions(cursorRange)
      };
    },
    // 상단/하단/inside/Y축 스크롤용 dataZoom 묶음 생성
    buildDataZoomOptions(layout, cursorRange) {
      const dataZoomLayout = this.getDataZoomLayout(layout);
      const { index } = CHART_CONFIG;
      const yWindow = this.getYAxisScrollWindow();

      return [
        {
          id: 'dz_top',
          type: 'slider',
          xAxisIndex: index.sliderAxis,
          filterMode: 'weakFilter',
          showDataShadow: false,
          brushSelect: false,
          ...dataZoomLayout.top,
          labelFormatter: '',
          startValue: cursorRange.start,
          endValue: cursorRange.end
        },
        {
          id: 'dz_bottom',
          type: 'slider',
          xAxisIndex: [index.axis.main, index.axis.pinned],
          filterMode: 'weakFilter',
          showDataShadow: false,
          brushSelect: false,
          ...dataZoomLayout.bottom,
          labelFormatter: '',
          startValue: this.viewStart,
          endValue: this.viewEnd
        },
        {
          id: 'dz_inside',
          type: 'inside',
          xAxisIndex: [index.axis.main, index.axis.pinned],
          filterMode: 'weakFilter',
          zoomOnMouseWheel: false,
          moveOnMouseWheel: false,
          moveOnMouseMove: false
        },
        {
          id: 'dz_y',
          type: 'slider',
          yAxisIndex: 0,
          filterMode: 'weakFilter',
          showDataShadow: false,
          brushSelect: false,
          zoomLock: true,
          showDetail: false,
          handleSize: 0,
          moveHandleSize: 0,
          startValue: yWindow.startValue,
          endValue: yWindow.endValue,
          right: 8,
          ...dataZoomLayout.y,
          width: 12
        }
      ];
    },
    buildXAxisOptions(cursorRange) {
      const minuteInterval = 60 * 1000;
      const { axisStyle, index } = CHART_CONFIG;

      return [
        {
          id: 'x-main',
          gridIndex: index.grid.main,
          type: 'time',
          min: this.domainStart,
          max: this.domainEnd,
          scale: true,
          minInterval: minuteInterval,
          splitNumber: 24,
          axisLine: {
            lineStyle: {
              color: axisStyle.lineColor
            }
          },
          axisLabel: {
            color: axisStyle.labelColor,
            rotate: 90,
            margin: 6,
            inside: false,
            hideOverlap: true,
            interval: 0,
            formatter: (value) => this.formatAxisLabel(value)
          },
          splitLine: {
            show: true,
            lineStyle: {
              color: axisStyle.splitLineColor
            }
          }
        },
        this.buildHiddenTimeAxis({
          id: 'x-pinned',
          gridIndex: index.grid.pinned,
          min: this.domainStart,
          max: this.domainEnd,
          minInterval: minuteInterval,
          splitNumber: 24
        }),
        this.buildHiddenTimeAxis({
          id: 'x-label',
          gridIndex: index.grid.label,
          min: cursorRange.topMin,
          max: cursorRange.topMax,
          minInterval: minuteInterval,
          splitNumber: 24
        }),
        this.buildHiddenTimeAxis({
          id: 'x-cursor',
          gridIndex: index.grid.cursor,
          min: cursorRange.topMin,
          max: cursorRange.topMax
        }),
        this.buildHiddenTimeAxis({
          id: 'x-slider',
          gridIndex: index.grid.cursor,
          min: cursorRange.topMin,
          max: cursorRange.topMax
        })
      ];
    },
    buildYAxisOptions() {
      const { index } = CHART_CONFIG;
      return [
        this.buildCategoryAxis({
          id: 'y-main',
          gridIndex: index.grid.main,
          data: this.scrollCategories
        }),
        this.buildCategoryAxis({
          id: 'y-pinned',
          gridIndex: index.grid.pinned,
          data: this.pinnedCategory ? [this.pinnedCategory] : []
        }),
        this.buildHiddenValueAxis({
          id: 'y-label',
          gridIndex: index.grid.label,
          min: 0,
          max: 1
        }),
        this.buildHiddenValueAxis({
          id: 'y-cursor',
          gridIndex: index.grid.cursor,
          min: 0,
          max: 1
        })
      ];
    },
    // 실제 데이터 + 선택/마커/커서용 보조 시리즈 구성
    buildSeriesOptions(cursorRange) {
      const { index } = CHART_CONFIG;
      return [
        this.buildBarSeries({
          id: 'sy-bars-main',
          xAxisIndex: index.axis.main,
          yAxisIndex: index.axis.main,
          data: this.scrollIntervals
        }),
        this.buildBarSeries({
          id: 'sy-bars-pinned',
          xAxisIndex: index.axis.pinned,
          yAxisIndex: index.axis.pinned,
          data: this.pinnedIntervals
        }),
        this.buildSelectionSeries({
          id: 'sy-selection-main',
          xAxisIndex: index.axis.main,
          yAxisIndex: index.axis.main
        }),
        this.buildSelectionSeries({
          id: 'sy-selection-pinned',
          xAxisIndex: index.axis.pinned,
          yAxisIndex: index.axis.pinned
        }),
        this.buildMarkerLineSeries({
          id: 'sy-markers-main',
          xAxisIndex: index.axis.main,
          yAxisIndex: index.axis.main,
          silent: false
        }),
        this.buildMarkerLineSeries({
          id: 'sy-markers-pinned-lines',
          xAxisIndex: index.axis.pinned,
          yAxisIndex: index.axis.pinned,
          silent: true
        }),
        this.buildHoverMarkerLineSeries({
          id: 'sy-markers-hover-main',
          xAxisIndex: index.axis.main,
          yAxisIndex: index.axis.main
        }),
        this.buildHoverMarkerLineSeries({
          id: 'sy-markers-hover-pinned',
          xAxisIndex: index.axis.pinned,
          yAxisIndex: index.axis.pinned
        }),
        this.buildMarkerLabelSeries(),
        this.buildRightMarkerLabelSeries(),
        this.buildHoverMarkerLabelSeries(),
        this.buildCursorOverlaySeries(cursorRange)
      ];
    },
    buildBarSeries(options) {
      const { id, xAxisIndex, yAxisIndex, data } = options;
      return {
        id,
        type: 'custom',
        renderItem: this.renderItem,
        xAxisIndex,
        yAxisIndex,
        itemStyle: {
          opacity: 0.85
        },
        encode: {
          x: [1, 2],
          y: 0
        },
        data
      };
    },
    buildSelectionSeries(options) {
      const { id, xAxisIndex, yAxisIndex } = options;
      const { lineStyle } = CHART_CONFIG;
      return {
        id,
        type: 'scatter',
        xAxisIndex,
        yAxisIndex,
        data: [],
        silent: true,
        markLine: {
          symbol: ['none', 'none'],
          z: 18,
          lineStyle: {
            ...lineStyle.selection
          },
          label: {
            show: false
          },
          data: []
        }
      };
    },
    buildMarkerLineSeries(options) {
      const { id, xAxisIndex, yAxisIndex, silent } = options;
      const { lineStyle } = CHART_CONFIG;
      return {
        id,
        type: 'scatter',
        xAxisIndex,
        yAxisIndex,
        data: [],
        silent,
        emphasis: {
          disabled: true
        },
        markLine: {
          symbol: ['none', 'none'],
          precision: 0,
          z: 20,
          lineStyle: {
            ...lineStyle.marker
          },
          emphasis: {
            disabled: true
          },
          label: {
            show: false
          },
          data: []
        }
      };
    },
    buildHoverMarkerLineSeries(options) {
      const { id, xAxisIndex, yAxisIndex } = options;
      const { lineStyle } = CHART_CONFIG;
      return {
        id,
        type: 'scatter',
        xAxisIndex,
        yAxisIndex,
        data: [],
        silent: true,
        emphasis: {
          disabled: true
        },
        markLine: {
          symbol: ['none', 'none'],
          precision: 0,
          z: 30,
          lineStyle: {
            ...lineStyle.marker
          },
          emphasis: {
            disabled: true
          },
          label: {
            show: false
          },
          data: []
        }
      };
    },
    buildMarkerLabelSeries() {
      return this.buildMarkerLabelSeriesConfig({
        id: 'sy-markers-labels',
        zIndex: 22,
        silent: false
      });
    },
    buildRightMarkerLabelSeries() {
      return this.buildMarkerLabelSeriesConfig({
        id: 'sy-markers-labels-right',
        zIndex: 24,
        silent: false
      });
    },
    buildHoverMarkerLabelSeries() {
      return this.buildMarkerLabelSeriesConfig({
        id: 'sy-markers-hover-labels',
        zIndex: 32,
        silent: false
      });
    },
    buildMarkerLabelSeriesConfig(options) {
      const { markerLabel, lineStyle, index } = CHART_CONFIG;
      const id = options.id;
      const zIndex = options.zIndex;
      const silent = options.silent ?? false;

      const defaultTheme = CHART_CONFIG.markerLabelThemes.event;

      return {
        id,
        type: 'scatter',
        xAxisIndex: index.axis.label,
        yAxisIndex: index.axis.label,
        data: [],
        silent,
        emphasis: {
          disabled: true
        },
        markLine: {
          symbol: ['none', 'none'],
          precision: 0,
          z: zIndex,
          lineStyle: {
            ...lineStyle.markerLabel
          },
          emphasis: {
            disabled: true
          },
          label: {
            show: true,
            position: markerLabel.position,
            align: markerLabel.align,
            verticalAlign: markerLabel.verticalAlign,
            distance: markerLabel.distance,
            offset: [0, markerLabel.offsetY],
            rotate: markerLabel.rotate,
            color: defaultTheme.textColor,
            backgroundColor: defaultTheme.backgroundColor,
            borderColor: defaultTheme.borderColor,
            borderWidth: markerLabel.borderWidth,
            padding: markerLabel.padding,
            borderRadius: 6,
            fontSize: markerLabel.fontSize,
            fontFamily: markerLabel.fontFamily,
            formatter: (params) => {
              return this.getMarkerLabelText(params);
            }
          },
          data: []
        }
      };
    },
    buildCursorOverlaySeries(cursorRange) {
      const { index, lineStyle } = CHART_CONFIG;
      return {
        id: 'sy-cursors-overlay',
        type: 'scatter',
        xAxisIndex: index.axis.cursor,
        yAxisIndex: index.axis.cursor,
        data: [],
        silent: true,
        markLine: {
          symbol: ['none', 'none'],
          z: 15,
          lineStyle: {
            ...lineStyle.cursor
          },
          label: {
            show: false
          },
          data: [
            { xAxis: cursorRange.start },
            { xAxis: cursorRange.end }
          ]
        }
      };
    },
    createHiddenAxisParts() {
      return {
        axisLine: { show: false },
        axisTick: { show: false },
        axisLabel: { show: false },
        splitLine: { show: false }
      };
    },
    buildHiddenTimeAxis(options) {
      const { id, gridIndex, min, max, minInterval, splitNumber, show = false } =
        options;
      const axis = {
        id,
        gridIndex,
        type: 'time',
        min,
        max,
        scale: true,
        show,
        ...this.createHiddenAxisParts()
      };
      if (minInterval != null) {
        axis.minInterval = minInterval;
      }
      if (splitNumber != null) {
        axis.splitNumber = splitNumber;
      }
      return axis;
    },
    buildHiddenValueAxis(options) {
      const { id, gridIndex, min, max } = options;
      return {
        id,
        gridIndex,
        type: 'value',
        min,
        max,
        ...this.createHiddenAxisParts()
      };
    },
    buildCategoryAxis(options) {
      const { axisStyle } = CHART_CONFIG;
      const { id, gridIndex, data } = options;
      return {
        id,
        gridIndex,
        type: 'category',
        inverse: true,
        data,
        axisTick: { show: false },
        axisLine: { show: false },
        axisLabel: { color: axisStyle.labelColor }
      };
    },
    buildGridOptions(layout) {
      const { grid } = CHART_CONFIG;
      return [
        {
          id: 'grid-main',
          top: layout.mainTop,
          bottom: layout.gridBottom,
          left: grid.left,
          right: grid.right,
          containLabel: false
        },
        {
          id: 'grid-pinned',
          top: layout.pinnedTop,
          height: layout.pinnedHeight,
          left: grid.left,
          right: grid.right,
          containLabel: false
        },
        {
          id: 'grid-label',
          top: layout.labelTop,
          height: layout.labelHeight,
          left: grid.left,
          right: grid.right,
          containLabel: false
        },
        {
          id: 'grid-cursor',
          top: layout.sliderBottom,
          bottom: layout.gridBottom,
          left: grid.left,
          right: grid.right,
          containLabel: false
        }
      ];
    },
    applyOptionPatch(option, config) {
      const chart = this.getChartInstance();
      if (!chart) return;
      const settings = {
        notMerge: false,
        lazyUpdate: true
      };
      if (config) {
        Object.assign(settings, config);
      }
      chart.setOption(option, settings);
    },
    updateGridLayout() {
      this.getChartInstance();
      if (!this.chart) return;
      const layout = this.getLayoutMetrics();
      if (!this.hasLayoutChanged(layout)) return;
      const dataZoomLayout = this.getDataZoomLayout(layout);

      this.applyOptionPatch(
        {
          grid: this.buildGridOptions(layout),
          dataZoom: [
            {
              id: 'dz_top',
              ...dataZoomLayout.top
            },
            {
              id: 'dz_bottom',
              ...dataZoomLayout.bottom
            },
            {
              id: 'dz_y',
              ...dataZoomLayout.y
            }
          ]
        },
        {
          lazyUpdate: false
        }
      );
    },
    hasLayoutChanged(layout) {
      const keys = [
        'sliderBottom',
        'gridBottom',
        'labelTop',
        'labelHeight',
        'pinnedHeight',
        'pinnedTop',
        'mainTop'
      ];
      const cache = this.layoutCache;

      if (!cache) {
        this.layoutCache = { ...layout };
        return true;
      }

      for (let index = 0; index < keys.length; index += 1) {
        const key = keys[index];
        if (cache[key] !== layout[key]) {
          this.layoutCache = { ...layout };
          return true;
        }
      }

      return false;
    },
    getYAxisScrollWindow() {
      const total = this.scrollCategories.length;
      if (!total) return { startValue: 0, endValue: 0 };
      const maxVisible = 12;
      return {
        startValue: 0,
        endValue: Math.min(total - 1, maxVisible - 1)
      };
    },
    getVisibleMainCount() {
      const total = this.scrollCategories.length;
      if (!total) return 0;

      const dz = this.getYAxisZoom();
      const fallback = this.getYAxisScrollWindow();
      const startValue =
        dz && dz.startValue != null ? dz.startValue : fallback.startValue;
      const endValue =
        dz && dz.endValue != null ? dz.endValue : fallback.endValue;

      const start = this.clamp(Math.round(startValue), 0, total - 1);
      const end = this.clamp(Math.round(endValue), 0, total - 1);
      return Math.max(1, end - start + 1);
    },
    getYAxisZoom() {
      if (this.isBuildingOption) return null;
      const chart = this.getChartInstance();
      if (!chart) return null;
      const option = chart.getOption();
      const dataZoom = option && option.dataZoom;
      if (!Array.isArray(dataZoom)) return null;
      return dataZoom.find((item) => item && item.id === 'dz_y') ?? null;
    },
    // =====================
    // 시리즈 렌더링
    // =====================
    // custom series: 막대를 직접 그리기
    renderItem(params, api) {
      const categoryIndex = api.value(0);
      const start = api.coord([api.value(1), categoryIndex]);
      const end = api.coord([api.value(2), categoryIndex]);
      const height = api.size([0, 1])[1] * 0.6;

      const rectShape = graphic.clipRectByRect(
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
    },
    // =====================
    // 차트 인터랙션
    // =====================
    getWheelDelta(event) {
      if (!event) return 0;
      if (typeof event.wheelDelta === 'number') {
        return event.wheelDelta;
      }
      if (typeof event.deltaY === 'number') {
        return -event.deltaY;
      }
      return 0;
    },
    scrollYAxis(event) {
      const chart = this.getChartInstance();
      if (!chart || !event) return;
      const total = this.scrollCategories.length;
      if (!total) return;

      const delta = this.getWheelDelta(event);
      if (!delta) return;

      const direction = delta > 0 ? -1 : 1;
      const fallback = this.getYAxisScrollWindow();
      const dz = this.getYAxisZoom();
      let startValue =
        dz && dz.startValue != null ? dz.startValue : fallback.startValue;
      let endValue = dz && dz.endValue != null ? dz.endValue : fallback.endValue;

      startValue = Math.round(startValue);
      endValue = Math.round(endValue);
      const window = Math.max(0, endValue - startValue);
      const maxStart = Math.max(0, total - 1 - window);
      const nextStart = this.clamp(startValue + direction, 0, maxStart);
      const nextEnd = nextStart + window;

      if (nextStart === startValue && nextEnd === endValue) return;

      chart.dispatchAction({
        type: 'dataZoom',
        dataZoomId: 'dz_y',
        startValue: nextStart,
        endValue: nextEnd
      });
    },
    zoomXAxis(event) {
      const chart = this.getChartInstance();
      if (!chart || !event) return;

      const axisMin = this.domainStart;
      const axisMax = this.domainEnd;
      if (axisMin == null || axisMax == null || axisMax <= axisMin) return;

      const currentStart = this.viewStart != null ? this.viewStart : axisMin;
      const currentEnd = this.viewEnd != null ? this.viewEnd : axisMax;
      const span = Math.max(1, currentEnd - currentStart);
      const delta = this.getWheelDelta(event);
      if (!delta) return;

      const minSpan = 5 * 60 * 1000;
      const fullSpan = axisMax - axisMin;
      const zoomIn = delta > 0;
      const scale = zoomIn ? 0.85 : 1.15;
      const nextSpan = this.clamp(span * scale, minSpan, fullSpan);

      const offsetX = event.offsetX;
      const offsetY = event.offsetY;
      let focus = currentStart + span / 2;
      if (offsetX != null && offsetY != null) {
        const value = chart.convertFromPixel(
          { xAxisIndex: CHART_CONFIG.index.axis.main },
          [offsetX, offsetY]
        );
        const candidate = Array.isArray(value) ? value[0] : value;
        if (typeof candidate === 'number' && !Number.isNaN(candidate)) {
          focus = candidate;
        }
      }

      const ratio = span > 0 ? (focus - currentStart) / span : 0.5;
      let nextStart = focus - ratio * nextSpan;
      let nextEnd = nextStart + nextSpan;

      if (nextStart < axisMin) {
        nextStart = axisMin;
        nextEnd = nextStart + nextSpan;
      }
      if (nextEnd > axisMax) {
        nextEnd = axisMax;
        nextStart = nextEnd - nextSpan;
      }

      chart.dispatchAction({
        type: 'dataZoom',
        dataZoomId: 'dz_bottom',
        startValue: Math.round(nextStart),
        endValue: Math.round(nextEnd)
      });
    },
    getDataZoomId(item) {
      if (!item) return null;
      if (item.dataZoomId) return item.dataZoomId;
      const chart = this.getChartInstance();
      if (typeof item.dataZoomIndex === 'number' && chart) {
        const option = chart.getOption();
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
      const dataZoomId = options.dataZoomId ?? 'dz_bottom';
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

      const chart = this.getChartInstance();
      if (!chart) {
        return { start: fallbackStart, end: fallbackEnd };
      }

      const option = chart.getOption();
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
/**
 * 상단 슬라이더(dz_top)의 활성 구간(커서 범위)을 시간값으로 지정합니다.
 * @param {number|string|Date} startTime
 * @param {number|string|Date} endTime
 */
setTopSliderRange(startTime, endTime) {
  this.getChartInstance();
  if (!this.chart) return;

  const topMin = this.viewStart != null ? this.viewStart : this.domainStart;
  const topMax = this.viewEnd != null ? this.viewEnd : this.domainEnd;

  const startValueRaw = this.toTimeValue(startTime);
  const endValueRaw = this.toTimeValue(endTime);
  if (startValueRaw == null || endValueRaw == null) return;

  const startValue = this.clamp(Math.min(startValueRaw, endValueRaw), topMin, topMax);
  const endValue = this.clamp(Math.max(startValueRaw, endValueRaw), topMin, topMax);

  // 내부 루프 방지 플래그 (onDataZoom에서 dz_top 처리할 때 걸러짐)
  this.isSyncingTopSlider = true;

  this.chart.dispatchAction({
    type: 'dataZoom',
    dataZoomId: 'dz_top',
    startValue: Math.round(startValue),
    endValue: Math.round(endValue)
  });

  // state도 함께 맞춰둠(즉시 반영 + 다른 로직과 일관성)
  this.cursorStart = startValue;
  this.cursorEnd = endValue;

  // 라인/라벨/버튼 갱신
  this.updateCursorLines();
  this.updateMarkerLines();
  this.updateMarkerButtons();

  setTimeout(() => {
    this.isSyncingTopSlider = false;
  }, 0);
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
        const zoomId = this.getDataZoomId(batch) ?? this.getDataZoomId(event);

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
            dataZoomId: zoomId ?? 'dz_bottom',
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
        this.updateMarkerLines();
      }
      if (viewUpdated || cursorUpdated) {
        this.updateMarkerButtons();
      }
    },
    onChartClick(params) {
      if (!params || params.seriesType !== 'custom') return;
      if (!params.value || typeof params.value[0] !== 'number') return;

      let categoryIndex = null;
      const isPinned = params.seriesId === 'sy-bars-pinned';
      const isMain = params.seriesId === 'sy-bars-main';

      if (isPinned) {
        categoryIndex = 0;
      } else if (isMain) {
        categoryIndex = params.value[0] + 1;
      }

      if (categoryIndex == null) return;
      this.selectedCategoryIndex = categoryIndex;
      this.updateSelectionLines();

      const payload = {
        seriesId: params.seriesId,
        categoryIndex,
        categoryName: this.resolveCategoryName(params),
        value: params.value,
        data: params.data
      };
      this.$emit('item-click', payload);
    },
    onChartMouseOver(params) {
      const markerId = this.getMarkerIdFromEvent(params);
      if (markerId == null) return;
      this.isHoveringMarkerLabel = true;
      this.cancelHoverClear();
      this.setHoveredMarker(markerId);
    },
    onChartMouseOut(params) {
      const markerId = this.getMarkerIdFromEvent(params);
      if (markerId == null) return;
      this.isHoveringMarkerLabel = false;
      this.scheduleHoverClear();
    },
    onChartWheel(event) {
      const nativeEvent = event && event.event ? event.event : null;
      if (nativeEvent && nativeEvent.ctrlKey) {
        if (typeof nativeEvent.preventDefault === 'function') {
          nativeEvent.preventDefault();
        }
        this.zoomXAxis(event);
        return;
      }
      this.scrollYAxis(event);
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

      this.applyOptionPatch({
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
    updateMarkerLines() {
      this.getChartInstance();
      if (!this.chart) return;

      const markerLines = [];
      const baseLabelLines = [];
      const rightLabelLines = [];
      const hoverLineData = [];
      const hoverLabelData = [];
      const markers = this.markers ?? [];
      const rightThreshold = this.getRightLabelThreshold();
      const hoveredId = this.hoveredMarkerId;

      markers.forEach((marker) => {
        const linePoint = this.buildMarkerLinePoint(marker);
        if (!linePoint) return;
        markerLines.push(linePoint);

        const isHovered =
          hoveredId != null && marker && marker.id === hoveredId;
        const labelEntry = this.buildMarkerLabelEntry(marker, !isHovered);
        if (labelEntry) {
          if (rightThreshold != null && labelEntry.xAxis >= rightThreshold) {
            rightLabelLines.push(labelEntry);
          } else {
            baseLabelLines.push(labelEntry);
          }
        }

        if (isHovered) {
          const hoverLine = this.buildMarkerLinePoint(marker);
          if (hoverLine) {
            hoverLineData.push(hoverLine);
          }
          const hoverLabel = this.buildMarkerLabelEntry(marker, true);
          if (hoverLabel) {
            hoverLabelData.push(hoverLabel);
          }
        }
      });

      this.applyOptionPatch({
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
              data: baseLabelLines
            }
          },
          {
            id: 'sy-markers-labels-right',
            markLine: {
              data: rightLabelLines
            }
          },
          {
            id: 'sy-markers-hover-main',
            markLine: {
              data: hoverLineData
            }
          },
          {
            id: 'sy-markers-hover-pinned',
            markLine: {
              data: hoverLineData
            }
          },
          {
            id: 'sy-markers-hover-labels',
            markLine: {
              data: hoverLabelData
            }
          }
        ]
      });
    },
    updateCursorLines() {
      this.getChartInstance();
      if (!this.chart) return;
      const cursorRange = this.getClampedCursorRange();

      this.applyOptionPatch({
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
    // 상단 슬라이더와 커서 범위 동기화 (루프 방지 플래그 사용)
    syncTopSlider() {
      this.getChartInstance();
      if (!this.chart) return;

      const cursorRange = this.getClampedCursorRange();

      this.isSyncingTopSlider = true;
      this.applyOptionPatch({
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
    // 마커 라인 좌표 → DOM 버튼 위치로 변환
    updateMarkerButtons() {
      this.getChartInstance();
      if (!this.chart) {
        this.markerButtons = [];
        return;
      }

      const { index, markerDelete } = CHART_CONFIG;
      const markers = this.getHoveredMarkers();
      const mainGridRect = this.getGridRect(index.grid.main);
      const pinnedGridRect = this.getGridRect(index.grid.pinned);
      const buttons = [];

      if (!markers.length) {
        this.hoveredMarkerId = null;
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
        const leftBound = markerDelete.radius;
        const rightBound = chartWidth - markerDelete.radius;
        const topBound = markerDelete.radius;
        const bottomBound = chartHeight - markerDelete.radius;

        markers.forEach((marker) => {
          const markerTime = this.toTimeValue(marker.time);
          if (markerTime == null) return;

          const lineX = this.getAxisPixelFromValue(
            markerTime,
            index.axis.main
          );
          if (lineX == null || Number.isNaN(lineX)) return;
          if (
            lineX < mainGridRect.x ||
            lineX > mainGridRect.x + mainGridRect.width
          ) {
            return;
          }

          const rawButtonX = lineX + markerDelete.offsetX;
          const rawButtonY = lineCenterY + markerDelete.offsetY;
          const buttonX = this.clamp(rawButtonX, leftBound, rightBound);
          const buttonY = this.clamp(rawButtonY, topBound, bottomBound);

          buttons.push({
            id: marker.id,
            x: buttonX,
            y: buttonY
          });
        });
      }

      this.markerButtons = buttons;
    },
    // =====================
    // ZRender 이벤트
    // =====================
    onZrMouseMove(event) {
      if (!event) return;
      if (this.isHoveringMarkerButton) return;

      const markerId = this.findHoveredMarkerId(event);
      if (markerId != null) {
        this.isHoveringMarkerLine = true;
        this.cancelHoverClear();
        this.setHoveredMarker(markerId);
        return;
      }

      this.isHoveringMarkerLine = false;
      this.scheduleHoverClear();
    },
    onZrGlobalOut() {
      this.isHoveringMarkerLine = false;
      this.isHoveringMarkerLabel = false;
      this.scheduleHoverClear();
    },
    findHoveredMarkerId(event) {
      if (!this.chart) return null;

      const offsetX = event.offsetX;
      const offsetY = event.offsetY;
      if (offsetX == null || offsetY == null) return null;

      const mainGridRect = this.getGridRect(CHART_CONFIG.index.grid.main);
      const pinnedGridRect = this.getGridRect(CHART_CONFIG.index.grid.pinned);
      const gridRects = [mainGridRect, pinnedGridRect].filter(
        (rect) => rect && rect.height > 0
      );

      const isInsideGrid = gridRects.some((rect) => {
        const withinX = offsetX >= rect.x && offsetX <= rect.x + rect.width;
        const withinY = offsetY >= rect.y && offsetY <= rect.y + rect.height;
        return withinX && withinY;
      });

      if (!isInsideGrid) return null;

      const markers = this.markers ?? [];
      const threshold = CHART_CONFIG.markerHoverThreshold;
      let closestId = null;
      let minDistance = threshold + 1;

      markers.forEach((marker) => {
        if (!marker || marker.time == null || marker.id == null) return;
        const markerTime = this.toTimeValue(marker.time);
        if (markerTime == null) return;
        const lineX = this.getAxisPixelFromValue(
          markerTime,
          CHART_CONFIG.index.axis.main
        );
        if (lineX == null) return;
        if (
          mainGridRect &&
          (lineX < mainGridRect.x ||
            lineX > mainGridRect.x + mainGridRect.width)
        ) {
          return;
        }

        const distance = Math.abs(offsetX - lineX);
        if (distance > threshold) return;
        if (marker.id === this.hoveredMarkerId) {
          closestId = marker.id;
          minDistance = distance;
          return;
        }
        if (distance < minDistance) {
          minDistance = distance;
          closestId = marker.id;
        }
      });

      return closestId;
    },
    setHoveredMarker(markerId) {
      if (this.hoveredMarkerId === markerId) return;
      this.hoveredMarkerId = markerId;
      this.applyHoverState();
    },
    clearHoveredMarker() {
      if (this.hoveredMarkerId == null) return;
      this.hoveredMarkerId = null;
      this.applyHoverState();
    },
    applyHoverState() {
      this.updateMarkerLines();
      this.updateMarkerButtons();
    },
    scheduleHoverClear() {
      this.cancelHoverClear();
      if (
        this.isHoveringMarkerLine ||
        this.isHoveringMarkerLabel ||
        this.isHoveringMarkerButton
      ) {
        return;
      }
      this.hoverClearTimer = setTimeout(() => {
        this.hoverClearTimer = null;
        if (
          this.isHoveringMarkerLine ||
          this.isHoveringMarkerLabel ||
          this.isHoveringMarkerButton
        ) {
          return;
        }
        this.clearHoveredMarker();
      }, 0);
    },
    cancelHoverClear() {
      if (this.hoverClearTimer == null) return;
      clearTimeout(this.hoverClearTimer);
      this.hoverClearTimer = null;
    },
    onMarkerButtonEnter(markerId) {
      this.isHoveringMarkerButton = true;
      this.cancelHoverClear();
      this.setHoveredMarker(markerId);
    },
    onMarkerButtonLeave() {
      this.isHoveringMarkerButton = false;
      this.scheduleHoverClear();
    },
    getMarkerIdFromEvent(params) {
      if (!params || params.componentType !== 'markLine') return null;
      const data = params.data;
      return data && data.markerId != null ? data.markerId : null;
    },
    getHoveredMarkers() {
      if (this.hoveredMarkerId == null) return [];
      const markers = this.markers ?? [];
      return markers.filter(
        (marker) =>
          marker &&
          marker.time != null &&
          marker.id === this.hoveredMarkerId
      );
    },
    getRightLabelThreshold() {
      const start = this.viewStart ?? this.domainStart;
      const end = this.viewEnd ?? this.domainEnd;
      if (start == null || end == null) return null;
      const span = end - start;
      if (span <= 0) return null;
      return start + span * CHART_CONFIG.markerLabelRightRatio;
    },
    resolveMarkerType(value) {
      const text = String(value ?? '').toLowerCase();
      if (text === 'warning' || text === 'warn') return 'warning';
      if (text === 'error' || text === 'err') return 'error';
      return 'event';
    },
    getMarkerLabelStyle(marker) {
      const markerType = marker && marker.type != null ? marker.type : '';
      const typeKey = this.resolveMarkerType(markerType);
      const themes = CHART_CONFIG.markerLabelThemes;
      const theme = themes[typeKey] ?? themes.event;
      return {
        color: theme.textColor,
        backgroundColor: theme.backgroundColor,
        borderColor: theme.borderColor
      };
    },
    buildMarkerLinePoint(marker) {
      if (!marker || marker.time == null) return null;
      const markerTime = this.toTimeValue(marker.time);
      if (markerTime == null) return null;
      return {
        xAxis: markerTime,
        markerId: marker.id
      };
    },
    buildMarkerLabelEntry(marker, showLabel) {
      const point = this.buildMarkerLinePoint(marker);
      if (!point) return null;
      const labelStyle = this.getMarkerLabelStyle(marker);
      const labelVisible = showLabel ?? true;

      return {
        xAxis: point.xAxis,
        markerId: point.markerId,
        markerCode: marker.code ?? '',
        markerType: marker.type ?? '',
        label: {
          show: labelVisible,
          color: labelStyle.color,
          backgroundColor: labelStyle.backgroundColor,
          borderColor: labelStyle.borderColor
        }
      };
    },
    getGridRect(gridIndex = CHART_CONFIG.index.grid.main) {
      const chart = this.getChartInstance();
      if (!chart) return null;
      const model = chart.getModel && chart.getModel();
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

      const option = chart.getOption();
      const gridOption = option && option.grid && option.grid[gridIndex];
      if (!gridOption) return null;

      const width = chart.getWidth();
      const height = chart.getHeight();
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
    getAxisPixelFromValue(value, xAxisIndex = CHART_CONFIG.index.axis.main) {
      const chart = this.getChartInstance();
      if (!chart) return null;
      const pixel = chart.convertToPixel({ xAxisIndex }, value);
      return Array.isArray(pixel) ? pixel[0] : pixel;
    },
    // =====================
    // 포맷팅
    // =====================
    formatAxisLabel(value) {
      const date = new Date(value);
      if (Number.isNaN(date.getTime())) return '';

      return [
        `${this.pad2(date.getMonth() + 1)}-${this.pad2(date.getDate())}`,
        `${this.pad2(date.getHours())}:${this.pad2(date.getMinutes())}`
      ].join(' ');
    },
    formatDateTime(value) {
      const date = new Date(value);
      if (Number.isNaN(date.getTime())) return '';

      return [
        `${date.getFullYear()}-${this.pad2(date.getMonth() + 1)}-${this.pad2(date.getDate())}`,
        `${this.pad2(date.getHours())}:${this.pad2(date.getMinutes())}:${this.pad2(date.getSeconds())}`
      ].join(' ');
    },
    pad2(value) {
      return String(value).padStart(2, '0');
    },
    getMarkerLabelText(params) {
      const data = params && params.data ? params.data : null;
      const markerCode = data && data.markerCode != null ? data.markerCode : '';
      if (markerCode) return String(markerCode);
      const xValue = data && data.xAxis != null ? data.xAxis : null;
      return this.formatDateTime(xValue);
    },
    getTimelineTooltipHtml(params) {
      if (!params || !Array.isArray(params.value)) return '';
      const isPinned = params.seriesId === 'sy-bars-pinned';
      const isMain = params.seriesId === 'sy-bars-main';
      if (!isPinned && !isMain) return '';
      return isPinned
        ? this.getPinnedTooltipHtml(params)
        : this.getRowTooltipHtml(params);
    },
    getPinnedTooltipHtml(params) {
      if (!params || !Array.isArray(params.value)) return '';
      const categoryName = this.resolveCategoryName(params);
      const start = this.formatDateTime(params.value[1]);
      const end = this.formatDateTime(params.value[2]);
      const durationMs = params.value[2] - params.value[1];
      const durationMin =
        typeof durationMs === 'number' && !Number.isNaN(durationMs)
          ? Math.max(0, Math.round(durationMs / 60000))
          : null;

      const lines = ['Pinned Row', this.safeText(categoryName)];
      if (params.name) {
        lines.push(`Type: ${params.name}`);
      }
      lines.push(`Start: ${start}`);
      lines.push(`End: ${end}`);
      if (durationMin != null) {
        lines.push(`Duration: ${durationMin} min`);
      }

      return lines.join('<br/>');
    },
    getRowTooltipHtml(params) {
      if (!params || !Array.isArray(params.value)) return '';
      const categoryName = this.resolveCategoryName(params);
      const start = this.formatDateTime(params.value[1]);
      const end = this.formatDateTime(params.value[2]);
      const durationMs = params.value[2] - params.value[1];
      const durationMin =
        typeof durationMs === 'number' && !Number.isNaN(durationMs)
          ? Math.max(0, Math.round(durationMs / 60000))
          : null;

      const lines = ['Row', this.safeText(categoryName)];
      if (params.name) {
        lines.push(`Type: ${params.name}`);
      }
      lines.push(`Start: ${start}`);
      lines.push(`End: ${end}`);
      if (durationMin != null) {
        lines.push(`Duration: ${durationMin} min`);
      }

      return lines.join('<br/>');
    },
    resolveCategoryName(params) {
      if (!params || !Array.isArray(params.value)) return null;
      if (params.seriesId === 'sy-bars-pinned') {
        return this.pinnedCategory || this.categories[0] || null;
      }
      const idx = params.value[0];
      return this.scrollCategories[idx] || null;
    },
    safeText(value) {
      return value == null ? '-' : String(value);
    },
    // =====================
    // 시간 파싱
    // =====================
    toTimeValue(value) {
      if (value == null) return null;
      if (typeof value === 'number') return value;
      if (number && typeof number.parseDate === 'function') {
        const parsed = number.parseDate(value);
        if (parsed && !Number.isNaN(parsed.getTime())) {
          return parsed.getTime();
        }
      }
      const time = new Date(value).getTime();
      return Number.isNaN(time) ? null : time;
    },
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



