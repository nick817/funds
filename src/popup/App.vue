<template>
  <div
    id="app"
    class="container"
    ref="app"
    :class="containerClass"
    :style="[zoom, grayscale, opacity]"
  >
    <div>
      <div v-if="isEdit" class="input-row">
        <span>添加新基金:</span>
        <el-select
          v-model="fundcode"
          multiple
          filterable
          :popper-append-to-body="false"
          remote
          size="mini"
          reserve-keyword
          @visible-change="selectChange"
          placeholder="请输入基金编码，支持按名称或编码搜索"
          :remote-method="remoteMethod"
          :loading="loading"
          style="width:300px"
        >
          <el-option
            v-for="item in searchOptions"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          >
            <span style="float: left">{{ item.label }}</span>
            <span
              style="float: right; color: #8492a6; font-size: 13px;margim-right:20px;padding-right:15px"
              >{{ item.value }}</span
            >
          </el-option>
        </el-select>
        <input @click="save" class="btn" type="button" value="确定" />
      </div>
      <p v-if="isEdit" class="tips center">
        部分新发基金或QDII基金可以搜索到，但可能无法获取估值情况
      </p>
      <div v-if="isGetStorage" class="holdings-tabs-wrap">
        <el-tabs v-model="activeHoldingsTab" class="page-tabs" stretch>
          <el-tab-pane label="首页" name="home">
            <div class="home-pane">
              <div
                class="tab-row market-overview"
                v-loading="loadingInd"
                :element-loading-background="
                  darkMode ? 'rgba(0, 0, 0, 0.9)' : 'rgba(255, 255, 255, 0.9)'
                "
                :style="seciList.length > 0 ? { minHeight: '55px' } : {}"
              >
                <div
                  v-for="(el, index) in indFundData"
                  :key="el.f12"
                  :draggable="isEdit"
                  class="tab-col indFund"
                  :class="drag"
                  @click.stop="!isEdit && indDetail(el)"
                  @dragstart="handleDragStart($event, el)"
                  @dragover.prevent="handleDragOver($event, el)"
                  @dragenter="handleDragEnter($event, el, index)"
                  @dragend="handleDragEnd($event, el)"
                >
                  <h5>
                    {{ el.f14 }}
                    <span
                      v-if="isEdit"
                      @click="dltIndFund(index)"
                      class="dltBtn edit red btn"
                      >✖</span
                    >
                  </h5>
                  <p :class="el.f3 >= 0 ? 'up' : 'down'">
                    {{ el.f2
                    }}<input
                      v-if="isEdit && BadgeContent == 3"
                      @click="sltInd(el)"
                      :class="el.f13 + '.' + el.f12 == RealtimeIndcode ? 'slt' : ''"
                      class="btn edit"
                      style="margin-left:5px"
                      value="✔"
                      type="button"
                    />
                  </p>
                  <p :class="el.f3 >= 0 ? 'up' : 'down'">
                    {{ el.f4 }}&nbsp;&nbsp;{{ el.f3 }}%
                  </p>
                </div>
                <div v-if="isEdit && indFundData.length < 4" class="tab-col">
                  <div
                    v-if="!showAddSeciInput"
                    class="addSeci"
                    @click="() => (showAddSeciInput = true)"
                  >
                    添加
                  </div>
                  <div v-else>
                    <div style="padding-top:2px">
                      <el-select
                        size="mini"
                        :popper-append-to-body="false"
                        v-model="sltSeci"
                        style="width:110px"
                        placeholder="请选择"
                      >
                        <el-option
                          v-for="item in userSeciList"
                          :key="item.value"
                          :label="item.label"
                          :value="item.value"
                        ></el-option>
                      </el-select>
                    </div>
                    <div style="margin-top:4px">
                      <input
                        class="btn"
                        type="button"
                        value="取消"
                        @click="() => (showAddSeciInput = false)"
                      />
                      <input class="btn" type="button" value="确定" @click="saveSeci" />
                    </div>
                  </div>
                </div>
              </div>
              <div class="summary-panel summary-dashboard" v-if="showCost || showGains || showAmount">
                <div class="summary-row summary-row-home">
                  <div v-if="showAmount && dataList.length" class="summary-card summary-card-soft summary-card-red">
                    <span class="summary-label">基金持仓</span>
                    <strong class="summary-value">{{ parseFloat(fundAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  </div>
                  <div v-if="showGains && dataList.length" class="summary-card summary-card-green">
                    <span class="summary-label">基金估算</span>
                    <strong class="summary-value">{{ parseFloat(fundGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                    <span class="summary-meta">{{ isNaN(fundGains[1]) ? '' : '（' + fundGains[1] + '%）' }}</span>
                  </div>
                  <div v-if="showCost && dataList.length" class="summary-card summary-card-red">
                    <span class="summary-label">基金持有</span>
                    <strong class="summary-value">{{ parseFloat(fundCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                    <span class="summary-meta">{{ isNaN(fundCostGains[1]) ? '' : '（' + fundCostGains[1] + '%）' }}</span>
                  </div>
                  <div v-if="showAmount && stockDataList.length" class="summary-card summary-card-soft summary-card-red">
                    <span class="summary-label">股票持仓</span>
                    <strong class="summary-value">{{ parseFloat(stockAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  </div>
                  <div v-if="showGains && stockDataList.length" class="summary-card summary-card-green">
                    <span class="summary-label">股票当日</span>
                    <strong class="summary-value">{{ parseFloat(stockGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                    <span class="summary-meta">{{ isNaN(stockGains[1]) ? '' : '（' + stockGains[1] + '%）' }}</span>
                  </div>
                  <div v-if="showCost && stockDataList.length" class="summary-card summary-card-red">
                    <span class="summary-label">股票持有</span>
                    <strong class="summary-value">{{ parseFloat(stockCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                    <span class="summary-meta">{{ isNaN(stockCostGains[1]) ? '' : '（' + stockGains[1] + '%）' }}</span>
                  </div>
                  <div v-if="showAmount" class="summary-card summary-card-soft summary-card-red">
                    <span class="summary-label">持仓总额</span>
                    <strong class="summary-value">{{ parseFloat(allAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  </div>
                  <div v-if="showGains" class="summary-card summary-card-green">
                    <span class="summary-label">估算金额</span>
                    <strong class="summary-value">{{ parseFloat(allGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                    <span class="summary-meta">{{ isNaN(allGains[1]) ? '' : '（' + allGains[1] + '%）' }}</span>
                  </div>
                  <div v-if="showCost" class="summary-card summary-card-red">
                    <span class="summary-label">持有收益</span>
                    <strong class="summary-value">{{ parseFloat(allCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                    <span class="summary-meta">{{ isNaN(allCostGains[1]) ? '' : '（' + allCostGains[1] + '%）' }}</span>
                  </div>
                </div>
              </div>
            </div>
          </el-tab-pane>
          <el-tab-pane :label="`基金（${dataList.length}）`" name="fund">
            <div class="summary-panel tab-summary-panel" v-if="showCost || showGains || showAmount">
              <div class="summary-row tab-summary-row">
                <div v-if="showAmount" class="summary-card summary-card-soft summary-card-red">
                  <span class="summary-label">基金持仓</span>
                  <strong class="summary-value">{{ parseFloat(fundAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" class="summary-card summary-card-green">
                  <span class="summary-label">基金估算</span>
                  <strong class="summary-value">{{ parseFloat(fundGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(fundGains[1]) ? '' : '（' + fundGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" class="summary-card summary-card-red">
                  <span class="summary-label">基金持有</span>
                  <strong class="summary-value">{{ parseFloat(fundCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(fundCostGains[1]) ? '' : '（' + fundCostGains[1] + '%）' }}</span>
                </div>
              </div>
            </div>
            <div
              v-loading="loadingList"
              :element-loading-background="
                darkMode ? 'rgba(0, 0, 0, 0.9)' : 'rgba(255, 255, 255, 0.9)'
              "
              class="table-row"
              style="min-height:160px"
            >
              <table :class="tableHeight">
                <thead>
                  <tr>
                    <th class="align-left">基金名称（{{ dataList.length }}）</th>
                    <th v-if="isEdit">基金代码</th>
                    <th v-if="showGSZ && !isEdit">估算净值</th>
                    <th
                      style="text-align:center"
                      v-if="isEdit && (showCostRate || showCost)"
                    >
                      成本价
                    </th>
                    <th @click="sortList('amount')" v-if="showAmount" class="pointer">
                      持有额
                      <span :class="sortType.amount" class="down-arrow"></span>
                    </th>
                    <th
                      @click="sortList('costGains')"
                      v-if="showCost"
                      class="pointer"
                    >
                      持有收益
                      <span :class="sortType.costGains" class="down-arrow"></span>
                    </th>
                    <th
                      @click="sortList('costGainsRate')"
                      v-if="showCostRate"
                      class="pointer"
                    >
                      持有收益率
                      <span :class="sortType.costGainsRate" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('gszzl')" class="pointer">
                      涨跌幅
                      <span :class="sortType.gszzl" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('gains')" v-if="showGains" class="pointer">
                      估算收益
                      <span :class="sortType.gains" class="down-arrow"></span>
                    </th>
                    <th v-if="!isEdit">更新时间</th>
                    <th
                      style="text-align:center"
                      v-if="
                        isEdit &&
                          (showAmount || showGains || showCost || showCostRate)
                      "
                    >
                      持有份额
                    </th>
                    <th v-if="isEdit && BadgeContent == 1">特别关注</th>
                    <th v-if="isEdit">删除</th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="(el, index) in dataList"
                    :key="el.fundcode"
                    :draggable="isEdit"
                    :class="drag"
                    @dragstart="handleDragStart($event, el)"
                    @dragover.prevent="handleDragOver($event, el)"
                    @dragenter="handleDragEnter($event, el, index)"
                    @dragend="handleDragEnd($event, el)"
                  >
                    <td
                      :class="
                        isEdit ? 'fundName-noclick align-left' : 'fundName align-left'
                      "
                      :title="el.name"
                      @click.stop="!isEdit && fundDetail(el)"
                    >
                      <span class="hasReplace-tip" v-if="el.hasReplace">✔</span>{{ el.name }}
                    </td>
                    <td v-if="isEdit">{{ el.fundcode }}</td>
                    <td v-if="showGSZ && !isEdit">{{ el.gsz }}</td>
                    <td v-if="isEdit && (showCostRate || showCost)">
                      <input
                        class="btn num"
                        placeholder="持仓成本价"
                        v-model="el.cost"
                        @input="changeCost(el, index)"
                        type="text"
                      />
                    </td>

                    <td v-if="showAmount">
                      {{
                        parseFloat(el.amount).toLocaleString("zh", {
                          minimumFractionDigits: 2,
                        })
                      }}
                    </td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">
                      {{
                        parseFloat(el.costGains).toLocaleString("zh", {
                          minimumFractionDigits: 2,
                        })
                      }}
                    </td>
                    <td
                      v-if="showCostRate"
                      :class="el.costGainsRate >= 0 ? 'up' : 'down'"
                    >
                      {{ el.cost > 0 ? el.costGainsRate + "%" : "" }}
                    </td>
                    <td :class="el.gszzl >= 0 ? 'up' : 'down'">{{ el.gszzl }}%</td>
                    <td v-if="showGains" :class="el.gains >= 0 ? 'up' : 'down'">
                      {{
                        parseFloat(el.gains).toLocaleString("zh", {
                          minimumFractionDigits: 2,
                        })
                      }}
                    </td>
                    <td v-if="!isEdit">{{ formatUpdateTime(el) }}</td>
                    <th
                      style="text-align:center"
                      v-if="
                        isEdit &&
                          (showAmount || showGains || showCost || showCostRate)
                      "
                    >
                      <input
                        class="btn num"
                        placeholder="输入持有份额"
                        v-model="el.num"
                        @input="changeNum(el, index)"
                        type="text"
                      />
                    </th>
                    <td v-if="isEdit && BadgeContent == 1">
                      <input
                        @click="slt(el.fundcode)"
                        :class="el.fundcode == RealtimeFundcode ? 'slt' : ''"
                        class="btn edit"
                        value="✔"
                        type="button"
                      />
                    </td>
                    <td v-if="isEdit">
                      <input
                        @click="dlt(el.fundcode)"
                        class="btn red edit"
                        value="✖"
                        type="button"
                      />
                    </td>
                  </tr>
                  <tr v-if="!dataList.length">
                    <td class="empty-row" :colspan="fundTableColspan">暂无基金持仓</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </el-tab-pane>
          <el-tab-pane :label="`财务自由（${freeDataList.length}）`" name="free">
            <div class="summary-panel tab-summary-panel" v-if="showCost || showGains || showAmount">
              <div class="summary-row tab-summary-row">
                <div v-if="showAmount" class="summary-card summary-card-soft summary-card-red">
                  <span class="summary-label">自由持仓</span>
                  <strong class="summary-value">{{ parseFloat(freeAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" class="summary-card summary-card-green">
                  <span class="summary-label">自由估算</span>
                  <strong class="summary-value">{{ parseFloat(freeGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(freeGains[1]) ? '' : '（' + freeGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" class="summary-card summary-card-red">
                  <span class="summary-label">自由持有</span>
                  <strong class="summary-value">{{ parseFloat(freeCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(freeCostGains[1]) ? '' : '（' + freeCostGains[1] + '%）' }}</span>
                </div>
              </div>
            </div>
            <div
              v-loading="loadingList"
              :element-loading-background="
                darkMode ? 'rgba(0, 0, 0, 0.9)' : 'rgba(255, 255, 255, 0.9)'
              "
              class="table-row"
              style="min-height:160px"
            >
              <table :class="tableHeight">
                <thead>
                  <tr>
                    <th class="align-left">基金名称（{{ freeDataList.length }}）</th>
                    <th v-if="isEdit">基金代码</th>
                    <th v-if="showGSZ && !isEdit">估算净值</th>
                    <th style="text-align:center" v-if="isEdit && (showCostRate || showCost)">成本价</th>
                    <th @click="sortList('amount')" v-if="showAmount" class="pointer">
                      持有额 <span :class="sortType.amount" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('costGains')" v-if="showCost" class="pointer">
                      持有收益 <span :class="sortType.costGains" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('costGainsRate')" v-if="showCostRate" class="pointer">
                      持有收益率 <span :class="sortType.costGainsRate" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('gszzl')" class="pointer">
                      涨跌幅 <span :class="sortType.gszzl" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('gains')" v-if="showGains" class="pointer">
                      估算收益 <span :class="sortType.gains" class="down-arrow"></span>
                    </th>
                    <th v-if="!isEdit">更新时间</th>
                    <th style="text-align:center" v-if="isEdit && (showAmount || showGains || showCost || showCostRate)">持有份额</th>
                    <th v-if="isEdit && BadgeContent == 1">特别关注</th>
                    <th v-if="isEdit">删除</th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="(el, index) in freeDataList"
                    :key="el.fundcode"
                    :draggable="isEdit"
                    :class="drag"
                    @dragstart="handleDragStart($event, el)"
                    @dragover.prevent="handleDragOver($event, el)"
                    @dragenter="handleDragEnter($event, el, index)"
                    @dragend="handleDragEnd($event, el)"
                  >
                    <td :class="isEdit ? 'fundName-noclick align-left' : 'fundName align-left'" :title="el.name" @click.stop="!isEdit && fundDetail(el)">
                      <span class="hasReplace-tip" v-if="el.hasReplace">✔</span>{{ el.name }}
                    </td>
                    <td v-if="isEdit">{{ el.fundcode }}</td>
                    <td v-if="showGSZ && !isEdit">{{ el.gsz }}</td>
                    <td v-if="isEdit && (showCostRate || showCost)">
                      <input class="btn num" placeholder="持仓成本价" v-model="el.cost" @input="changeCost(el, index)" type="text" />
                    </td>
                    <td v-if="showAmount">{{ parseFloat(el.amount).toLocaleString("zh", { minimumFractionDigits: 2 }) }}</td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">{{ parseFloat(el.costGains).toLocaleString("zh", { minimumFractionDigits: 2 }) }}</td>
                    <td v-if="showCostRate" :class="el.costGainsRate >= 0 ? 'up' : 'down'">{{ el.cost > 0 ? el.costGainsRate + "%" : "" }}</td>
                    <td :class="el.gszzl >= 0 ? 'up' : 'down'">{{ el.gszzl }}%</td>
                    <td v-if="showGains" :class="el.gains >= 0 ? 'up' : 'down'">{{ parseFloat(el.gains).toLocaleString("zh", { minimumFractionDigits: 2 }) }}</td>
                    <td v-if="!isEdit">{{ formatUpdateTime(el) }}</td>
                    <th style="text-align:center" v-if="isEdit && (showAmount || showGains || showCost || showCostRate)">
                      <input class="btn num" placeholder="输入持有份额" v-model="el.num" @input="changeNum(el, index)" type="text" />
                    </th>
                    <td v-if="isEdit && BadgeContent == 1">
                      <input @click="slt(el.fundcode)" :class="el.fundcode == RealtimeFundcode ? 'slt' : ''" class="btn edit" value="✔" type="button" />
                    </td>
                    <td v-if="isEdit">
                      <input @click="dlt(el.fundcode)" class="btn red edit" value="✖" type="button" />
                    </td>
                  </tr>
                  <tr v-if="!freeDataList.length">
                    <td class="empty-row" :colspan="fundTableColspan">暂无财务自由账户持仓</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </el-tab-pane>
          <el-tab-pane :label="`财务安全（${safeDataList.length}）`" name="safe">
            <div class="summary-panel tab-summary-panel" v-if="showCost || showGains || showAmount">
              <div class="summary-row tab-summary-row">
                <div v-if="showAmount" class="summary-card summary-card-soft summary-card-red">
                  <span class="summary-label">安全持仓</span>
                  <strong class="summary-value">{{ parseFloat(safeAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" class="summary-card summary-card-green">
                  <span class="summary-label">安全估算</span>
                  <strong class="summary-value">{{ parseFloat(safeGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(safeGains[1]) ? '' : '（' + safeGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" class="summary-card summary-card-red">
                  <span class="summary-label">安全持有</span>
                  <strong class="summary-value">{{ parseFloat(safeCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(safeCostGains[1]) ? '' : '（' + safeCostGains[1] + '%）' }}</span>
                </div>
              </div>
            </div>
            <div
              v-loading="loadingList"
              :element-loading-background="
                darkMode ? 'rgba(0, 0, 0, 0.9)' : 'rgba(255, 255, 255, 0.9)'
              "
              class="table-row"
              style="min-height:160px"
            >
              <table :class="tableHeight">
                <thead>
                  <tr>
                    <th class="align-left">基金名称（{{ safeDataList.length }}）</th>
                    <th v-if="isEdit">基金代码</th>
                    <th v-if="showGSZ && !isEdit">估算净值</th>
                    <th style="text-align:center" v-if="isEdit && (showCostRate || showCost)">成本价</th>
                    <th @click="sortList('amount')" v-if="showAmount" class="pointer">
                      持有额 <span :class="sortType.amount" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('costGains')" v-if="showCost" class="pointer">
                      持有收益 <span :class="sortType.costGains" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('costGainsRate')" v-if="showCostRate" class="pointer">
                      持有收益率 <span :class="sortType.costGainsRate" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('gszzl')" class="pointer">
                      涨跌幅 <span :class="sortType.gszzl" class="down-arrow"></span>
                    </th>
                    <th @click="sortList('gains')" v-if="showGains" class="pointer">
                      估算收益 <span :class="sortType.gains" class="down-arrow"></span>
                    </th>
                    <th v-if="!isEdit">更新时间</th>
                    <th style="text-align:center" v-if="isEdit && (showAmount || showGains || showCost || showCostRate)">持有份额</th>
                    <th v-if="isEdit && BadgeContent == 1">特别关注</th>
                    <th v-if="isEdit">删除</th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="(el, index) in safeDataList"
                    :key="el.fundcode"
                    :draggable="isEdit"
                    :class="drag"
                    @dragstart="handleDragStart($event, el)"
                    @dragover.prevent="handleDragOver($event, el)"
                    @dragenter="handleDragEnter($event, el, index)"
                    @dragend="handleDragEnd($event, el)"
                  >
                    <td :class="isEdit ? 'fundName-noclick align-left' : 'fundName align-left'" :title="el.name" @click.stop="!isEdit && fundDetail(el)">
                      <span class="hasReplace-tip" v-if="el.hasReplace">✔</span>{{ el.name }}
                    </td>
                    <td v-if="isEdit">{{ el.fundcode }}</td>
                    <td v-if="showGSZ && !isEdit">{{ el.gsz }}</td>
                    <td v-if="isEdit && (showCostRate || showCost)">
                      <input class="btn num" placeholder="持仓成本价" v-model="el.cost" @input="changeCost(el, index)" type="text" />
                    </td>
                    <td v-if="showAmount">{{ parseFloat(el.amount).toLocaleString("zh", { minimumFractionDigits: 2 }) }}</td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">{{ parseFloat(el.costGains).toLocaleString("zh", { minimumFractionDigits: 2 }) }}</td>
                    <td v-if="showCostRate" :class="el.costGainsRate >= 0 ? 'up' : 'down'">{{ el.cost > 0 ? el.costGainsRate + "%" : "" }}</td>
                    <td :class="el.gszzl >= 0 ? 'up' : 'down'">{{ el.gszzl }}%</td>
                    <td v-if="showGains" :class="el.gains >= 0 ? 'up' : 'down'">{{ parseFloat(el.gains).toLocaleString("zh", { minimumFractionDigits: 2 }) }}</td>
                    <td v-if="!isEdit">{{ formatUpdateTime(el) }}</td>
                    <th style="text-align:center" v-if="isEdit && (showAmount || showGains || showCost || showCostRate)">
                      <input class="btn num" placeholder="输入持有份额" v-model="el.num" @input="changeNum(el, index)" type="text" />
                    </th>
                    <td v-if="isEdit && BadgeContent == 1">
                      <input @click="slt(el.fundcode)" :class="el.fundcode == RealtimeFundcode ? 'slt' : ''" class="btn edit" value="✔" type="button" />
                    </td>
                    <td v-if="isEdit">
                      <input @click="dlt(el.fundcode)" class="btn red edit" value="✖" type="button" />
                    </td>
                  </tr>
                  <tr v-if="!safeDataList.length">
                    <td class="empty-row" :colspan="fundTableColspan">暂无财务安全账户持仓</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </el-tab-pane>
          <el-tab-pane :label="`股票（${stockDataList.length}）`" name="stock">
            <div class="summary-panel tab-summary-panel" v-if="showCost || showGains || showAmount">
              <div class="summary-row tab-summary-row">
                <div v-if="showAmount" class="summary-card summary-card-soft summary-card-red">
                  <span class="summary-label">股票持仓</span>
                  <strong class="summary-value">{{ parseFloat(stockAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" class="summary-card summary-card-green">
                  <span class="summary-label">股票当日</span>
                  <strong class="summary-value">{{ parseFloat(stockGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(stockGains[1]) ? '' : '（' + stockGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" class="summary-card summary-card-red">
                  <span class="summary-label">股票持有</span>
                  <strong class="summary-value">{{ parseFloat(stockCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(stockCostGains[1]) ? '' : '（' + stockCostGains[1] + '%）' }}</span>
                </div>
              </div>
            </div>
            <div class="table-row stock-table-row" style="min-height:160px">
              <table :class="tableHeight">
                <thead>
                  <tr>
                    <th class="align-left">股票名称（{{ stockDataList.length }}）</th>
                    <th v-if="isEdit">股票代码</th>
                    <th
                      @click="sortStockList('latestPriceValue')"
                      class="pointer"
                    >
                      最新价
                      <span :class="stockSortType.latestPriceValue" class="down-arrow"></span>
                    </th>
                    <th
                      style="text-align:center"
                      v-if="showCostRate || showCost"
                    >
                      成本价
                    </th>
                    <th @click="sortStockList('amount')" v-if="showAmount" class="pointer">
                      持有额
                      <span :class="stockSortType.amount" class="down-arrow"></span>
                    </th>
                    <th @click="sortStockList('costGains')" v-if="showCost" class="pointer">
                      持有收益
                      <span :class="stockSortType.costGains" class="down-arrow"></span>
                    </th>
                    <th @click="sortStockList('costGainsRate')" v-if="showCostRate" class="pointer">
                      持有收益率
                      <span :class="stockSortType.costGainsRate" class="down-arrow"></span>
                    </th>
                    <th @click="sortStockList('gszzl')" class="pointer">
                      涨跌幅
                      <span :class="stockSortType.gszzl" class="down-arrow"></span>
                    </th>
                    <th @click="sortStockList('gains')" v-if="showGains" class="pointer">
                      当日收益
                      <span :class="stockSortType.gains" class="down-arrow"></span>
                    </th>
                    <th v-if="!isEdit">更新时间</th>
                    <th
                      style="text-align:center"
                      v-if="
                        isEdit &&
                          (showAmount || showGains || showCost || showCostRate)
                      "
                    >
                      持有股数
                    </th>
                    <th v-if="isEdit">删除</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(el, index) in stockDataList" :key="el.stockcode">
                    <td
                      :class="isEdit ? 'fundName-noclick align-left stockNameCell' : 'fundName align-left stockNameCell'"
                      :title="el.name"
                      @click.stop="!isEdit && stockDetail(el)"
                    >
                      {{ el.name }}
                    </td>
                    <td v-if="isEdit">{{ el.stockcode }}</td>
                    <td>{{ el.latestPrice }}</td>
                    <td v-if="showCostRate || showCost">
                      <input
                        v-if="isEdit"
                        class="btn num"
                        placeholder="持仓成本价"
                        v-model="el.cost"
                        @input="changeStockCost(el, index)"
                        type="text"
                      />
                      <span v-else>{{ formatStockPrice(el.cost) }}</span>
                    </td>
                    <td v-if="showAmount">
                      {{
                        parseFloat(el.amount).toLocaleString("zh", {
                          minimumFractionDigits: 2,
                        })
                      }}
                    </td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">
                      {{
                        parseFloat(el.costGains).toLocaleString("zh", {
                          minimumFractionDigits: 2,
                        })
                      }}
                    </td>
                    <td
                      v-if="showCostRate"
                      :class="el.costGainsRate >= 0 ? 'up' : 'down'"
                    >
                      {{ el.cost > 0 ? el.costGainsRate + "%" : "" }}
                    </td>
                    <td :class="el.gszzl >= 0 ? 'up' : 'down'">{{ el.gszzl }}%</td>
                    <td v-if="showGains" :class="el.gains >= 0 ? 'up' : 'down'">
                      {{
                        parseFloat(el.gains).toLocaleString("zh", {
                          minimumFractionDigits: 2,
                        })
                      }}
                    </td>
                    <td v-if="!isEdit">{{ formatStockUpdateTime(el.gztime) }}</td>
                    <th
                      style="text-align:center"
                      v-if="
                        isEdit &&
                          (showAmount || showGains || showCost || showCostRate)
                      "
                    >
                      <input
                        class="btn num"
                        placeholder="输入持有股数"
                        v-model="el.num"
                        @input="changeStockNum(el, index)"
                        type="text"
                      />
                    </th>
                    <td v-if="isEdit">
                      <input
                        @click="dltStock(el.stockcode)"
                        class="btn red edit"
                        value="✖"
                        type="button"
                      />
                    </td>
                  </tr>
                  <tr v-if="!stockDataList.length">
                    <td class="empty-row" :colspan="stockTableColspan">暂无股票持仓</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </el-tab-pane>
        </el-tabs>
      </div>
    </div>
    <p v-if="isEdit" class="tips">
      特别关注功能介绍：指定一个基金，在程序图标中以角标的形式实时更新，请在设置中选择角标类型与内容。
    </p>

    <div v-show="isEdit" class="input-row gear-input-row">
      <el-switch
        v-model="darkMode"
        @change="changeDarkMode"
        active-color="#484848"
        inactive-color="#13ce66"
        inactive-text="标准模式"
        active-text="暗色模式"
      >
      </el-switch>
      <span class="slider-title">界面灰度：</span>
      <el-slider
        class="slider"
        v-model="grayscaleValue"
        @change="changeGrayscaleValue"
        :format-tooltip="formatTooltip"
      ></el-slider>
      <span class="slider-title">透明度：</span>
      <el-slider
        class="slider"
        :max="90"
        v-model="opacityValue"
        @change="changeOpacityValue"
        :format-tooltip="formatTooltip"
      ></el-slider>
      <!-- <input v-model="containerWidth" type="number" />
      <input v-model="containerHeight" type="number" /> -->
    </div>

    <div class="input-row action-row">
      <input class="btn" type="button" @click="market" value="行情中心" />
      <input
        class="btn"
        v-if="isDuringDate"
        type="button"
        :value="isLiveUpdate ? '暂停更新' : '实时更新'"
        :title="
          isLiveUpdate ? '正在实时更新，点击暂停' : '已暂停，点击切换为实时更新'
        "
        @click="changeLiveUpdate"
      />
      <input class="btn" v-if="!isDuringDate" type="button" value="休市中" />
      <input
        class="btn"
        type="button"
        :value="isEdit ? '完成编辑' : '编辑'"
        @click="isEdit = !isEdit"
      />
      <input class="btn" type="button" value="设置" @click="option" />
      <input class="btn" type="button" value="日志" @click="changelog" />
      <input
        class="btn primary"
        type="button"
        title="φ(>ω<*)"
        value="打赏"
        @click="reward"
      />
    </div>
    <div
      class="refresh"
      :class="{ isRefresh: isRefresh }"
      title="手动刷新数据"
      @click="refresh"
    >
      <i class="el-icon-refresh"></i>
    </div>
    <market
      :darkMode="darkMode"
      @close="closeCharts"
      ref="marketShadow"
    ></market>
    <ind-detail @close="closeCharts" :darkMode="darkMode" ref="indDetail">
    </ind-detail>
    <fund-detail
      @close="closeCharts"
      :fund="sltFund"
      :darkMode="darkMode"
      ref="fundDetail"
    ></fund-detail>
    <reward @close="rewardShadow = false" ref="reward"></reward>
    <change-log
      @close="closeChangelog"
      :darkMode="darkMode"
      ref="changelog"
      :top="30"
    ></change-log>
  </div>
</template>

<script>
const { version } = require("../../package.json");
import reward from "../common/reward";
import indDetail from "../common/indDetail";
import fundDetail from "../common/fundDetail";
import changeLog from "../common/changeLog";
import market from "../common/market";
//防抖
let timeout = null;
function debounce(fn, wait = 700) {
  if (timeout !== null) clearTimeout(timeout);
  timeout = setTimeout(fn, wait);
}

export default {
  components: {
    reward,
    fundDetail,
    indDetail,
    changeLog,
    market,
  },
  data() {
    return {
      isEdit: false,
      fundcode: "",
      isAdd: false,
      indFundData: [],
      isLiveUpdate: false,
      isDuringDate: false,
      RealtimeFundcode: null,
      RealtimeIndcode: null,
      dataList: [],
      dataListDft: [],
      stockDataList: [],
      stockDataListDft: [],
      myVar: null,
      myVar1: null,
      rewardShadow: false,
      checked: "wepay",
      showGains: false,
      showAmount: false,
      showCost: false,
      showCostRate: false,
      showGSZ: false,
      fundList: ["001618"],
      fundListM: [],
      stockListM: [],
      sortType: {
        gszzl: "none",
        amount: "none",
        gains: "none",
        costGains: "none",
        costGainsRate: "none",
      },
      stockSortType: {
        latestPriceValue: "none",
        amount: "none",
        gains: "none",
        costGains: "none",
        costGainsRate: "none",
        gszzl: "none",
      },
      sortTypeObj: {
        name: null,
        value: null,
      },
      stockSortTypeObj: {
        name: null,
        type: null,
      },
      searchOptions: [],
      value: [],
      list: [],
      loading: false,
      activeHoldingsTab: "home",
      dragging: null,
      showAddSeciInput: false,
      seciList: ["1.000001", "1.000300", "0.399001", "0.399006"],
      allSeciList: [
        {
          value: "1.000001",
          label: "上证指数",
        },
        {
          value: "1.000300",
          label: "沪深300",
        },
        {
          value: "0.399001",
          label: "深证成指",
        },
        {
          value: "1.000688",
          label: "科创50",
        },
        {
          value: "0.399006",
          label: "创业板指",
        },
        {
          value: "0.399005",
          label: "中小板指",
        },
        {
          value: "100.HSI",
          label: "恒生指数",
        },
        {
          value: "100.DJIA",
          label: "道琼斯",
        },
        {
          value: "100.NDX",
          label: "纳斯达克",
        },
        {
          value: "100.SPX",
          label: "标普500",
        },
      ],
      sltSeci: "",
      darkMode: false,
      normalFontSize: false,
      diyContainer: false,
      containerWidth: 790,
      containerHeight: 590,
      detailShadow: false,
      changelogShadow: false,
      sltFund: {},
      sltIndCode: "",
      localVersion: version,
      BadgeContent: 1,
      showBadge: 1,
      userId: null,
      loadingInd: false,
      loadingList: true,
      isGetStorage: false,
      zoom: {
        zoom: 1,
      },
      grayscale: {},
      grayscaleValue: 0,
      opacity: {},
      opacityValue: 0,
      isRefresh: false,
      marketShadow: false,
    };
  },
  mounted() {
    setTimeout(() => {
      let aa = window.screen.width;
      let bb = this.$refs.app.clientWidth;
      if (aa < bb) {
        this.zoom = {
          zoom: aa / bb,
        };
      }
    }, 10);
    this.init();
  },
  computed: {
    allHoldingList() {
      return [...this.dataList, ...this.stockDataList];
    },
    freeDataList() {
      return this.dataList.filter(item => item.accountType === '财务自由账户');
    },
    safeDataList() {
      return this.dataList.filter(item => item.accountType === '财务安全账户');
    },
    freeAmount() {
      return this.sumAmount(this.freeDataList);
    },
    safeAmount() {
      return this.sumAmount(this.safeDataList);
    },
    freeGains() {
      return this.sumGains(this.freeDataList);
    },
    safeGains() {
      return this.sumGains(this.safeDataList);
    },
    freeCostGains() {
      return this.sumCostGains(this.freeDataList);
    },
    safeCostGains() {
      return this.sumCostGains(this.safeDataList);
    },
    fundAmount() {
      return this.sumAmount(this.dataList);
    },
    stockAmount() {
      return this.sumAmount(this.stockDataList);
    },
    allAmount() {
      return this.sumAmount(this.allHoldingList);
    },
    fundGains() {
      return this.sumGains(this.dataList);
    },
    stockGains() {
      return this.sumGains(this.stockDataList);
    },
    allGains() {
      return this.sumGains(this.allHoldingList);
    },
    fundCostGains() {
      return this.sumCostGains(this.dataList);
    },
    stockCostGains() {
      return this.sumCostGains(this.stockDataList);
    },
    allCostGains() {
      return this.sumCostGains(this.allHoldingList);
    },
    containerClass() {
      let className = "";
      if (this.normalFontSize) {
        className += "normalFontSize ";
      }
      if (this.darkMode) {
        className += "darkMode ";
      }
      if (this.changelogShadow) {
        className += "changelog-container";
      } else if (this.rewardShadow) {
        className += "more-height";
      } else if (this.detailShadow) {
        className += "detail-container";
      } else if (this.isEdit) {
        className += "more-width";
      } else {
        if (this.activeHoldingsTab === "stock") {
          className += "stock-more-width ";
        }
        let tablist = [
          this.showAmount,
          this.showGains,
          this.showCost,
          this.showCostRate,
          this.showGSZ,
        ];
        let num = 0;
        tablist.forEach((val) => {
          if (val) {
            num++;
          }
        });
        className += "num-width-" + num;
      }
      return className;
    },
    userSeciList() {
      return this.allSeciList.filter((val) => {
        return this.seciList.indexOf(val.value) == -1;
      });
    },
    tableHeight() {
      if (this.isEdit) {
        return "table-more-height";
      }
    },
    fundTableColspan() {
      let total = 4;
      if (this.showGSZ && !this.isEdit) {
        total += 1;
      }
      if (this.isEdit && (this.showCostRate || this.showCost)) {
        total += 1;
      }
      if (this.showAmount) {
        total += 1;
      }
      if (this.showCost) {
        total += 1;
      }
      if (this.showCostRate) {
        total += 1;
      }
      if (this.showGains) {
        total += 1;
      }
      if (!this.isEdit) {
        total += 1;
      }
      if (this.isEdit && (this.showAmount || this.showGains || this.showCost || this.showCostRate)) {
        total += 1;
      }
      if (this.isEdit && this.BadgeContent == 1) {
        total += 1;
      }
      if (this.isEdit) {
        total += 1;
      }
      return total;
    },
    stockTableColspan() {
      let total = 4;
      if (this.showCostRate || this.showCost) {
        total += 1;
      }
      if (this.showAmount) {
        total += 1;
      }
      if (this.showCost) {
        total += 1;
      }
      if (this.showCostRate) {
        total += 1;
      }
      if (this.showGains) {
        total += 1;
      }
      if (!this.isEdit) {
        total += 1;
      }
      if (this.isEdit && (this.showAmount || this.showGains || this.showCost || this.showCostRate)) {
        total += 1;
      }
      if (this.isEdit) {
        total += 1;
      }
      return total;
    },
    drag() {
      if (this.isEdit) {
        return "table-drag";
      } else {
        return "";
      }
    },
  },
  watch: {
    //编辑状态停止更新
    isEdit(val) {
      if (val) {
        clearInterval(this.myVar);
        clearInterval(this.myVar1);
        this.dataList = [...this.dataListDft];
        this.stockDataList = [...this.stockDataListDft];
        for (const key in this.sortType) {
          if (this.sortType.hasOwnProperty(key)) {
            this.sortType[key] = "none";
          }
        }
        for (const key in this.stockSortType) {
          if (this.stockSortType.hasOwnProperty(key)) {
            this.stockSortType[key] = "none";
          }
        }
      } else {
        this.checkInterval();
      }
    },
  },
  methods: {
    sumAmount(list) {
      let totalAmount = 0;
      list.forEach((val) => {
        totalAmount += parseFloat(val.amount || 0);
      });
      return totalAmount.toFixed(2);
    },
    sumGains(list) {
      let allGains = 0;
      let allNum = 0;
      list.forEach((val) => {
        allGains += parseFloat(val.gains || 0);
        allNum += parseFloat(val.amount || 0);
      });
      allGains = allGains.toFixed(2);
      let allGainsRate = allNum ? ((allGains * 100) / allNum).toFixed(2) : NaN;
      return [allGains, allGainsRate];
    },
    sumCostGains(list) {
      let allCostGains = 0;
      let allNum = 0;
      list.forEach((val) => {
        allCostGains += parseFloat(val.costGains || 0);
        allNum += parseFloat(val.amount || 0);
      });
      allCostGains = allCostGains.toFixed(2);
      const denominator = allNum - allCostGains;
      let allCostGainsRate = denominator ? ((allCostGains * 100) / denominator).toFixed(2) : NaN;
      return [allCostGains, allCostGainsRate];
    },
    toNullableNumber(value) {
      if (value === null || value === undefined || value === "") {
        return null;
      }

      const numberValue = Number(value);
      return Number.isNaN(numberValue) ? null : numberValue;
    },
    parseFundEstimateResponse(payload) {
      if (!payload || typeof payload !== "string") {
        return null;
      }

      const matched = payload.match(/^jsonpgz\((.*)\);?$/);
      if (!matched || !matched[1]) {
        return null;
      }

      try {
        return JSON.parse(matched[1]);
      } catch (error) {
        return null;
      }
    },
    fetchFundEstimate(code) {
      return this.$axios
        .get(`https://fundgz.1234567.com.cn/js/${code}.js`, {
          params: {
            rt: Date.now(),
          },
          responseType: "text",
          transformResponse: [(value) => value],
        })
        .then((res) => this.parseFundEstimateResponse(res.data))
        .catch(() => null);
    },
    applyFundEstimate(baseData, estimateData) {
      if (!estimateData) {
        return baseData;
      }

      const estimatedNav = this.toNullableNumber(estimateData.gsz);
      const estimatedRate = this.toNullableNumber(estimateData.gszzl);

      if (estimatedNav === null) {
        return baseData;
      }

      return {
        ...baseData,
        gsz: estimatedNav,
        gszzl: estimatedRate === null ? baseData.gszzl : estimatedRate,
        gztime: estimateData.gztime || baseData.gztime,
        hasReplace: false,
      };
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
        num: this.toNullableNumber(source.num ?? source.quantity ?? source.shares) || 0,
        cost: this.toNullableNumber(
          source.cost ?? source.costPrice ?? source.avgCost ?? source.price
        ) || 0,
      };
    },
    formatStockPrice(value) {
      const numberValue = this.toNullableNumber(value);
      if (numberValue === null) {
        return "--";
      }

      return numberValue.toFixed(2);
    },
    formatStockUpdateTime(timestamp) {
      if (!timestamp) {
        return "--";
      }

      const timeValue = Number(timestamp);
      if (Number.isNaN(timeValue)) {
        return String(timestamp);
      }

      const date = new Date(timeValue * 1000);
      if (Number.isNaN(date.getTime())) {
        return "--";
      }

      const month = `${date.getMonth() + 1}`.padStart(2, "0");
      const day = `${date.getDate()}`.padStart(2, "0");
      const hour = `${date.getHours()}`.padStart(2, "0");
      const minute = `${date.getMinutes()}`.padStart(2, "0");
      return `${month}-${day} ${hour}:${minute}`;
    },
    calculateStockDailyGain(latestPrice, prevClose, num) {
      if (latestPrice === null || prevClose === null) {
        return 0;
      }

      return ((latestPrice - prevClose) * num).toFixed(2);
    },
    calculateStockCostGain(latestPrice, cost, num) {
      if (cost === null || cost === undefined || cost === "") {
        return 0;
      }

      return ((latestPrice - cost) * num).toFixed(2);
    },
    calculateStockCostGainRate(latestPrice, cost) {
      if (!cost) {
        return 0;
      }

      return (((latestPrice - cost) / cost) * 100).toFixed(2);
    },
    refresh() {
      this.init();
      this.isRefresh = true;
      setTimeout(() => {
        this.isRefresh = false;
      }, 1500);
    },
    formatTooltip(val) {
      return val + "%";
    },
    changeGrayscaleValue(val) {
      this.grayscale = {
        filter: "grayscale(" + val / 100 + ")",
      };
      chrome.storage.sync.set({
        grayscaleValue: this.grayscaleValue,
      });
    },
    changeOpacityValue(val) {
      this.opacity = {
        opacity: 1 - val / 100,
      };
      chrome.storage.sync.set({
        opacityValue: this.opacityValue,
      });
    },
    init() {
      chrome.storage.sync.get(
        [
          "RealtimeFundcode",
          "RealtimeIndcode",
          "fundListM",
          "showAmount",
          "showGains",
          "fundList",
          "seciList",
          "darkMode",
          "normalFontSize",
          "isLiveUpdate",
          "showCost",
          "showCostRate",
          "showGSZ",
          "version",
          "showBadge",
          "BadgeContent",
          "userId",
          "grayscaleValue",
          "opacityValue",
          "sortTypeObj",
          "stockSortTypeObj",
          "stockListM",
        ],
        (res) => {
          this.fundList = res.fundList ? res.fundList : this.fundList;
          if (res.fundListM) {
            this.fundListM = res.fundListM;
          } else {
            for (const fund of this.fundList) {
              let val = {
                code: fund,
                num: 0,
              };
              this.fundListM.push(val);
            }
            chrome.storage.sync.set({
              fundListM: this.fundListM,
            });
          }
          this.stockListM = (res.stockListM || [])
            .map((item) => this.normalizeStockItem(item))
            .filter(Boolean);
          if (res.userId) {
            this.userId = res.userId;
          } else {
            this.userId = this.getGuid();
            chrome.storage.sync.set({
              userId: this.userId,
            });
          }
          this.darkMode = res.darkMode ? res.darkMode : false;
          this.normalFontSize = res.normalFontSize ? res.normalFontSize : false;
          this.seciList = res.seciList ? res.seciList : this.seciList;
          this.showAmount = res.showAmount ? res.showAmount : false;
          this.showGains = res.showGains ? res.showGains : false;
          this.RealtimeFundcode = res.RealtimeFundcode;
          this.RealtimeIndcode = res.RealtimeIndcode;
          this.isLiveUpdate = res.isLiveUpdate ? res.isLiveUpdate : false;
          this.showCost = res.showCost ? res.showCost : false;
          this.showCostRate = res.showCostRate ? res.showCostRate : false;
          this.showGSZ = res.showGSZ ? res.showGSZ : false;
          this.BadgeContent = res.BadgeContent ? res.BadgeContent : 1;
          this.showBadge = res.showBadge ? res.showBadge : 1;
          this.grayscaleValue = res.grayscaleValue ? res.grayscaleValue : 0;
          this.opacityValue = res.opacityValue ? res.opacityValue : 0;
          this.sortTypeObj = res.sortTypeObj ? res.sortTypeObj : {};
          this.stockSortTypeObj = res.stockSortTypeObj ? res.stockSortTypeObj : {};

          if (this.seciList.length > 0) {
            this.loadingInd = true;
          }

          this.grayscale = {
            filter: "grayscale(" + this.grayscaleValue / 100 + ")",
          };
          this.opacity = {
            opacity: 1 - this.opacityValue / 100,
          };

          this.isGetStorage = true;
          this.getIndFundData();
          this.getData();
          this.getStockData();
          this.checkInterval(true);

          let ver = res.version ? res.version : "1.0.0";
          if (ver != this.localVersion) {
            this.changelog();
          }
        }
      );
    },
    getGuid() {
      return "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx".replace(/[xy]/g, function(
        c
      ) {
        var r = (Math.random() * 16) | 0,
          v = c == "x" ? r : (r & 0x3) | 0x8;
        return v.toString(16);
      });
    },
    indDetail(val) {
      // this.sltIndCode = val.f13 + "." + val.f12;
      this.detailShadow = true;
      this.$refs.indDetail.init(val);
    },
    fundDetail(val) {
      this.sltFund = val;
      this.detailShadow = true;
      this.$refs.fundDetail.init();
    },
    stockDetail(val) {
      this.detailShadow = true;
      this.$refs.indDetail.init({
        f12: val.stockcode,
        f13: Number(String(val.secid).split(".")[0]),
        f14: val.name,
      });
    },
    closeCharts() {
      this.detailShadow = false;
    },
    market() {
      this.detailShadow = true;
      this.$refs.marketShadow.init();
    },
    checkInterval(isFirst) {
      clearInterval(this.myVar);
      clearInterval(this.myVar1);
      chrome.runtime.sendMessage({ type: "DuringDate" }, (response) => {
        this.isDuringDate = response.farewell;
        if (this.isLiveUpdate && this.isDuringDate) {
          if (!isFirst) {
            this.getIndFundData();
            this.getData();
            this.getStockData();
          }
          this.myVar = setInterval(() => {
            this.getIndFundData();
          }, 5 * 1000);
          this.myVar1 = setInterval(() => {
            this.getData();
            this.getStockData();
          }, 60 * 1000);
        } else {
          clearInterval(this.myVar);
          clearInterval(this.myVar1);
        }
      });
    },
    selectChange() {
      this.searchOptions = [];
    },
    remoteMethod(query) {
      if (query !== "") {
        this.loading = true;
        let url =
          "https://fundsuggest.eastmoney.com/FundSearch/api/FundSearchAPI.ashx?&m=9&key=" +
          query +
          "&_=" +
          new Date().getTime();
        this.$axios.get(url).then((res) => {
          this.searchOptions = res.data.Datas.filter((val) => {
            let hasCode = this.fundListM.some((currentValue, index, array) => {
              return currentValue.code == val.CODE;
            });
            return !hasCode;
          }).map((val) => {
            return {
              value: val.CODE,
              label: val.NAME,
            };
          });
          this.loading = false;
        });
      } else {
        this.searchOptions = [];
      }
    },

    option() {
      if (chrome.runtime && chrome.runtime.openOptionsPage) {
        chrome.runtime.openOptionsPage();
        return;
      }

      chrome.tabs.create({ url: chrome.runtime.getURL("options/options.html") });
    },
    reward() {
      this.rewardShadow = true;
      this.$refs.reward.init();
    },
    changelog() {
      this.changelogShadow = true;
      this.$refs.changelog.init();
    },
    closeChangelog() {
      this.changelogShadow = false;
      chrome.storage.sync.set({
        version: this.localVersion,
      });
    },
    sortList(type) {
      for (const key in this.sortType) {
        if (this.sortType.hasOwnProperty(key)) {
          if (key != type) {
            this.sortType[key] = "none";
          }
        }
      }
      this.sortType[type] =
        this.sortType[type] == "desc"
          ? "asc"
          : this.sortType[type] == "asc"
          ? "none"
          : "desc";
      if (this.sortType[type] == "none") {
        this.dataList = [...this.dataListDft];
      } else {
        this.dataList = [...this.dataList].sort(
          this.compare(type, this.sortType[type])
        );
      }
      this.sortTypeObj = {
        name: type,
        type: this.sortType[type],
      };
      chrome.storage.sync.set({
        sortTypeObj: this.sortTypeObj,
      });
    },
    sortStockList(type) {
      for (const key in this.stockSortType) {
        if (this.stockSortType.hasOwnProperty(key) && key != type) {
          this.stockSortType[key] = "none";
        }
      }
      this.stockSortType[type] =
        this.stockSortType[type] == "desc"
          ? "asc"
          : this.stockSortType[type] == "asc"
          ? "none"
          : "desc";
      if (this.stockSortType[type] == "none") {
        this.stockDataList = [...this.stockDataListDft];
      } else {
        this.stockDataList = [...this.stockDataList].sort(
          this.compare(type, this.stockSortType[type])
        );
      }
      this.stockSortTypeObj = {
        name: type,
        type: this.stockSortType[type],
      };
      chrome.storage.sync.set({
        stockSortTypeObj: this.stockSortTypeObj,
      });
    },
    compare(property, type) {
      return function(obj1, obj2) {
        const normalizeSortValue = (value) => {
          if (value === null || value === undefined || value === "") {
            return Number.NEGATIVE_INFINITY;
          }

          const numericValue = Number(value);
          if (!Number.isNaN(numericValue)) {
            return numericValue;
          }

          return String(value);
        };

        const val1 = normalizeSortValue(obj1[property]);
        const val2 = normalizeSortValue(obj2[property]);
        if (typeof val1 === "string" || typeof val2 === "string") {
          if (type == "asc") {
            return String(val1).localeCompare(String(val2), "zh-Hans-CN");
          }
          return String(val2).localeCompare(String(val1), "zh-Hans-CN");
        }
        if (type == "asc") {
          return val1 - val2;
        } else {
          return val2 - val1;
        }
      };
    },

    changeDarkMode() {
      chrome.storage.sync.set({
        darkMode: this.darkMode,
      });
    },
    changeLiveUpdate() {
      chrome.storage.sync.set(
        {
          isLiveUpdate: !this.isLiveUpdate,
        },
        () => {
          this.isLiveUpdate = !this.isLiveUpdate;
          this.checkInterval();
        }
      );
    },
    saveSeci() {
      this.seciList.push(this.sltSeci);
      chrome.storage.sync.set(
        {
          seciList: this.seciList,
        },
        () => {
          this.sltSeci = "";
          this.getIndFundData();
        }
      );
    },
    dltIndFund(ind) {
      this.seciList.splice(ind, 1);
      chrome.storage.sync.set(
        {
          seciList: this.seciList,
        },
        () => {
          this.getIndFundData();
        }
      );
    },
    getIndFundData() {
      let seciListStr = this.seciList.join(",");
      let url =
        "https://push2.eastmoney.com/api/qt/ulist.np/get?fltt=2&fields=f2,f3,f4,f12,f13,f14&secids=" +
        seciListStr +
        "&_=" +
        new Date().getTime();
      this.$axios.get(url).then((res) => {
        this.loadingInd = false;
        this.indFundData = res.data.data.diff;
      });
    },
    getStockData() {
      if (!this.stockListM.length) {
        this.stockDataList = [];
        this.stockDataListDft = [];
        return;
      }

      const stockList = this.stockListM
        .map((item) => this.normalizeStockItem(item))
        .filter(Boolean);
      const secids = stockList.map((item) => item.secid).join(",");
      if (!secids) {
        this.stockDataList = [];
        this.stockDataListDft = [];
        return;
      }

      const url =
        "https://push2.eastmoney.com/api/qt/ulist.np/get?fltt=2&fields=f2,f3,f4,f12,f13,f14,f18,f124&secids=" +
        secids +
        "&_=" +
        new Date().getTime();

      this.$axios
        .get(url)
        .then((res) => {
          const quoteList = (res.data.data && res.data.data.diff) || [];
          const quoteMap = Object.fromEntries(
            quoteList.map((item) => [`${item.f13}.${item.f12}`, item])
          );
          const stockDataList = stockList.map((item) => {
            const quote = quoteMap[item.secid] || {};
            const latestPrice = this.toNullableNumber(quote.f2);
            const prevClose =
              this.toNullableNumber(quote.f18) ?? latestPrice ?? 0;
            const displayPrice = latestPrice === null ? prevClose : latestPrice;
            const changeRate =
              this.toNullableNumber(quote.f3) ??
              (prevClose
                ? (((displayPrice - prevClose) / prevClose) * 100).toFixed(2)
                : 0);
            const num = this.toNullableNumber(item.num) || 0;
            const cost = this.toNullableNumber(item.cost) || 0;

            return {
              stockcode: item.code,
              secid: item.secid,
              name: quote.f14 || item.name || item.code,
              latestPrice: this.formatStockPrice(displayPrice),
              latestPriceValue: displayPrice,
              dwjz: prevClose,
              gsz: displayPrice,
              gszzl: Number(changeRate).toFixed(2),
              gztime: quote.f124 || null,
              num,
              cost,
              amount: this.calculateMoney({ dwjz: displayPrice, num }),
              gains: this.calculateStockDailyGain(displayPrice, prevClose, num),
              costGains: this.calculateStockCostGain(displayPrice, cost, num),
              costGainsRate: this.calculateStockCostGainRate(displayPrice, cost),
            };
          });

          this.stockDataListDft = [...stockDataList];
          if (this.stockSortTypeObj.type && this.stockSortTypeObj.type != "none") {
            this.stockSortType[this.stockSortTypeObj.name] = this.stockSortTypeObj.type;
            this.stockDataList = [...stockDataList].sort(
              this.compare(this.stockSortTypeObj.name, this.stockSortTypeObj.type)
            );
          } else {
            this.stockDataList = [...stockDataList];
          }
        })
        .catch(() => {
          this.stockDataList = [];
          this.stockDataListDft = [];
        });
    },
    getData(type) {
      let fundlist = this.fundListM.map((val) => val.code).join(",");
      let url =
        "https://fundmobapi.eastmoney.com/FundMNewApi/FundMNFInfo?pageIndex=1&pageSize=200&plat=Android&appType=ttjj&product=EFund&Version=1&deviceid=" +
        this.userId +
        "&Fcodes=" +
        fundlist;
      this.$axios
        .get(url)
        .then(async (res) => {
          this.loadingList = false;
          let data = res.data.Datas || [];
          this.dataList = [];
          let dataList = [];
          let missingEstimateCodes = [];

          data.forEach((val) => {
            const nav = this.toNullableNumber(val.NAV);
            const estimatedNav = this.toNullableNumber(val.GSZ);
            const estimatedRate = this.toNullableNumber(val.GSZZL);
            const actualRate = this.toNullableNumber(val.NAVCHGRT);
            const hasRealtimeValue =
              !!val.GZTIME && val.PDATE != "--" && val.PDATE == val.GZTIME.substr(0, 10);
            const hasDailyRate = actualRate !== null && val.PDATE != "--";
            let data = {
              fundcode: val.FCODE,
              name: val.SHORTNAME,
              jzrq: val.PDATE,
              dwjz: nav,
              gsz: estimatedNav,
              gszzl: estimatedRate,
              gztime: val.GZTIME || val.PDATE || "--",
            };
            if (hasRealtimeValue) {
              data.gsz = nav;
              data.gszzl = actualRate;
              data.hasReplace = true;
            } else if (data.gsz === null && hasDailyRate) {
              data.gsz = nav;
              data.gszzl = actualRate;
              data.hasReplace = true;
            }

            if (estimatedNav === null) {
              missingEstimateCodes.push(data.fundcode);
            }

            if (data.gszzl === null) {
              data.gszzl = 0;
            }

            let slt = this.fundListM.filter(
              (item) => item.code == data.fundcode
            );
            data.num = slt[0].num;
            data.cost = slt[0].cost;
            data.accountType = slt[0].accountType || '';
            data.amount = this.calculateMoney(data);
            data.gains = this.calculate(data, data.hasReplace);
            data.costGains = this.calculateCost(data);
            data.costGainsRate = this.calculateCostRate(data);

            if (data.fundcode == this.RealtimeFundcode) {
              if (this.showBadge == 1) {
                if (this.BadgeContent == 1) {
                  chrome.runtime.sendMessage({
                    type: "refreshBadge",
                    data: data,
                  });
                }
              }
            }

            dataList.push(data);
          });

          if (missingEstimateCodes.length) {
            const estimateEntries = await Promise.all(
              missingEstimateCodes.map(async (code) => [
                code,
                await this.fetchFundEstimate(code),
              ])
            );
            const estimateMap = Object.fromEntries(estimateEntries);
            dataList = dataList.map((item) =>
              this.applyFundEstimate(item, estimateMap[item.fundcode])
            );
            dataList = dataList.map((item) => ({
              ...item,
              gains: this.calculate(item, item.hasReplace),
            }));
          }

          if (this.showBadge == 1) {
            if (this.BadgeContent == 2) {
              chrome.runtime.sendMessage({
                type: "refreshBadgeAllGains",
                data: dataList,
              });
            }
          }

          this.dataListDft = [...dataList];
          if (type == "add") {
            this.dataList = dataList;
          } else if (this.sortTypeObj.type != "none") {
            this.sortType[this.sortTypeObj.name] = this.sortTypeObj.type;
            this.dataList = [...dataList].sort(
              this.compare(this.sortTypeObj.name, this.sortTypeObj.type)
            );
          } else {
            this.dataList = dataList;
          }
        })
        .catch((error) => {});
    },
    formatUpdateTime(item) {
      if (!item || !item.gztime || item.gztime === "--") {
        return "--";
      }

      if (item.hasReplace) {
        return item.gztime.length >= 10 ? item.gztime.substr(5, 5) : item.gztime;
      }

      return item.gztime.length > 10 ? item.gztime.substr(10) : item.gztime;
    },
    changeNum(item, ind) {
      debounce(() => {
        for (let fund of this.fundListM) {
          if (fund.code == item.fundcode) {
            fund.num = item.num;
          }
        }
        chrome.storage.sync.set(
          {
            fundListM: this.fundListM,
          },
          () => {
            item.amount = this.calculateMoney(item);
            item.gains = this.calculate(item, item.hasReplace);
            item.costGains = this.calculateCost(item);
            chrome.runtime.sendMessage({ type: "refresh" });
          }
        );
      });
    },
    changeCost(item, ind) {
      debounce(() => {
        for (let fund of this.fundListM) {
          if (fund.code == item.fundcode) {
            fund.cost = item.cost;
          }
        }
        chrome.storage.sync.set(
          {
            fundListM: this.fundListM,
          },
          () => {
            item.costGains = this.calculateCost(item);
            item.costGainsRate = this.calculateCostRate(item);
          }
        );
      });
    },
    changeStockNum(item, ind) {
      debounce(() => {
        for (let stock of this.stockListM) {
          if (stock.code == item.stockcode) {
            stock.num = item.num;
          }
        }

        chrome.storage.sync.set(
          {
            stockListM: this.stockListM,
          },
          () => {
            item.amount = this.calculateMoney({ dwjz: item.gsz, num: item.num });
            item.gains = this.calculateStockDailyGain(item.gsz, item.dwjz, item.num);
            item.costGains = this.calculateStockCostGain(item.gsz, item.cost, item.num);
            item.costGainsRate = this.calculateStockCostGainRate(item.gsz, item.cost);
          }
        );
      });
    },
    changeStockCost(item, ind) {
      debounce(() => {
        for (let stock of this.stockListM) {
          if (stock.code == item.stockcode) {
            stock.cost = item.cost;
          }
        }

        chrome.storage.sync.set(
          {
            stockListM: this.stockListM,
          },
          () => {
            item.costGains = this.calculateStockCostGain(item.gsz, item.cost, item.num);
            item.costGainsRate = this.calculateStockCostGainRate(item.gsz, item.cost);
          }
        );
      });
    },
    calculateMoney(val) {
      if (val.dwjz === null || val.dwjz === undefined) {
        return 0;
      }
      let sum = (val.dwjz * val.num).toFixed(2);
      return sum;
    },
    calculate(val, hasReplace) {
      var sum = 0;
      let num = val.num ? val.num : 0;
      if (hasReplace) {
        sum = ((val.dwjz - val.dwjz / (1 + val.gszzl * 0.01)) * num).toFixed(2);
      } else {
        if (val.gsz != null) {
          sum = ((val.gsz - val.dwjz) * num).toFixed(2);
        }
      }
      return sum;
    },
    calculateCost(val) {
      if (val.cost) {
        let sum = ((val.dwjz - val.cost) * val.num).toFixed(2);
        return sum;
      } else {
        return 0;
      }
    },
    calculateCostRate(val) {
      if (val.cost && val.cost != 0) {
        let sum = (((val.dwjz - val.cost) / val.cost) * 100).toFixed(2);
        return sum;
      } else {
        return 0;
      }
    },
    save() {
      this.fundcode.forEach((code) => {
        let val = {
          code: code,
          num: 0,
        };
        this.fundListM.push(val);
      });

      chrome.storage.sync.set(
        {
          fundListM: this.fundListM,
        },
        () => {
          this.fundcode = [];
          this.getData("add");
          chrome.runtime.sendMessage({ type: "refresh" });
        }
      );
    },
    sltInd(val) {
      let code = val.f13 + "." + val.f12;
      if (code == this.RealtimeIndcode) {
        chrome.storage.sync.set(
          {
            RealtimeIndcode: null,
          },
          () => {
            this.RealtimeIndcode = null;
            chrome.runtime.sendMessage({ type: "endInterval" });
          }
        );
      } else {
        chrome.storage.sync.set(
          {
            RealtimeIndcode: code,
          },
          () => {
            this.RealtimeIndcode = code;
            chrome.runtime.sendMessage({ type: "refresh" });
          }
        );
      }
    },
    slt(id) {
      if (id == this.RealtimeFundcode) {
        chrome.storage.sync.set(
          {
            RealtimeFundcode: null,
          },
          () => {
            this.RealtimeFundcode = null;
            chrome.runtime.sendMessage({ type: "endInterval" });
          }
        );
      } else {
        chrome.storage.sync.set(
          {
            RealtimeFundcode: id,
          },
          () => {
            this.RealtimeFundcode = id;
            chrome.runtime.sendMessage({ type: "refresh" });
          }
        );
      }
    },
    dlt(id) {
      this.fundListM = this.fundListM.filter(function(ele) {
        return ele.code != id;
      });

      if (id == this.RealtimeFundcode) {
        chrome.storage.sync.set(
          {
            RealtimeFundcode: null,
          },
          () => {
            this.RealtimeFundcode = null;
            if (this.BadgeContent == 1) {
              chrome.runtime.sendMessage({ type: "endInterval" });
            }
          }
        );
      }
      chrome.storage.sync.set(
        {
          fundListM: this.fundListM,
        },
        () => {
          this.dataList = this.dataList.filter(function(ele) {
            return ele.fundcode != id;
          });
          if (this.BadgeContent == 2) {
            chrome.runtime.sendMessage({ type: "refresh" });
          }
        }
      );
    },
    dltStock(id) {
      this.stockListM = this.stockListM.filter(function(ele) {
        return ele.code != id;
      });

      chrome.storage.sync.set(
        {
          stockListM: this.stockListM,
        },
        () => {
          this.stockDataList = this.stockDataList.filter(function(ele) {
            return ele.stockcode != id;
          });
          this.stockDataListDft = this.stockDataListDft.filter(function(ele) {
            return ele.stockcode != id;
          });
        }
      );
    },
    handleDragStart(e, item) {
      this.dragging = item;
    },
    handleDragOver(e) {
      e.dataTransfer.dropEffect = "move";
    },
    handleDragEnd(e, item) {
      this.dragging = null;
      if (item.fundcode) {
        chrome.storage.sync.set(
          {
            fundListM: this.fundListM,
          },
          () => {}
        );
      } else if (item.f12) {
        chrome.storage.sync.set(
          {
            seciList: this.seciList,
          },
          () => {}
        );
      }
    },
    handleDragEnter(e, item, index) {
      // 基金排序
      if (this.dragging && this.dragging.fundcode && item.fundcode) {
        e.dataTransfer.effectAllowed = "move";
        if (item.fundcode === this.dragging.fundcode) {
          return;
        }
        const newItems = [...this.fundListM];
        const src = newItems.findIndex((n) => n.code == this.dragging.fundcode);
        const dst = newItems.findIndex((n) => n.code == item.fundcode);
        // // 替换
        newItems.splice(dst, 0, ...newItems.splice(src, 1));

        this.fundListM = newItems;

        //数据列表也同步更新
        const newDataItems = [...this.dataList];
        const dataSrc = newDataItems.findIndex(
          (n) => n.fundcode == this.dragging.fundcode
        );
        const dataDst = newDataItems.findIndex(
          (n) => n.fundcode == item.fundcode
        );
        newDataItems.splice(dataDst, 0, ...newDataItems.splice(dataSrc, 1));
        this.dataList = newDataItems;
      } else if (this.dragging && this.dragging.f12 && item.f12) {
        e.dataTransfer.effectAllowed = "move";
        if (item.f12 === this.dragging.f12) {
          return;
        }
        const newIndItems = [...this.seciList];
        const indSrc = newIndItems.findIndex(
          (n) => n.split(".")[1] == this.dragging.f12
        );
        const indDst = newIndItems.findIndex(
          (n) => n.split(".")[1] == item.f12
        );
        newIndItems.splice(indDst, 0, ...newIndItems.splice(indSrc, 1));
        this.seciList = newIndItems;

        const newIndDataItems = [...this.indFundData];
        const indDataSrc = newIndDataItems.findIndex(
          (n) => n.f12 == this.dragging.f12
        );
        const indDataDst = newIndDataItems.findIndex((n) => n.f12 == item.f12);
        newIndDataItems.splice(
          indDataDst,
          0,
          ...newIndDataItems.splice(indDataSrc, 1)
        );
        this.indFundData = newIndDataItems;
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.container {
  --sun-surface: rgba(255, 255, 255, 0.86);
  --sun-surface-strong: rgba(255, 252, 246, 0.96);
  --sun-border: rgba(226, 198, 146, 0.42);
  --sun-shadow: 0 18px 40px rgba(201, 157, 77, 0.16);
  --sun-shadow-soft: 0 10px 24px rgba(201, 157, 77, 0.12);
  --sun-text: #4f4334;
  --sun-muted: #8d7a60;
  min-width: 0;
  width: 100%;
  min-height: 680px;
  overflow-x: hidden;
  overflow-y: auto;
  padding: 16px 14px 18px;
  box-sizing: border-box;
  position: relative;
  font-size: 12px;
  font-family: "Helvetica Neue", Helvetica, "PingFang SC", "Hiragino Sans GB",
    "Microsoft YaHei", "微软雅黑", Arial, sans-serif;
  color: var(--sun-text);
  background:
    radial-gradient(circle at top left, rgba(255, 213, 111, 0.28), transparent 24%),
    radial-gradient(circle at top right, rgba(130, 208, 255, 0.22), transparent 28%),
    linear-gradient(180deg, #fffdf8 0%, #fff8ee 52%, #fff5ea 100%);
  border: 1px solid rgba(232, 211, 177, 0.78);
  border-radius: 24px;
  box-shadow: 0 22px 48px rgba(168, 131, 58, 0.18);
}

.refresh {
  position: absolute;
  right: 16px;
  width: 18px;
  bottom: 16px;
  z-index: 6;

  cursor: pointer;
  i {
    color: #409eff;
    font-size: 18px;
    font-weight: bold;
  }
}

.holdings-tabs-wrap {
  margin-top: 16px;
  min-height: 420px;
  min-width: 0;
  padding: 12px 14px 16px;
  border-radius: 22px;
  border: 1px solid var(--sun-border);
  background: var(--sun-surface);
  box-shadow: var(--sun-shadow-soft);
  backdrop-filter: blur(10px);
}

.page-tabs {
  min-height: 620px;
  min-width: 0;
}

.home-pane {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.stock-table-row {
  min-height: 420px;
}

.refresh.isRefresh {
  animation: changDeg 1.5s linear 0s infinite;
}

@keyframes changDeg {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(-360deg);
  }
}

.more-height {
  min-height: 720px;
}

.more-width {
  min-width: 0;
}

.stock-more-width {
  min-width: 0;
}

.changelog-container {
  min-height: 575px;
  min-width: 550px;
}

.table-more-height {
  min-height: 220px;
}
.table-drag {
  cursor: move;
}

.container {
  &.num-width-1 {
    min-width: 0;
  }
  &.num-width-2 {
    min-width: 0;
  }
  &.num-width-3 {
    min-width: 0;
  }
  &.num-width-4 {
    min-width: 0;
  }
  &.num-width-5 {
    min-width: 0;
  }
}

.table-row {
  min-height: 420px;
  max-height: 520px;
  position: relative;
  overflow-y: auto;
  padding-top: 6px;
}

.empty-row {
  text-align: center;
  color: #999;
  padding: 16px 8px;
}

.hasReplace {
  background-color: #409eff;
}

.hasReplace-tip {
  display: inline-block;
  padding: 0 2px;
  margin-right: 2px;
  border-radius: 2px;
  line-height: 12px;
  color: #409eff;
  border: 1px solid #409eff;
}

table {
  margin: 0 auto;
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  text-align: right;
  overflow: hidden;
  border-radius: 18px;
  background: var(--sun-surface-strong);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.9);
}
.align-left {
  text-align: left;
}

.center {
  text-align: center;
}

table th {
  padding: 12px 10px;
  white-space: nowrap;
  color: #6c5a44;
  font-weight: 700;
}

thead th {
  position: sticky;
  top: 0;
  z-index: 2;
  background: linear-gradient(180deg, #fffdf8 0%, #fff1dc 100%);
  box-shadow: 0 1px 0 rgba(223, 198, 162, 0.95);
}

table td {
  padding: 10px 10px 9px;
  white-space: nowrap;
  color: var(--sun-text);
  border-bottom: 1px solid rgba(244, 234, 218, 0.95);
}

.up {
  color: #f56c6c;
  font-weight: bold;
}

.down {
  color: #4eb61b;
  font-weight: bold;
}

tbody tr:hover {
  background: linear-gradient(90deg, rgba(255, 244, 220, 0.74), rgba(255, 251, 243, 0.96));
}

tbody tr:last-child td,
tbody tr:last-child th {
  border-bottom: none;
}

.btn {
  display: inline-block;
  line-height: 1;
  cursor: pointer;
  background: rgba(255, 255, 255, 0.9);
  padding: 8px 12px;
  border-radius: 12px;
  font-size: 12px;
  color: var(--sun-text);
  margin: 0 4px;
  outline: none;
  border: 1px solid rgba(225, 202, 164, 0.84);
  box-shadow: 0 6px 14px rgba(201, 157, 77, 0.1);
  transition: transform 0.18s ease, box-shadow 0.18s ease, border-color 0.18s ease;
}

.btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px rgba(201, 157, 77, 0.14);
}

.btn.edit {
  padding: 2px 5px;
  margin: 0;
}

.btn.red {
  color: #f56c6c;
}

.btn.num {
  width: 75px;
  padding: 3px 6px;
}

.btn-up {
  color: #f56c6c;
  border-color: #f56c6c;
  background: linear-gradient(180deg, #fff9f9 0%, #ffe8e8 100%);
  font-weight: bold;
}

.btn-down {
  color: #4eb61b;
  border-color: #4eb61b;
  background: linear-gradient(180deg, #fbfff7 0%, #effbdd 100%);
  font-weight: bold;
}

.btn-total-amount {
  color: #c75b5b;
  border-color: #f0b3b3;
  background-color: #fde9e9;
  font-weight: bold;
}

.slt {
  color: #fff;
  background-color: #67c23a;
  border-color: #67c23a;
}

.input-row {
  text-align: center;
  margin-top: 10px;
}

.summary-panel {
  margin: 10px 0 4px;
  padding: 10px;
  border-radius: 18px;
  border: 1px solid var(--sun-border);
  background: var(--sun-surface);
  box-shadow: var(--sun-shadow);
  backdrop-filter: blur(10px);
}

.summary-row {
  margin-top: 8px;
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 8px;
}

.summary-row:first-child {
  margin-top: 0;
}

.summary-row-home {
  grid-template-columns: repeat(3, minmax(0, 1fr));
}

.tab-summary-panel {
  margin: 0 0 8px;
  padding: 8px 10px;
}

.tab-summary-row {
  grid-template-columns: repeat(3, minmax(0, 1fr));
}

.summary-row .btn {
  width: 100%;
  min-width: 0;
  margin: 0;
  min-height: 52px;
  padding: 12px 14px;
  border-radius: 14px;
  font-size: 15px;
  font-weight: 700;
}

.summary-card {
  min-width: 0;
  min-height: 0;
  padding: 8px 12px;
  border-radius: 12px;
  border: 1px solid rgba(225, 202, 164, 0.84);
  background: rgba(255, 255, 255, 0.92);
  box-shadow: 0 6px 14px rgba(201, 157, 77, 0.08);
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 3px;
  text-align: left;
}

.summary-card-soft {
  border-style: dashed;
}

.summary-card-green {
  border-color: rgba(92, 180, 46, 0.88);
  background: linear-gradient(180deg, rgba(248, 255, 242, 0.96) 0%, rgba(238, 251, 221, 0.94) 100%);
  color: #43a61d;
}

.summary-card-red {
  border-color: rgba(245, 108, 108, 0.72);
  background: linear-gradient(180deg, rgba(255, 249, 249, 0.96) 0%, rgba(255, 236, 236, 0.94) 100%);
  color: #de6d6d;
}

.summary-label {
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.02em;
}

.summary-value {
  font-size: 15px;
  line-height: 1.1;
  font-weight: 700;
  white-space: nowrap;
}

.summary-meta {
  font-size: 11px;
  font-weight: 600;
  opacity: 0.9;
}

.action-row {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
  margin-top: 14px;
}

.action-row .btn {
  margin: 0;
  min-width: 86px;
  border-radius: 999px;
  padding: 9px 14px;
}

.market-overview {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
  margin: 0;
  padding: 0;
}

.market-overview .tab-col {
  min-width: 0;
  margin: 0;
  padding: 14px 12px 12px;
  border-radius: 20px;
  border: 1px solid var(--sun-border);
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.94) 0%, rgba(255, 248, 235, 0.9) 100%);
  box-shadow: var(--sun-shadow-soft);
}

.market-overview .tab-col h5 {
  margin: 0 0 10px;
  font-size: 14px;
  font-weight: 700;
  color: #6d5a43;
}

.market-overview .tab-col p {
  margin: 4px 0;
  font-size: 13px;
  font-weight: 700;
}
.gear-input-row {
  display: flex;
  justify-content: center;
  align-items: center;
  .slider-title {
    font-size: 14px;
    margin: 0 5px 0 15px;
  }
  .slider {
    display: inline-block;
    width: 20%;
  }
}

::v-deep .el-tabs__header {
  margin: 0 0 14px;
}

::v-deep .page-tabs > .el-tabs__header {
  background: rgba(255, 249, 239, 0.9);
  border: 1px solid rgba(232, 210, 179, 0.72);
  border-radius: 18px;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.9);
  padding: 6px;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__nav-wrap::after {
  display: none;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__nav {
  display: flex;
  gap: 8px;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__item {
  width: auto;
  min-width: 120px;
  height: 44px;
  line-height: 44px;
  text-align: center;
  padding: 0 16px;
  font-size: 14px;
  border-radius: 14px;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__active-bar {
  height: 3px;
  width: auto;
  border-radius: 999px;
}

::v-deep .page-tabs > .el-tabs__content {
  overflow: visible;
  min-width: 0;
}

::v-deep .el-tabs__item {
  height: 34px;
  line-height: 34px;
  font-size: 14px;
  font-weight: 700;
  color: #8a7457;
}

::v-deep .el-tabs__item.is-active {
  color: #d08c1f;
}

::v-deep .el-tabs__active-bar {
  height: 3px;
  border-radius: 999px;
  background: linear-gradient(90deg, #f6bf53 0%, #ee9f55 100%);
}

::v-deep .el-tabs__nav-wrap::after {
  background-color: rgba(232, 210, 179, 0.9);
}

::v-deep .el-tabs__content {
  min-height: 420px;
}

.btn-subtotal {
  border-style: dashed;
  color: #c75b5b;
  border-color: #f0b3b3;
  background-color: #fde9e9;
  font-weight: bold;
}

.tab-col {
  flex: 1;
  margin: 0 4px;
  text-align: center;
  h5 {
    margin: 4px 0;
    font-size: 12px;
    .dltBtn {
      margin-left: 3px;
    }
  }
  p {
    margin: 4px 0;
  }
  .addSeci {
    margin: 10px auto;
    width: 40px;
    height: 40px;
    cursor: pointer;
    line-height: 40px;
    border: 1px solid #dcdfe6;
    border-radius: 50%;
  }
}
.indFund {
  cursor: pointer;
}

.tab-row:after,
.tab-row:before {
  display: table;
  content: "";
}

.tab-row:after {
  clear: both;
}

.tab-row {
  padding: 6px 0;
  display: flex;
  margin: 0 -3px;
}

.primary {
  color: #409eff;
  border-color: #409eff;
}

.tips {
  font-size: 12px;
  margin: 0;
  color: var(--sun-muted);
  line-height: 1.4;
  padding: 6px 15px;
}

.fundName {
  max-width: 180px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  cursor: pointer;
  user-select: none;
}

.fundName-noclick {
  max-width: 180px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fundName:hover {
  color: #d08c1f;
}

.stockNameCell {
  max-width: 300px;
}

.down-arrow {
  display: inline-block;
  position: relative;
  width: 8px;
  height: 0;
}

.down-arrow::after {
  display: inline-block;
  content: " ";
  height: 6px;
  width: 6px;
  border-width: 0 1px 1px 0;
  border-color: #666;
  border-style: solid;
  transform-origin: center;
  transition: all 0.3s;
  position: absolute;
  right: 0;
}
.down-arrow.desc::after {
  transform-origin: center;
  transform: rotate(45deg);
  top: -10px;
}
.down-arrow.asc::after {
  transform-origin: center;
  transform: rotate(-135deg);
  top: -6px;
}

.down-arrow.none {
  display: none;
}

.pointer {
  cursor: pointer;
  user-select: none;
}

//标准字号
.normalFontSize {
  min-width: 450px;
  font-size: 14px;
  &.num-width-1 {
    min-width: 0;
  }
  &.num-width-2 {
    min-width: 0;
  }
  &.num-width-3 {
    min-width: 0;
  }
  &.num-width-4 {
    min-width: 0;
  }
  &.num-width-5 {
    min-width: 0;
  }

  &.stock-more-width {
    min-width: 0;
  }

  .btn,
  .tips {
    font-size: 14px;
  }
  .tab-col {
    h5 {
      font-size: 14px;
    }
  }
}

.detail-container {
  min-height: 720px;
  min-width: 610px;
}

.detailTable {
  th,
  td {
    p {
      margin: 2px 0;
      color: rgba($color: #000000, $alpha: 0.6);
    }
  }
}

//暗黑主题
.container.darkMode {
  color: rgba($color: #ffffff, $alpha: 0.6);
  background-color: #121212;
  background-image: none;
  border-color: rgba($color: #ffffff, $alpha: 0.08);
  box-shadow: none;
  thead th {
    background-color: #1e1e1e;
    background-image: none;
    box-shadow: 0 1px 0 rgba(255, 255, 255, 0.1);
    color: rgba($color: #ffffff, $alpha: 0.5);
  }
  .summary-panel {
    background: rgba($color: #ffffff, $alpha: 0.06);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.08);
    box-shadow: none;
  }
  .summary-card {
    background: rgba($color: #ffffff, $alpha: 0.08);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.1);
    box-shadow: none;
  }
  .summary-card-green {
    color: #79d85f;
  }
  .summary-card-red {
    color: #ff9b9b;
  }
  .holdings-tabs-wrap,
  .market-overview .tab-col {
    background: rgba($color: #ffffff, $alpha: 0.06);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.08);
    box-shadow: none;
  }
  ::v-deep .page-tabs > .el-tabs__header {
    background: rgba($color: #ffffff, $alpha: 0.06);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.08);
  }
  .refresh {
    color: rgba($color: #409eff, $alpha: 0.6);
  }
  .btn {
    background-color: rgba($color: #ffffff, $alpha: 0.16);
    color: rgba($color: #ffffff, $alpha: 0.6);
    border: 1px solid rgba($color: #ffffff, $alpha: 0.6);
  }
  .primary {
    border: 1px solid rgba($color: #409eff, $alpha: 0.6);
    background-color: rgba($color: #409eff, $alpha: 0.6);
  }
  .action-row .btn {
    background-color: rgba($color: #ffffff, $alpha: 0.12);
  }
  ::v-deep .el-input__inner {
    background-color: rgba($color: #ffffff, $alpha: 0.16);
    color: rgba($color: #ffffff, $alpha: 0.6);
  }
  ::v-deep .el-select__input {
    color: rgba($color: #ffffff, $alpha: 0.6);
  }

  ::v-deep tbody tr:hover {
    background: rgba($color: #ffffff, $alpha: 0.08) !important;
  }

  tbody tr:hover {
    background: rgba($color: #ffffff, $alpha: 0.08) !important;
  }

  .slt {
    border: 1px solid rgba($color: #67c23a, $alpha: 0.6);
    background-color: rgba($color: #67c23a, $alpha: 0.6);
  }

  .btn.red {
    border: 1px solid rgba($color: #f56c6c, $alpha: 0.6);
    background-color: rgba($color: #f56c6c, $alpha: 0.6);
  }

  .btn-up {
    border: 1px solid rgba($color: #f56c6c, $alpha: 0.6);
    background-color: rgba($color: #f56c6c, $alpha: 0.6);
  }

  .btn-down {
    border: 1px solid rgba($color: #4eb61b, $alpha: 0.6);
    background-color: rgba($color: #4eb61b, $alpha: 0.6);
  }

  .btn-total-amount {
    color: rgba($color: #ffd4d4, $alpha: 0.95);
    border: 1px solid rgba($color: #f0b3b3, $alpha: 0.6);
    background-color: rgba($color: #c75b5b, $alpha: 0.26);
  }

  .btn-subtotal {
    color: rgba($color: #ffd4d4, $alpha: 0.95);
    border: 1px solid rgba($color: #f0b3b3, $alpha: 0.6);
    background-color: rgba($color: #c75b5b, $alpha: 0.26);
  }

  .tab-col {
    background-color: rgba($color: #ffffff, $alpha: 0.09);
    border-radius: 5px;
  }

  table {
    background-color: rgba($color: #ffffff, $alpha: 0.12);
    border-radius: 5px;
  }

  table td {
    border-bottom-color: rgba($color: #ffffff, $alpha: 0.06);
    color: rgba($color: #ffffff, $alpha: 0.6);
  }

  table th {
    color: rgba($color: #ffffff, $alpha: 0.5);
  }

  ::v-deep .el-tabs__item {
    color: rgba($color: #ffffff, $alpha: 0.5);
  }

  ::v-deep .el-tabs__item.is-active {
    color: rgba($color: #ffd37b, $alpha: 0.92);
  }

  ::v-deep .el-tabs__active-bar {
    background: linear-gradient(90deg, rgba(255, 209, 112, 0.9), rgba(255, 160, 122, 0.88));
  }

  ::v-deep .el-tabs__nav-wrap::after {
    background-color: rgba($color: #ffffff, $alpha: 0.08);
  }

  ::placeholder {
    color: rgba($color: #ffffff, $alpha: 0.38);
  }

  ::v-deep .el-select .el-input.is-focus .el-input__inner {
    border-color: rgba($color: #409eff, $alpha: 0.6);
  }

  ::v-deep .el-select .el-tag {
    background-color: rgba($color: #ffffff, $alpha: 0.14);
    color: rgba($color: #ffffff, $alpha: 0.6);
  }

  ::v-deep .el-select-dropdown {
    background-color: #383838;
    border: 1px solid rgba($color: #ffffff, $alpha: 0.38);
    .popper__arrow::after {
      border-bottom-color: #383838;
    }
    .el-scrollbar {
      background-color: rgba($color: #ffffff, $alpha: 0.16);
    }
    .el-select-dropdown__item {
      color: rgba($color: #ffffff, $alpha: 0.6);
    }

    .el-select-dropdown__item.hover,
    .el-select-dropdown__item:hover {
      background-color: rgba($color: #ffffff, $alpha: 0.08);
    }
    .el-select-dropdown__item.selected {
      color: rgba($color: #409eff, $alpha: 0.6);
      background-color: rgba($color: #ffffff, $alpha: 0.08);
    }
    .el-select-dropdown__item.selected::after {
      color: rgba($color: #409eff, $alpha: 0.6);
    }
  }

  ::v-deep .el-switch__label.is-active {
    color: rgba($color: #409eff, $alpha: 0.87);
  }
  ::v-deep .el-switch__label {
    color: rgba($color: #ffffff, $alpha: 0.6);
  }

  ::v-deep .hasReplace-tip {
    color: rgba($color: #ffffff, $alpha: 0.6);
    border: 1px solid rgba($color: #409eff, $alpha: 0.6);
    background-color: rgba($color: #409eff, $alpha: 0.6);
  }

  .fundName:hover {
    color: rgba($color: #ffd37b, $alpha: 0.92);
  }
}
</style>
