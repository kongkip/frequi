<template>
  <div class="row flex-grow-1 chart-wrapper">
    <v-chart v-if="hasData" theme="dark" autoresize :options="chartOptions" />
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';
import ECharts from 'vue-echarts';
import * as echarts from 'echarts';
import { Trade, PairHistory, PlotConfig } from '@/types';
import randomColor from '@/shared/randomColor';
import { roundTimeframe } from '@/shared/timemath';

import 'echarts';

// Chart default options
const MARGINLEFT = '4%';
const MARGINRIGHT = '1%';
// const upColor = '#00da3c';
// const downColor = '#ec0000';
// const upBorderColor = '#008F28';
// const downBorderColor = '#8A0000';

// Binance colors
const upColor = '#2ed191';
const upBorderColor = '#19d189';
const downColor = '#f84960';
const downBorderColor = '#e33249';

@Component({
  components: { 'v-chart': ECharts },
})
export default class CandleChart extends Vue {
  @Prop({ required: false, default: [] }) readonly trades!: Array<Trade>;

  @Prop({ required: true }) readonly dataset!: PairHistory;

  @Prop({ default: true }) readonly useUTC!: boolean;

  @Prop({ required: true }) plotConfig!: PlotConfig;

  // Only recalculate buy / sell data if necessary
  signalsCalculated = false;

  buyData = [] as Array<number>[];

  sellData = [] as Array<number>[];

  @Watch('timeframe')
  timeframeChanged() {
    this.signalsCalculated = false;
  }

  @Watch('dataset')
  datasetChanged() {
    this.signalsCalculated = false;
  }

  get strategy() {
    return this.dataset ? this.dataset.strategy : '';
  }

  get pair() {
    return this.dataset ? this.dataset.pair : '';
  }

  get timeframe() {
    return this.dataset ? this.dataset.timeframe : '';
  }

  get timeframems() {
    return this.dataset ? this.dataset.timeframe_ms : 0;
  }

  get datasetColumns() {
    return this.dataset ? this.dataset.columns : [];
  }

  get hasData() {
    return this.dataset !== null && typeof this.dataset === 'object';
  }

  get filteredTrades() {
    return this.trades.filter((item: Trade) => item.pair === this.pair);
  }

  get chartOptions() {
    if (!this.hasData) {
      return {};
    }

    // console.log(`Available Columns: ${this.dataset.columns}`);
    // Find default columns (sequence might be different, depending on the strategy)
    const colDate = this.dataset.columns.findIndex((el) => el === '__date_ts');
    const colOpen = this.dataset.columns.findIndex((el) => el === 'open');
    const colHigh = this.dataset.columns.findIndex((el) => el === 'high');
    const colLow = this.dataset.columns.findIndex((el) => el === 'low');
    const colClose = this.dataset.columns.findIndex((el) => el === 'close');
    const colVolume = this.dataset.columns.findIndex((el) => el === 'volume');
    const colBuyData = this.dataset.columns.findIndex((el) => el === '_buy_signal_open');
    const colSellData = this.dataset.columns.findIndex((el) => el === '_sell_signal_open');

    const subplotCount =
      'subplots' in this.plotConfig ? Object.keys(this.plotConfig.subplots).length : 0;

    // console.log(`subplotcount: ${subplotCount}`);

    // Always show ~250 candles max as starting point
    const startingZoom = (1 - 250 / this.dataset.length) * 100;

    const options: echarts.EChartOption = {
      title: {
        text: `${this.strategy} - ${this.pair} - ${this.timeframe}`,
        show: true,
      },
      backgroundColor: '#1b1b1b',
      useUTC: this.useUTC,
      dataset: {
        source: this.dataset.data,
      },
      animation: false,
      legend: {
        data: ['Candles', 'Volume', 'Buy', 'Sell'],
        right: '1%',
      },
      tooltip: {
        show: true,
        trigger: 'axis',
        axisPointer: {
          type: 'cross',
          lineStyle: {
            color: '#cccccc',
            width: 1,
            opacity: 1,
          },
        },
      },
      axisPointer: {
        link: [{ xAxisIndex: 'all' }],
        label: {
          backgroundColor: '#777',
        },
      },
      xAxis: [
        {
          type: 'time',
          scale: true,
          boundaryGap: false,
          axisLine: { onZero: false },
          axisTick: { show: true },
          axisLabel: { show: true },
          position: 'top',
          splitLine: { show: false },
          splitNumber: 20,
          min: 'dataMin',
          max: 'dataMax',
        },
        {
          type: 'time',
          gridIndex: 1,
          scale: true,
          boundaryGap: false,
          axisLine: { onZero: false },
          axisTick: { show: false },
          axisLabel: { show: false },
          splitLine: { show: false },
          splitNumber: 20,
          min: 'dataMin',
          max: 'dataMax',
        },
      ],
      yAxis: [
        {
          scale: true,
        },
        {
          scale: true,
          gridIndex: 1,
          splitNumber: 2,
          axisLabel: { show: false },
          axisLine: { show: false },
          axisTick: { show: false },
          splitLine: { show: false },
        },
      ],
      grid: [
        {
          left: MARGINLEFT,
          right: MARGINRIGHT,
          // Grid Layout from bottom to top
          bottom: `${subplotCount * 10 + 10}%`,
        },
        {
          // Volume
          left: MARGINLEFT,
          right: MARGINRIGHT,
          // Grid Layout from bottom to top
          bottom: `${subplotCount * 10 + 5}%`,
          height: '10%',
        },
      ],
      dataZoom: [
        {
          type: 'inside',
          xAxisIndex: [0, 1],
          start: startingZoom,
          end: 100,
        },
        {
          show: true,
          xAxisIndex: [0, 1],
          type: 'slider',
          bottom: 10,
          start: startingZoom,
          end: 100,
        },
      ],
      // visualMap: {
      //   //  TODO: this would allow to colorize volume bars (if we'd want this)
      //   //  Needs green / red indicator column in data.
      //   show: true,
      //   seriesIndex: 1,
      //   dimension: 5,
      //   pieces: [
      //     {
      //       max: 500000.0,
      //       color: downColor,
      //     },
      //     {
      //       min: 500000.0,
      //       color: upColor,
      //     },
      //   ],
      // },
      series: [
        {
          name: 'Candles',
          type: 'candlestick',
          barWidth: '80%',
          itemStyle: {
            color: upColor,
            color0: downColor,
            borderColor: upBorderColor,
            borderColor0: downBorderColor,
          },
          encode: {
            x: colDate,
            // open, close, low, high
            y: [colOpen, colClose, colLow, colHigh],
          },
        },
        {
          name: 'Volume',
          type: 'bar',
          xAxisIndex: 1,
          yAxisIndex: 1,
          itemStyle: {
            color: '#777777',
          },
          large: true,
          encode: {
            x: colDate,
            y: colVolume,
          },
        },
        {
          name: 'Buy',
          type: 'scatter',
          symbol: 'triangle',
          symbolSize: 10,
          xAxisIndex: 0,
          yAxisIndex: 0,
          itemStyle: {
            color: '#00ff26',
          },
          encode: {
            x: colDate,
            y: colBuyData,
          },
        },
        {
          name: 'Sell',
          type: 'scatter',
          symbol: 'diamond',
          symbolSize: 8,
          xAxisIndex: 0,
          yAxisIndex: 0,
          itemStyle: {
            color: '#FF0000',
          },
          encode: {
            x: colDate,
            y: colSellData,
          },
        },
      ],
    };

    // this.createSignalData(colDate, colOpen, colBuy, colSell);

    // This will be merged into final plot config
    // const subPlots = {
    //   legend: [] as string[],
    //   grid: [] as object[],
    //   yaxis: [] as object[],
    //   xaxis: [] as object[],
    //   xaxisIndex: [] as number[],
    //   series: [] as object[],
    // };

    if ('main_plot' in this.plotConfig) {
      Object.entries(this.plotConfig.main_plot).forEach(([key, value]) => {
        const col = this.dataset.columns.findIndex((el) => el === key);
        if (col > 1) {
          if (options.legend && options.legend.data) {
            options.legend.data.push(key);
          }
          const sp: echarts.EChartOption.Series = {
            name: key,
            type: value.type || 'line',
            xAxisIndex: 0,
            yAxisIndex: 0,
            itemStyle: {
              color: value.color,
            },
            encode: {
              x: colDate,
              y: col,
            },
            showSymbol: false,
          };
          if (options.series) {
            options.series.push(sp);
          }
        } else {
          console.log(`element ${key} for main plot not found in columns.`);
        }
      });
    }

    // START Subplots
    if ('subplots' in this.plotConfig) {
      let plotIndex = 2;
      Object.entries(this.plotConfig.subplots).forEach(([key, value]) => {
        // define yaxis
        if (options.yAxis && Array.isArray(options.yAxis)) {
          options.yAxis.push({
            scale: true,
            gridIndex: plotIndex,
            name: key,
            nameLocation: 'middle',
            nameGap: 60,
            axisLabel: { show: true },
            axisLine: { show: false },
            axisTick: { show: false },
            splitLine: { show: false },
          });
        }
        if (options.xAxis && Array.isArray(options.xAxis)) {
          options.xAxis.push({
            type: 'time',
            scale: true,
            gridIndex: plotIndex,
            boundaryGap: false,
            axisLine: { onZero: false },
            axisTick: { show: false },
            axisLabel: { show: false },
            splitLine: { show: false },
            splitNumber: 20,
          });
        }
        if (options.dataZoom) {
          options.dataZoom.forEach((el) =>
            el.xAxisIndex && Array.isArray(el.xAxisIndex) ? el.xAxisIndex.push(plotIndex) : null,
          );
        }
        if (options.grid && Array.isArray(options.grid)) {
          options.grid.push({
            left: MARGINLEFT,
            right: MARGINRIGHT,
            bottom: `${plotIndex * 8}%`,
            height: '8%',
          });
        }
        Object.entries(value).forEach(([sk, sv]) => {
          if (options.legend && options.legend.data && Array.isArray(options.legend.data)) {
            options.legend.data.push(sk);
          }
          // entries per subplot
          const col = this.dataset.columns.findIndex((el) => el === sk);
          if (col > 0) {
            const sp: echarts.EChartOption.Series = {
              name: sk,
              type: sv.type || 'line',
              xAxisIndex: plotIndex,
              yAxisIndex: plotIndex,
              itemStyle: {
                color: sv.color || randomColor(),
              },
              encode: {
                x: colDate,
                y: col,
              },
              showSymbol: false,
            };
            if (options.series && Array.isArray(options.series)) {
              options.series.push(sp);
            }
          } else {
            console.log(`element ${sk} was not found in the columns.`);
          }
        });

        plotIndex += 1;
      });
    }
    // END Subplots
    // Last subplot should show xAxis labels
    // if (options.xAxis && Array.isArray(options.xAxis)) {
    //   options.xAxis[options.xAxis.length - 1].axisLabel.show = true;
    //   options.xAxis[options.xAxis.length - 1].axisTick.show = true;
    // }
    if (options.grid && Array.isArray(options.grid)) {
      // Last subplot is bottom
      options.grid[options.grid.length - 1].bottom = '50px';
      delete options.grid[options.grid.length - 1].top;
    }

    if (this.filteredTrades.length > 0) {
      // Show trades
      const trades: Array<string | number>[] = [];
      const tradesClose: Array<string | number>[] = [];
      for (let i = 0, len = this.filteredTrades.length; i < len; i += 1) {
        const trade: Trade = this.filteredTrades[i];
        if (
          trade.open_timestamp >= this.dataset.data_start_ts &&
          trade.open_timestamp <= this.dataset.data_stop_ts
        ) {
          trades.push([roundTimeframe(this.timeframems, trade.open_timestamp), trade.open_rate]);
        }
        if (
          trade.close_timestamp !== undefined &&
          trade.close_timestamp < this.dataset.data_stop_ts &&
          trade.close_timestamp > this.dataset.data_start_ts
        ) {
          if (trade.close_date !== undefined && trade.close_rate !== undefined) {
            tradesClose.push([
              roundTimeframe(this.timeframems, trade.close_timestamp),
              trade.close_rate,
            ]);
          }
        }
      }
      // console.log(`Trades: ${trades.length}`);
      // console.log(trades);
      // console.log(`ClosesTrades: ${tradesClose.length}`);
      // console.log(tradesClose);

      const name = 'Trades';
      const nameClose = 'Trades Close';
      if (options.legend && options.legend.data) {
        options.legend.data.push(name);
      }
      const sp: echarts.EChartOption.SeriesScatter = {
        name,
        type: 'scatter',
        xAxisIndex: 0,
        yAxisIndex: 0,
        itemStyle: {
          color: 'cyan',
        },
        data: trades,
      };
      if (options.series) {
        options.series.push(sp);
      }
      if (options.legend && options.legend.data) {
        options.legend.data.push(nameClose);
      }
      const closeSeries: echarts.EChartOption.SeriesScatter = {
        name: nameClose,
        type: 'scatter',
        xAxisIndex: 0,
        yAxisIndex: 0,
        itemStyle: {
          color: 'pink',
        },
        data: tradesClose,
      };
      if (options.series) {
        options.series.push(closeSeries);
      }
    }

    // console.log(options);
    // TODO: Rebuilding this causes a full redraw for every new step
    return options;
  }

  // createSignalData(colDate: number, colOpen: number, colBuy: number, colSell: number): void {
  // Calculate Buy and sell Series
  // if (!this.signalsCalculated) {
  //   // Generate Buy and sell array (using open rate to display marker)
  //   for (let i = 0, len = this.dataset.data.length; i < len; i += 1) {
  //     if (this.dataset.data[i][colBuy] === 1) {
  //       this.buyData.push([this.dataset.data[i][colDate], this.dataset.data[i][colOpen]]);
  //     }
  //     if (this.dataset.data[i][colSell] === 1) {
  //       this.sellData.push([this.dataset.data[i][colDate], this.dataset.data[i][colOpen]]);
  //     }
  //   }
  //   this.signalsCalculated = true;
  // }
  // }
}
</script>

<style scoped>
.chart-wrapper {
  width: 100%;
  height: 100%;
}
.echarts {
  width: 100%;
  min-height: 200px;
  /* TODO: height calculation is not working correctly - uses min-height for now */
  /* height: 600px; */
  height: 100%;
}
</style>
