<template>
  <div v-if="configShadow" class="shadow">
    <div class="config-box" :style="{ marginTop: top + 'px' }">
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
      <p class="mode-tip" v-if="mode == 'stock'">
        股票文本模式仅处理 stockListM。可直接粘贴 {"stockListM":[...]}，也可直接粘贴数组 [...]
      </p>
      <p class="mode-tip" v-if="mode == 'bank'">
        存款文本模式仅处理 bankList。可直接粘贴 {"bankList":[...]}，也可直接粘贴数组 [...]
      </p>
      <div class="tab-content" v-if="checked == 'export'">
        <el-input
          type="textarea"
          :rows="15"
          :placeholder="mode == 'stock' ? '此处显示股票 JSON 文本' : mode == 'bank' ? '此处显示存款 JSON 文本' : '请输入内容'"
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
          :placeholder="mode == 'stock' ? '请在此输入框粘贴股票 JSON 文本' : mode == 'bank' ? '请在此输入框粘贴存款 JSON 文本' : '请在此输入框粘贴配置文本'"
          v-model="inputConfigStr"
        >
        </el-input>
        <input
          class="btn success"
          type="button"
          :value="mode == 'stock' ? '提交股票文本' : mode == 'bank' ? '提交存款文本' : '提交配置文本'"
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
function flattenFundListGroup(config) {
  if (!config || !Array.isArray(config.fundListGroup)) {
    return [];
  }

  const merged = [];
  config.fundListGroup.forEach((group) => {
    if (!group || !Array.isArray(group.funds)) {
      return;
    }
    group.funds.forEach((fund) => {
      if (!fund || !fund.code) {
        return;
      }
      merged.push({
        code: fund.code,
        num: fund.num || 0,
        cost: fund.cost || 0,
      });
    });
  });

  return merged;
}

export default {
  components: {},
  name: "configBox",
  props: {
    top: {
      type: Number,
      default: 0,
    },
  },
  data() {
    return {
      configShadow: false,
      checked: "export",
      mode: "all",
      textarea: "",
      exportConfigStr: null,
      inputConfigStr: null,
    };
  },
  watch: {},
  mounted() {},
  methods: {
    init(mode = "all") {
      this.mode = mode;
      this.configShadow = true;
      this.checked = "export";
      this.inputConfigStr = null;
      chrome.storage.sync.get(null, (syncRes) => {
        delete syncRes.holiday;
        if (this.mode === "stock") {
          this.exportConfigStr = JSON.stringify({
            stockListM: syncRes.stockListM || [],
          });
          return;
        }
        if (this.mode === "bank") {
          this.exportConfigStr = JSON.stringify({
            bankList: syncRes.bankList || [],
          });
          return;
        }

        // 全部导出时，从 local 获取 fundListM 合并
        chrome.storage.local.get(["fundListM"], (localRes) => {
          if (localRes.fundListM && localRes.fundListM.length) {
            syncRes.fundListM = localRes.fundListM;
          }
          this.exportConfigStr = JSON.stringify(syncRes);
        });
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
            if (Array.isArray(config)) {
              config = { stockListM: config };
            } else if (!Array.isArray(config.stockListM)) {
              throw new Error("stock config format invalid");
            }
          }

          if (this.mode === "bank") {
            if (Array.isArray(config)) {
              config = { bankList: config };
            } else if (!Array.isArray(config.bankList)) {
              throw new Error("bank config format invalid");
            }
          }

          if ((!config.fundListM || !config.fundListM.length) && config.fundListGroup) {
            config.fundListM = flattenFundListGroup(config);
          }

          // fundListM 存储到 local（突破 sync 的 8KB 限制）
          let fundListM = null;
          if (config.fundListM) {
            fundListM = config.fundListM;
            delete config.fundListM;
          }

          const doImport = () => {
            chrome.storage.sync.set(config, (val) => {
              this.$emit("success", false);

              this.$message({
                message: this.mode === "stock" ? "恭喜,导入股票配置成功！" : this.mode === "bank" ? "恭喜,导入存款配置成功！" : "恭喜,导入配置成功！",
                type: "success",
                center: true,
              });
            });
          };

          if (fundListM) {
            chrome.storage.local.set({ fundListM: fundListM }, doImport);
          } else {
            doImport();
          }
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

.mode-tip {
  font-size: 12px;
  color: #999999;
  line-height: 1.5;
  padding: 0 15px 8px;
  text-align: left;
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
