<template>
  <div
    class="asset-pie-echarts"
    ref="pieCharts"
    :style="{ width: '100%', height: '280px' }"
  ></div>
</template>

<script>
let echarts = require("echarts/lib/echarts");
import "./js/customed.js";
import "./js/dark.js";

require("echarts/lib/chart/pie");
require("echarts/lib/component/tooltip");
require("echarts/lib/component/legend");

export default {
  name: "assetPie",
  props: {
    darkMode: {
      type: Boolean,
      default: false,
    },
    valData: {
      type: Array,
      default: () => [],
    },
  },
  data() {
    return {
      chartEL: null,
      myChart: null,
      option: {},
    };
  },
  watch: {
    darkMode(val) {
      if (this.myChart) {
        this.myChart.dispose();
      }
      this.init();
    },
    valData: {
      handler(newVal) {
        if (this.myChart) {
          this.option.series[0].data = newVal;
          this.option.legend.data = newVal.map((item) => item.name);
          this.myChart.setOption(this.option);
        }
      },
      deep: true,
    },
  },
  mounted() {
    this.init();
  },
  beforeDestroy() {
    if (this.myChart) {
      this.myChart.dispose();
    }
  },
  methods: {
    init() {
      this.chartEL = this.$refs.pieCharts;
      if (!this.chartEL) return;
      this.myChart = echarts.init(this.chartEL, this.darkMode ? "dark" : "customed");

      this.option = {
        tooltip: {
          trigger: "item",
          formatter: function (params) {
            return params.name + ' : ' + params.value.toLocaleString() + ' (' + params.percent.toFixed(1) + '%)';
          },
          backgroundColor: this.darkMode ? 'rgba(50,50,50,0.7)' : 'rgba(255,255,255,0.9)',
          textStyle: {
            color: this.darkMode ? '#fff' : '#000'
          }
        },
        legend: {
          orient: "vertical",
          left: "right",
          top: "middle",
          data: this.valData.map((item) => item.name),
          textStyle: {
            color: this.darkMode ? "rgba(255,255,255,0.6)" : "#000",
          },
        },
        series: [
          {
            name: "资产占比",
            type: "pie",
            radius: ["40%", "70%"],
            center: ["40%", "50%"],
            data: this.valData,
            label: {
              show: true,
              formatter: function (params) {
                return params.name + ': ' + params.percent.toFixed(1) + '%';
              },
              color: this.darkMode ? 'rgba(255,255,255,0.6)' : '#000'
            },
            emphasis: {
              itemStyle: {
                shadowBlur: 10,
                shadowOffsetX: 0,
                shadowColor: "rgba(0, 0, 0, 0.5)",
              },
            },
          },
        ],
      };
      this.myChart.setOption(this.option);
    },
  },
};
</script>

<style lang="scss" scoped>
.asset-pie-echarts {
  margin-top: 10px;
}
</style>
