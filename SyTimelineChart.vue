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
    backgroundColor: 'rgba(37, 99, 235, 0.12)',
    borderColor: 'rgba(37, 99, 235, 0.35)'
  },
  markerDelete: {
    radius: 10,
    offsetX: 0,
    offsetY: 0
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
      markerButtons: [],
      categories: [],
      pinnedCategory: null,
      scrollCategories: [],
      pinnedIntervals: [],
      scrollIntervals: [],
      activeMarkerId: null,
      ignoreNextZrClick: false,
      dragOrigin: null,
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
    this.initChart();
    window.addEventListener('resize', this.resizeChart, { passive: true });
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.resizeChart);
    this.teardownChart();
  },
  methods: {
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

      const defaultViewStart =
        span > 0 ? resolvedDomain.start + span * 0.22 : resolvedDomain.start;
      const defaultViewEnd =
        span > 0 ? resolvedDomain.start + span * 0.42 : resolvedDomain.end;
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
    initChart() {
      const el = this.$refs.chart;
      if (!el) return;

      this.chart = echarts.init(el);
      this.chart.setOption(this.buildOption(), { notMerge: true });
      this.updateGridLayout();

      this.bindChartEvents();

      this.updateMarkerLines();
      this.updateSelectionLines();
      this.updateMarkerButtons();
      this.syncTopSlider();
    },
    refreshChart() {
      if (!this.chart) return;
      this.chart.setOption(this.buildOption(), { notMerge: true });
      this.updateGridLayout();
      this.updateMarkerLines();
      this.updateSelectionLines();
      this.updateMarkerButtons();
      this.syncTopSlider();
    },
    teardownChart() {
      if (!this.chart) return;
      this.unbindChartEvents();
      this.chart.dispose();
      this.chart = null;
    },
    bindChartEvents() {
      if (!this.chart) return;
      this.chart.on('datazoom', this.onDataZoom);
      this.chart.on('click', this.onChartClick);
      const zr = this.chart.getZr();
      if (!zr) return;
      zr.on('click', this.onZrClick);
      zr.on('mousewheel', this.onChartWheel);
      zr.on('mousedown', this.onZrMouseDown);
      zr.on('mousemove', this.onZrMouseMove);
      zr.on('mouseup', this.onZrMouseUp);
    },
    unbindChartEvents() {
      if (!this.chart) return;
      this.chart.off('datazoom', this.onDataZoom);
      this.chart.off('click', this.onChartClick);
      const zr = this.chart.getZr();
      if (!zr) return;
      zr.off('click', this.onZrClick);
      zr.off('mousewheel', this.onChartWheel);
      zr.off('mousedown', this.onZrMouseDown);
      zr.off('mousemove', this.onZrMouseMove);
      zr.off('mouseup', this.onZrMouseUp);
    },
    resizeChart() {
      if (!this.chart) return;
      this.chart.resize();
      this.updateGridLayout();
      this.updateMarkerButtons();
    },
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
    getYAxisScrollWindow() {
      const total = this.scrollCategories.length;
      if (!total) {
        return { startValue: 0, endValue: 0 };
      }

      const maxVisible = 12;
      const endValue = Math.min(total - 1, maxVisible - 1);
      return { startValue: 0, endValue };
    },
    getVisibleMainCount() {
      const total = this.scrollCategories.length;
      if (!total) return 0;

      const dz = this.getDataZoomById('dz_y');
      let startValue = dz && dz.startValue != null ? dz.startValue : null;
      let endValue = dz && dz.endValue != null ? dz.endValue : null;

      if (startValue == null || endValue == null) {
        const fallback = this.getYAxisScrollWindow();
        startValue = fallback.startValue;
        endValue = fallback.endValue;
      }

      const start = this.clamp(Math.round(startValue), 0, total - 1);
      const end = this.clamp(Math.round(endValue), 0, total - 1);
      return Math.max(1, end - start + 1);
    },
    getDataZoomById(id) {
      if (!this.chart || !id) return null;
      const option = this.chart.getOption();
      const dataZoom = option && option.dataZoom;
      if (!Array.isArray(dataZoom)) return null;
      return dataZoom.find((item) => item && item.id === id) || null;
    },
    scrollYAxis(event) {
      if (!this.chart || !event) return;
      const total = this.scrollCategories.length;
      if (!total) return;

      const delta =
        typeof event.wheelDelta === 'number'
          ? event.wheelDelta
          : typeof event.deltaY === 'number'
            ? -event.deltaY
            : 0;
      if (!delta) return;

      const direction = delta > 0 ? -1 : 1;
      const fallback = this.getYAxisScrollWindow();
      const dz = this.getDataZoomById('dz_y');
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

      this.chart.dispatchAction({
        type: 'dataZoom',
        dataZoomId: 'dz_y',
        startValue: nextStart,
        endValue: nextEnd
      });
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
    updateGridLayout() {
      if (!this.chart) return;
      const layout = this.getLayoutMetrics();
      const dataZoomLayout = this.getDataZoomLayout(layout);

      this.chart.setOption({
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
      });
    },
    onChartClick(params) {
      this.ignoreNextZrClick = true;
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
      if (this.ignoreNextZrClick) {
        this.ignoreNextZrClick = false;
        return;
      }
      this.clearActiveMarker();
    },
    onChartWheel(event) {
      this.clearActiveMarker();
      this.scrollYAxis(event);
    },
    onZrMouseDown(event) {
      if (this.activeMarkerId == null) return;
      if (!event) return;
      this.dragOrigin = { x: event.offsetX, y: event.offsetY };
    },
    onZrMouseMove(event) {
      if (!this.dragOrigin || this.activeMarkerId == null) return;
      if (!event) return;
      const dx = event.offsetX - this.dragOrigin.x;
      const dy = event.offsetY - this.dragOrigin.y;
      if (dx * dx + dy * dy >= 16) {
        this.dragOrigin = null;
        this.clearActiveMarker();
      }
    },
    onZrMouseUp() {
      this.dragOrigin = null;
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
    getActiveMarkers() {
      if (this.activeMarkerId == null) return [];
      return (this.markers || []).filter(
        (marker) =>
          marker &&
          marker.time != null &&
          marker.id === this.activeMarkerId
      );
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
    buildOption() {
      const layout = this.getLayoutMetrics();
      const cursorRange = this.getClampedCursorRange();

      return {
        backgroundColor: '#ffffff',
        animation: false,
        tooltip: {
          show: false,
          formatter: (params) => {
            if (!params || !params.value) return '';
            const start = this.formatDateTime(params.value[1]);
            const end = this.formatDateTime(params.value[2]);
            return [
              `${params.marker}${params.name}`,
              `${start} → ${end}`,
              `${Math.round(params.value[3] / 1000)}s`
            ].join('<br/>');
          }
        },
        dataZoom: this.buildDataZoomOptions(layout, cursorRange),
        grid: this.buildGridOptions(layout),
        xAxis: this.buildXAxisOptions(cursorRange),
        yAxis: this.buildYAxisOptions(),
        series: this.buildSeriesOptions(cursorRange)
      };
    },
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
        this.buildMarkerLabelSeries(),
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
    buildMarkerLabelSeries() {
      const { markerLabel, lineStyle, index } = CHART_CONFIG;
      return {
        id: 'sy-markers-labels',
        type: 'scatter',
        xAxisIndex: index.axis.label,
        yAxisIndex: index.axis.label,
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
            color: markerLabel.color,
            backgroundColor: markerLabel.backgroundColor,
            borderColor: markerLabel.borderColor,
            borderWidth: markerLabel.borderWidth,
            padding: markerLabel.padding,
            borderRadius: 6,
            fontSize: markerLabel.fontSize,
            fontFamily: markerLabel.fontFamily,
            formatter: (params) => {
              const x = params && params.data && params.data.xAxis;
              return this.formatDateTime(x);
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
    renderItem(params, api) {
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
    },
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
    toTimeValue(value) {
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
    },
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
    getGridRect(gridIndex = CHART_CONFIG.index.grid.main) {
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
    getAxisPixelFromValue(value, xAxisIndex = CHART_CONFIG.index.axis.main) {
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

      const { index, markerDelete } = CHART_CONFIG;
      const markers = this.getActiveMarkers();
      const mainGridRect = this.getGridRect(index.grid.main);
      const pinnedGridRect = this.getGridRect(index.grid.pinned);
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
