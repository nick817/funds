<template>
  <div v-if="configShadow" class="shadow">
    <div class="config-box" :style="{ marginTop: top + 'px' }">
      <div class="box-title">{{ dialogTitle }}</div>
      <div class="tab-row">
        <button
          @click="checked = 'export'"
          :class="checked == 'export' ? 'checked' : ''"
        >
          导出(JSON文本)
        </button>
        <button
          @click="checked = 'import'"
          :class="checked == 'import' ? 'checked' : ''"
        >
          导入(JSON文本)
        </button>
      </div>
      <div class="tab-content" v-if="checked == 'export'">
        <el-input
          type="textarea"
          :rows="15"
          :placeholder="exportPlaceholder"
          v-model="exportConfigStr"
        >
        </el-input>
        <input
          class="btn success"
          type="button"
          value="复制到剪贴板"
          v-clipboard:copy="exportConfigStr"
          v-clipboard:success="copy"
          v-clipboard:error="onError"
        />
      </div>
      <div class="tab-content" v-else>
        <el-input
          type="textarea"
          :rows="15"
          :placeholder="importPlaceholder"
          v-model="inputConfigStr"
        >
        </el-input>
        <input
          class="btn success"
          type="button"
          :value="submitButtonText"
          @click="importInput"
        />
      </div>

      <div class="tab-row">
        <input class="btn" type="button" value="返回" @click="close" />
      </div>
    </div>
  </div>
</template>

<script>
export default {
  components: {},
  name: "configBox",
  props: {
    top: {
      type: Number,
      default: 0,
    },
    darkMode: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      configShadow: false,
      checked: "export",
      mode: "config",
      textarea: "",
      exportConfigStr: null,
      inputConfigStr: null,
    };
  },
  watch: {},
  mounted() {},
  computed: {
    dialogTitle() {
      return this.mode === "stock" ? "股票导入导出文本" : "配置导入导出文本";
    },
    exportPlaceholder() {
      return this.mode === "stock" ? "此处会生成股票列表 JSON 文本" : "此处会生成完整配置 JSON 文本";
    },
    importPlaceholder() {
      return this.mode === "stock"
        ? "请在此输入框粘贴股票列表 JSON 文本"
        : "请在此输入框粘贴配置文本";
    },
    submitButtonText() {
      return this.mode === "stock" ? "提交股票文本" : "提交配置文本";
    },
  },
  methods: {
    normalizeStockItem(item) {
      const source = typeof item === "object" && item !== null ? item : { code: item };
      const normalizedCode = this.resolveStockSecid(
        source.secid || source.code || source.stockCode || source.symbol,
        source.market || source.exchange || source.mkt
      );

      if (!normalizedCode) {
        return null;
      }

      return {
        code: normalizedCode.code,
        secid: normalizedCode.secid,
        market: normalizedCode.market,
        name: source.name || source.stockName || "",
        num: this.normalizeNumber(source.num ?? source.quantity ?? source.shares),
        cost: this.normalizeNumber(
          source.cost ?? source.costPrice ?? source.avgCost ?? source.price
        ),
      };
    },
    normalizeNumber(value) {
      if (value === null || value === undefined || value === "") {
        return 0;
      }

      const numberValue = Number(value);
      return Number.isNaN(numberValue) ? 0 : numberValue;
    },
    resolveStockSecid(rawCode, rawMarket) {
      if (rawCode === null || rawCode === undefined || rawCode === "") {
        return null;
      }

      const codeText = String(rawCode).trim().toUpperCase();
      if (/^[01]\.\d{6}$/.test(codeText)) {
        const [market, code] = codeText.split(".");
        return { secid: codeText, code, market };
      }

      const prefixMatched = codeText.match(/^(SH|SZ)(\d{6})$/);
      if (prefixMatched) {
        const market = prefixMatched[1] === "SH" ? "1" : "0";
        return { secid: `${market}.${prefixMatched[2]}`, code: prefixMatched[2], market };
      }

      const suffixMatched = codeText.match(/^(\d{6})\.(SH|SZ)$/);
      if (suffixMatched) {
        const market = suffixMatched[2] === "SH" ? "1" : "0";
        return { secid: `${market}.${suffixMatched[1]}`, code: suffixMatched[1], market };
      }

      const digitMatched = codeText.match(/\d{6}/);
      if (!digitMatched) {
        return null;
      }

      const code = digitMatched[0];
      const marketText = rawMarket === null || rawMarket === undefined ? "" : String(rawMarket).trim().toLowerCase();
      let market = null;
      if (["1", "sh", "sha", "shanghai", "沪", "沪市"].includes(marketText)) {
        market = "1";
      } else if (["0", "sz", "sza", "shenzhen", "深", "深市"].includes(marketText)) {
        market = "0";
      } else if (/^(5|6|9|11|13)/.test(code)) {
        market = "1";
      } else {
        market = "0";
      }

      return { secid: `${market}.${code}`, code, market };
    },
    init(options = {}) {
      this.configShadow = true;
      this.checked = "export";
      this.mode = options.mode || "config";
      this.inputConfigStr = null;
      chrome.storage.sync.get(null, (res) => {
        if (this.mode === "stock") {
          this.exportConfigStr = JSON.stringify(res.stockListM || [], null, 2);
          return;
        }

        delete res.holiday;
        this.exportConfigStr = JSON.stringify(res, null, 2);
      });
    },
    close() {
      this.configShadow = false;
      this.$emit("close", false);
    },
    exportConfig() {},
    copy(e) {
      this.$message({
        message: "已复制到剪贴板，请自行保存！",
        type: "success",
        center: true,
      });
    },
    onError(e) {},
    importInput() {
      try {
        if (typeof JSON.parse(this.inputConfigStr) == "object") {
          let config = JSON.parse(this.inputConfigStr);

          if (this.mode === "stock") {
            const rawList = Array.isArray(config)
              ? config
              : config.stockListM || config.stocks || config.stockList;
            if (!Array.isArray(rawList)) {
              throw new Error("Invalid stock list");
            }

            const stockListM = rawList
              .map((item) => this.normalizeStockItem(item))
              .filter(Boolean);

            if (!stockListM.length) {
              throw new Error("Empty stock list");
            }

            chrome.storage.sync.set({ stockListM }, () => {
              this.$emit("success", false);
              this.$message({
                message: "恭喜,导入股票列表成功！",
                type: "success",
                center: true,
              });
            });

            return;
          }

          chrome.storage.sync.set(config, (val) => {
            this.$emit("success", false);

            this.$message({
              message: "恭喜,导入配置成功！",
              type: "success",
              center: true,
            });
          });
        }
      } catch (e) {
        this.$message({
          message: "导入失败，配置文本格式不正确！",
          type: "error",
          center: true,
        });
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.shadow {
  position: fixed;
  width: 100%;
  height: 100%;
  padding: 20px 40px;
  box-sizing: border-box;
  top: 0;
  left: 0;
  background-color: rgba(0, 0, 0, 0.7);
}

.config-box {
  background: #ffffff;
  border-radius: 15px;
  width: 500px;
  margin: 0 auto;
  text-align: center;
  line-height: 1;
  vertical-align: middle;
  font-size: 0;
  .tab-content {
    .btn {
      margin: 15px 0 5px;
    }
  }
}

.box-title {
  padding-top: 18px;
  font-size: 16px;
  font-weight: bold;
}

.config-box button {
  line-height: 1;
  white-space: nowrap;
  vertical-align: middle;
  background: #fff;
  border: 1px solid #dcdfe6;
  font-weight: 500;
  border-left: 0;
  color: #606266;
  -webkit-appearance: none;
  text-align: center;
  box-sizing: border-box;
  margin: 0;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.645, 0.045, 0.355, 1);
  padding: 12px 20px;
  font-size: 14px;
  width: 150px;
  position: relative;
  display: inline-block;
  outline: none;
}

.config-box button:first-child {
  border-left: 1px solid #dcdfe6;
  border-radius: 4px 0 0 4px;
  box-shadow: none !important;
}

.config-box button:last-child {
  border-radius: 0 4px 4px 0;
}

.config-box button.checked {
  color: #fff;
  background-color: #409eff;
  border-color: #409eff;
}

.tab-row {
  padding: 12px 0;
}

.tab-row:after,
.tab-row:before {
  display: table;
  content: "";
}

.tab-row:after {
  clear: both;
}

.tips {
  font-size: 12px;
  margin: 0;
  color: #aaaaaa;
  line-height: 1.4;
  padding: 5px 15px;
}

.reward-tips {
  padding: 0 50px;
  margin-top: 5px;
}

.btn {
  display: inline-block;
  line-height: 1;
  cursor: pointer;
  background: #fff;
  padding: 5px 6px;
  border-radius: 3px;
  font-size: 12px;
  color: #000000;
  margin: 0 5px;
  outline: none;
  border: 1px solid #dcdfe6;
}
.success {
  color: #4eb61b;
  border-color: #4eb61b;
}

.darkMode .config-box {
  color: rgba($color: #ffffff, $alpha: 0.6);
  background-color: #373737;
  .btn {
    background-color: rgba($color: #ffffff, $alpha: 0.16);
    color: rgba($color: #ffffff, $alpha: 0.6);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.6);
  }
  .success {
    border: 1px solid rgba($color: #4eb61b, $alpha: 0.6);
    background-color: rgba($color: #4eb61b, $alpha: 0.6);
  }

  button {
    background-color: rgba($color: #ffffff, $alpha: 0.16);
    color: rgba($color: #ffffff, $alpha: 0.6);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.38);
  }

  button.checked {
    color: rgba($color: #ffffff, $alpha: 0.6);
    border: 1px solid rgba($color: #409eff, $alpha: 0.38);
    background-color: rgba($color: #409eff, $alpha: 0.6);
  }
}
</style>
