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
              <table class="summary-table" v-if="showCost || showGains || showAmount">
                <thead>
                  <tr>
                    <th>类别</th>
                    <th v-if="showAmount">持仓</th>
                    <th v-if="showGains">估算</th>
                    <th v-if="showCost">持有收益</th>
                  </tr>
                </thead>
                <tbody>
                  <tr class="summary-table-total">
                    <td>
                      <span class="tree-root-node">
                        <svg width="14" height="14" viewBox="0 0 14 14" style="margin-right:5px;vertical-align:-2px"><rect x="1" y="1" width="12" height="12" rx="3" fill="#6366f1" opacity="0.15"/><rect x="3" y="3" width="8" height="8" rx="2" fill="#6366f1"/></svg>
                        总计
                      </span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(allAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showGains" :class="allGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(allGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(allGains[1]) ? '' : ' ' + allGains[1] + '%' }}</span></td>
                    <td v-if="showCost" :class="allCostGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(allCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(allCostGains[1]) ? '' : ' ' + allCostGains[1] + '%' }}</span></td>
                  </tr>
                  <tr v-if="dataList.length" class="tree-level-1 tree-mid">
                    <td>
                      <span class="tree-node">
                        <span class="tree-dot" style="background:#f59e0b"></span>
                        基金
                      </span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(fundAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showGains" :class="fundGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(fundGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(fundGains[1]) ? '' : ' ' + fundGains[1] + '%' }}</span></td>
                    <td v-if="showCost" :class="fundCostGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(fundCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(fundCostGains[1]) ? '' : ' ' + fundCostGains[1] + '%' }}</span></td>
                  </tr>
                  <tr v-if="freeDataList.length" :class="['tree-level-2', safeDataList.length ? 'tree-mid' : 'tree-last']">
                    <td>
                      <span class="tree-node">
                        <span class="tree-dot" style="background:#f59e0b;opacity:0.5"></span>
                        自由
                      </span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(freeAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showGains" :class="freeGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(freeGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(freeGains[1]) ? '' : ' ' + freeGains[1] + '%' }}</span></td>
                    <td v-if="showCost" :class="freeCostGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(freeCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(freeCostGains[1]) ? '' : ' ' + freeCostGains[1] + '%' }}</span></td>
                  </tr>
                  <tr v-if="safeDataList.length" class="tree-level-2 tree-last">
                    <td>
                      <span class="tree-node">
                        <span class="tree-dot" style="background:#f59e0b;opacity:0.4"></span>
                        安全
                      </span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(safeAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showGains" :class="safeGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(safeGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(safeGains[1]) ? '' : ' ' + safeGains[1] + '%' }}</span></td>
                    <td v-if="showCost" :class="safeCostGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(safeCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(safeCostGains[1]) ? '' : ' ' + safeCostGains[1] + '%' }}</span></td>
                  </tr>
                  <tr v-if="stockDataList.length" :class="['tree-level-1', bankList.length ? 'tree-mid' : 'tree-last']">
                    <td>
                      <span class="tree-node">
                        <span class="tree-dot" style="background:#10b981"></span>
                        股票
                      </span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(stockAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showGains" :class="stockGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(stockGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(stockGains[1]) ? '' : ' ' + stockGains[1] + '%' }}</span></td>
                    <td v-if="showCost" :class="stockCostGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(stockCostGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(stockCostGains[1]) ? '' : ' ' + stockCostGains[1] + '%' }}</span></td>
                  </tr>
                  <tr v-if="bankList.length" class="tree-level-1 tree-last">
                    <td>
                      <span class="tree-node">
                        <span class="tree-dot" style="background:#3b82f6"></span>
                        现金
                      </span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(bankTotalAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showGains">{{ parseFloat(bankPrincipal).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct"> 本金</span></td>
                    <td v-if="showCost" :class="bankGains[0] >= 0 ? 'up' : 'down'">{{ parseFloat(bankGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}<span class="pct">{{ isNaN(bankGains[1]) ? '' : ' ' + bankGains[1] + '%' }}</span></td>
                  </tr>
                </tbody>
              </table>
              <asset-pie v-if="pieChartData.length > 0 && showAmount" :valData="pieChartData" :darkMode="darkMode"></asset-pie>
            </div>
          </el-tab-pane>
          <el-tab-pane :label="`基金（${dataList.length}）`" name="fund">
            <div class="summary-panel tab-summary-panel" v-if="showCost || showGains || showAmount">
              <div class="summary-row tab-summary-row">
                <div v-if="showAmount" class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">基金持仓</span>
                  <strong class="summary-value">{{ parseFloat(fundAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" :class="['summary-card summary-card-compact', summaryGainCardClass(fundGains[0])]">
                  <span class="summary-label">基金估算</span>
                  <strong class="summary-value">{{ parseFloat(fundGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(fundGains[1]) ? '' : '（' + fundGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" :class="['summary-card summary-card-compact', summaryGainCardClass(fundCostGains[0])]">
                  <span class="summary-label">基金收益</span>
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
                    :class="[drag, el.isPrivate ? 'private-fund-row' : '']"
                    @dragstart="handleDragStart($event, el)"
                    @dragover.prevent="handleDragOver($event, el)"
                    @dragenter="handleDragEnter($event, el, index)"
                    @dragend="handleDragEnd($event, el)"
                    @dblclick="dblclickFund(el)"
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
                    <td v-if="showGSZ && !isEdit" @dblclick="editNav(el)">
                      <template v-if="editingNavCode === el.fundcode && el.isPrivate">
                        <input
                          class="btn num small-input"
                          v-model="el.gsz"
                          @blur="saveNav(el)"
                          @keyup.enter="saveNav(el)"
                          v-focus
                          type="text"
                        />
                      </template>
                      <template v-else>
                        {{ el.gsz }}
                      </template>
                    </td>
                    <td v-if="isEdit && (showCostRate || showCost)">
                      <input
                        class="btn num"
                        placeholder="持仓成本价"
                        v-model="el.cost"
                        @input="changeCost(el, index)"
                        type="text"
                      />
                    </td>
                    <td v-if="!isEdit && (showCostRate || showCost)">
                       <template v-if="editingFundCode === el.fundcode && el.isPrivate">
                        <input
                          class="btn num small-input"
                          v-model="el.cost"
                          @blur="saveFundEdit(el, index)"
                          @keyup.enter="saveFundEdit(el, index)"
                          v-focus
                          type="text"
                        />
                      </template>
                      <span v-else>{{ el.cost || '--' }}</span>
                    </td>

                    <td v-if="showAmount">
                      {{
                        parseFloat(el.amount).toLocaleString("zh", {
                          maximumFractionDigits: 0,
                        })
                      }}
                    </td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">
                      {{
                        parseFloat(el.costGains).toLocaleString("zh", {
                          maximumFractionDigits: 0,
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
                          maximumFractionDigits: 0,
                        })
                      }}
                    </td>
                    <td v-if="!isEdit">{{ formatUpdateTime(el) }}</td>
                    <th
                      style="text-align:center"
                      v-if="
                        (isEdit || (editingFundCode === el.fundcode && el.isPrivate)) &&
                          (showAmount || showGains || showCost || showCostRate)
                      "
                    >
                      <input
                        class="btn num"
                        v-model="el.num"
                        @input="isEdit && changeNum(el, index)"
                        @blur="!isEdit && saveFundEdit(el, index)"
                        @keyup.enter="!isEdit && saveFundEdit(el, index)"
                        placeholder="输入持有份额"
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
                <div v-if="showAmount" class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">自由持仓</span>
                  <strong class="summary-value">{{ parseFloat(freeAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" :class="['summary-card summary-card-compact', summaryGainCardClass(freeGains[0]) ]">
                  <span class="summary-label">自由估算</span>
                  <strong class="summary-value">{{ parseFloat(freeGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(freeGains[1]) ? '' : '（' + freeGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" :class="['summary-card summary-card-compact', summaryGainCardClass(freeCostGains[0])]">
                  <span class="summary-label">自由收益</span>
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
                    :class="[drag, el.isPrivate ? 'private-fund-row' : '']"
                    @dragstart="handleDragStart($event, el)"
                    @dragover.prevent="handleDragOver($event, el)"
                    @dragenter="handleDragEnter($event, el, index)"
                    @dragend="handleDragEnd($event, el)"
                    @dblclick="dblclickFund(el)"
                  >
                    <td :class="isEdit ? 'fundName-noclick align-left' : 'fundName align-left'" :title="el.name" @click.stop="!isEdit && fundDetail(el)">
                      <span class="hasReplace-tip" v-if="el.hasReplace">✔</span>{{ el.name }}
                    </td>
                    <td v-if="isEdit">{{ el.fundcode }}</td>
                    <td v-if="showGSZ && !isEdit" @dblclick="el.isPrivate && dblclickFund(el)">
                      <template v-if="editingFundCode === el.fundcode && el.isPrivate">
                        <input
                          class="btn num small-input"
                          v-model="el.gsz"
                          @blur="saveFundEdit(el, index)"
                          @keyup.enter="saveFundEdit(el, index)"
                          v-focus
                          type="text"
                        />
                      </template>
                      <template v-else>
                        {{ el.gsz }}
                      </template>
                    </td>
                    <td v-if="isEdit && (showCostRate || showCost)">
                      <input class="btn num" placeholder="持仓成本价" v-model="el.cost" @input="changeCost(el, index)" type="text" />
                    </td>
                    <td v-if="!isEdit && (showCostRate || showCost)">
                      <template v-if="editingFundCode === el.fundcode && el.isPrivate">
                        <input class="btn num small-input" v-model="el.cost" @blur="saveFundEdit(el, index)" @keyup.enter="saveFundEdit(el, index)" v-focus type="text" />
                      </template>
                      <span v-else>{{ el.cost || '--' }}</span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(el.amount).toLocaleString("zh", { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">{{ parseFloat(el.costGains).toLocaleString("zh", { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showCostRate" :class="el.costGainsRate >= 0 ? 'up' : 'down'">{{ el.cost > 0 ? el.costGainsRate + "%" : "" }}</td>
                    <td :class="el.gszzl >= 0 ? 'up' : 'down'">{{ el.gszzl }}%</td>
                    <td v-if="showGains" :class="el.gains >= 0 ? 'up' : 'down'">{{ parseFloat(el.gains).toLocaleString("zh", { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="!isEdit">{{ formatUpdateTime(el) }}</td>
                    <th style="text-align:center" v-if="(isEdit || (editingFundCode === el.fundcode && el.isPrivate)) && (showAmount || showGains || showCost || showCostRate)">
                      <input class="btn num" placeholder="输入持有份额" v-model="el.num" @input="isEdit && changeNum(el, index)" @blur="!isEdit && saveFundEdit(el, index)" @keyup.enter="!isEdit && saveFundEdit(el, index)" type="text" />
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
                <div v-if="showAmount" class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">安全持仓</span>
                  <strong class="summary-value">{{ parseFloat(safeAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" :class="['summary-card summary-card-compact', summaryGainCardClass(safeGains[0])]">
                  <span class="summary-label">安全估算</span>
                  <strong class="summary-value">{{ parseFloat(safeGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(safeGains[1]) ? '' : '（' + safeGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" :class="['summary-card summary-card-compact', summaryGainCardClass(safeCostGains[0])]">
                  <span class="summary-label">安全收益</span>
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
                    :class="[drag, el.isPrivate ? 'private-fund-row' : '']"
                    @dragstart="handleDragStart($event, el)"
                    @dragover.prevent="handleDragOver($event, el)"
                    @dragenter="handleDragEnter($event, el, index)"
                    @dragend="handleDragEnd($event, el)"
                    @dblclick="dblclickFund(el)"
                  >
                    <td :class="isEdit ? 'fundName-noclick align-left' : 'fundName align-left'" :title="el.name" @click.stop="!isEdit && fundDetail(el)">
                      <span class="hasReplace-tip" v-if="el.hasReplace">✔</span>{{ el.name }}
                    </td>
                    <td v-if="isEdit">{{ el.fundcode }}</td>
                    <td v-if="showGSZ && !isEdit" @dblclick="el.isPrivate && dblclickFund(el)">
                      <template v-if="editingFundCode === el.fundcode && el.isPrivate">
                        <input
                          class="btn num small-input"
                          v-model="el.gsz"
                          @blur="saveFundEdit(el, index)"
                          @keyup.enter="saveFundEdit(el, index)"
                          v-focus
                          type="text"
                        />
                      </template>
                      <template v-else>
                        {{ el.gsz }}
                      </template>
                    </td>
                    <td v-if="isEdit && (showCostRate || showCost)">
                      <input class="btn num" placeholder="持仓成本价" v-model="el.cost" @input="changeCost(el, index)" type="text" />
                    </td>
                    <td v-if="!isEdit && (showCostRate || showCost)">
                      <template v-if="editingFundCode === el.fundcode && el.isPrivate">
                        <input class="btn num small-input" v-model="el.cost" @blur="saveFundEdit(el, index)" @keyup.enter="saveFundEdit(el, index)" v-focus type="text" />
                      </template>
                      <span v-else>{{ el.cost || '--' }}</span>
                    </td>
                    <td v-if="showAmount">{{ parseFloat(el.amount).toLocaleString("zh", { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">{{ parseFloat(el.costGains).toLocaleString("zh", { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="showCostRate" :class="el.costGainsRate >= 0 ? 'up' : 'down'">{{ el.cost > 0 ? el.costGainsRate + "%" : "" }}</td>
                    <td :class="el.gszzl >= 0 ? 'up' : 'down'">{{ el.gszzl }}%</td>
                    <td v-if="showGains" :class="el.gains >= 0 ? 'up' : 'down'">{{ parseFloat(el.gains).toLocaleString("zh", { maximumFractionDigits: 0 }) }}</td>
                    <td v-if="!isEdit">{{ formatUpdateTime(el) }}</td>
                    <th style="text-align:center" v-if="(isEdit || editingFundCode === el.fundcode) && (showAmount || showGains || showCost || showCostRate)">
                      <input class="btn num" placeholder="输入持有份额" v-model="el.num" @input="isEdit && changeNum(el, index)" @blur="!isEdit && saveFundEdit(el, index)" @keyup.enter="!isEdit && saveFundEdit(el, index)" type="text" />
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
                <div v-if="showAmount" class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">股票持仓</span>
                  <strong class="summary-value">{{ parseFloat(stockAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div v-if="showGains" :class="['summary-card summary-card-compact', summaryGainCardClass(stockGains[0])]">
                  <span class="summary-label">股票估算</span>
                  <strong class="summary-value">{{ parseFloat(stockGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(stockGains[1]) ? '' : '（' + stockGains[1] + '%）' }}</span>
                </div>
                <div v-if="showCost" :class="['summary-card summary-card-compact', summaryGainCardClass(stockCostGains[0])]">
                  <span class="summary-label">股票收益</span>
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
                      <div v-if="isEdit">
                        <input
                          class="btn num"
                          placeholder="持仓成本价"
                          v-model="el.cost"
                          @input="changeStockCost(el, index)"
                          type="text"
                        />
                      </div>
                      <span v-else>{{ formatStockPrice(el.cost) }}</span>
                    </td>
                    <td v-if="showAmount">
                      {{
                        parseFloat(el.amount).toLocaleString("zh", {
                          maximumFractionDigits: 0,
                        })
                      }}
                    </td>
                    <td v-if="showCost" :class="el.costGains >= 0 ? 'up' : 'down'">
                      {{
                        parseFloat(el.costGains).toLocaleString("zh", {
                          maximumFractionDigits: 0,
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
                          maximumFractionDigits: 0,
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
                        v-model="el.num"
                        @input="changeStockNum(el, index)"
                        placeholder="输入持有股数"
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
          <el-tab-pane :label="`现金（${bankList.length}）`" name="bank">
            <div class="summary-panel tab-summary-panel" v-if="showAmount">
              <div class="summary-row tab-summary-row">
                <div class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">现金总额</span>
                  <strong class="summary-value">{{ parseFloat(bankTotalAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">现金本金</span>
                  <strong class="summary-value">{{ parseFloat(bankPrincipal).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
                <div :class="['summary-card summary-card-compact', summaryGainCardClass(bankGains[0])]">
                  <span class="summary-label">现金收益</span>
                  <strong class="summary-value">{{ parseFloat(bankGains[0]).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                  <span class="summary-meta">{{ isNaN(bankGains[1]) ? '' : '（' + bankGains[1] + '%）' }}</span>
                </div>
                <div class="summary-card summary-card-compact summary-card-red">
                  <span class="summary-label">年收入</span>
                  <strong class="summary-value">{{ parseFloat(bankAnnualIncome).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</strong>
                </div>
              </div>
            </div>
            <div class="table-row" style="min-height:160px">
              <table :class="tableHeight">
                <thead>
                  <tr>
                    <th class="align-left">银行名称（{{ bankList.length }}）</th>
                    <th @click="sortBankList('principal')" class="pointer">本金<span :class="bankSortType.principal" class="down-arrow"></span></th>
                    <th @click="sortBankList('totalAmount')" class="pointer">总额<span :class="bankSortType.totalAmount" class="down-arrow"></span></th>
                    <th @click="sortBankList('profit')" class="pointer">收益<span :class="bankSortType.profit" class="down-arrow"></span></th>
                    <th @click="sortBankList('annualIncome')" class="pointer">年收入<span :class="bankSortType.annualIncome" class="down-arrow"></span></th>
                    <th @click="sortBankList('annualRate')" class="pointer">年利率<span :class="bankSortType.annualRate" class="down-arrow"></span></th>
                    <th v-if="editingBankIndex >= 0">操作</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(el, index) in bankList" :key="index" @dblclick="dblclickBank(index)">
                    <template v-if="editingBankIndex === index">
                      <td class="align-left"><input class="btn num" v-model="el.bankName" type="text" /></td>
                      <td><input class="btn num" v-model="el.principal" type="text" /></td>
                      <td><input class="btn num" v-model="el.totalAmount" type="text" /></td>
                      <td :class="el.totalAmount - el.principal >= 0 ? 'up' : 'down'">{{ parseFloat((el.totalAmount - el.principal)).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                      <td><input class="btn num" v-model="el.annualIncome" type="text" /></td>
                      <td><input class="btn num" v-model="el.annualRate" type="text" /></td>
                      <td>
                        <input class="btn" type="button" value="保存" @click="saveBankEdit" />
                        <input class="btn red edit" type="button" value="✖" @click="dltBank(index)" />
                      </td>
                    </template>
                    <template v-else>
                      <td class="align-left">{{ el.bankName }}</td>
                      <td>{{ parseFloat(el.principal).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                      <td>{{ parseFloat(el.totalAmount).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                      <td :class="el.totalAmount - el.principal >= 0 ? 'up' : 'down'">{{ parseFloat((el.totalAmount - el.principal)).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                      <td>{{ parseFloat(el.annualIncome).toLocaleString('zh', { maximumFractionDigits: 0 }) }}</td>
                      <td>{{ el.annualRate }}%</td>
                      <td v-if="editingBankIndex >= 0"></td>
                    </template>
                  </tr>
                  <tr v-if="!bankList.length">
                    <td class="empty-row" :colspan="editingBankIndex >= 0 ? 7 : 6">暂无现金数据</td>
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
import assetPie from "../common/assetPie";
//防抖
let timeout = null;

function flattenFundListGroup(fundListGroup) {
  if (!Array.isArray(fundListGroup)) {
    return [];
  }

  const merged = [];
  fundListGroup.forEach((group) => {
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
    assetPie,
  },
  directives: {
    focus: {
      inserted: function(el) {
        el.focus();
      },
    },
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
      bankList: [],
      bankListDft: [],
      editingBankIndex: -1,
      myVar: null,
      myVar1: null,
      rewardShadow: false,
      checked: "wepay",
      showGains: true,
      showAmount: true,
      showCost: true,
      showCostRate: true,
      showGSZ: true,
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
      bankSortType: {
        principal: "none",
        totalAmount: "none",
        profit: "none",
        annualIncome: "none",
        annualRate: "none",
      },
      bankSortTypeObj: {
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
      containerWidth: 1050,
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
      opacity: {},
      opacityValue: 0,
      editingNavCode: null,
      editingFundCode: null,
      editingStockCode: null,
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
    // 兜底：如果 storage 回调未执行，3秒后强制显示界面
    setTimeout(() => {
      if (!this.isGetStorage) {
        console.warn("storage callback timeout, forcing isGetStorage=true");
        this.isGetStorage = true;
      }
    }, 3000);
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
      return this.sumGains(this.freeDataList.filter(item => !item.isPrivate));
    },
    safeGains() {
      return this.sumGains(this.safeDataList.filter(item => !item.isPrivate));
    },
    freeCostGains() {
      return this.sumCostGains(this.freeDataList.filter(item => !item.isPrivate));
    },
    safeCostGains() {
      return this.sumCostGains(this.safeDataList.filter(item => !item.isPrivate));
    },
    fundAmount() {
      return this.sumAmount(this.dataList);
    },
    stockAmount() {
      return this.sumAmount(this.stockDataList);
    },
    bankTotalAmount() {
      return this.bankList.reduce((sum, item) => sum + (parseFloat(item.totalAmount) || 0), 0).toFixed(0);
    },
    bankPrincipal() {
      return this.bankList.reduce((sum, item) => sum + (parseFloat(item.principal) || 0), 0).toFixed(0);
    },
    bankAnnualIncome() {
      return this.bankList.reduce((sum, item) => sum + (parseFloat(item.annualIncome) || 0), 0).toFixed(0);
    },
    bankGains() {
      let total = parseFloat(this.bankTotalAmount);
      let principal = parseFloat(this.bankPrincipal);
      let gains = (total - principal).toFixed(0);
      let rate = principal > 0 ? ((total - principal) / principal * 100).toFixed(1) : '0.00';
      return [gains, rate];
    },
    allAmount() {
      let holdingAmount = parseFloat(this.sumAmount(this.allHoldingList)) || 0;
      let bankAmount = parseFloat(this.bankTotalAmount) || 0;
      return (holdingAmount + bankAmount).toFixed(0);
    },
    fundGains() {
      return this.sumGains(this.dataList.filter(item => !item.isPrivate));
    },
    stockGains() {
      return this.sumGains(this.stockDataList);
    },
    allGains() {
      return this.sumGains(this.allHoldingList.filter(item => !item.isPrivate));
    },
    fundCostGains() {
      return this.sumCostGains(this.dataList.filter(item => !item.isPrivate));
    },
    stockCostGains() {
      return this.sumCostGains(this.stockDataList);
    },
    allCostGains() {
      return this.sumCostGains(this.allHoldingList.filter(item => !item.isPrivate));
    },
    pieChartData() {
      const data = [];
      const fundAmount = parseFloat(this.fundAmount) || 0;
      const freeAmount = parseFloat(this.freeAmount) || 0;
      const safeAmount = parseFloat(this.safeAmount) || 0;
      const stockAmount = parseFloat(this.stockAmount) || 0;
      const bankAmount = parseFloat(this.bankTotalAmount) || 0;

      const otherFundAmount = fundAmount - freeAmount - safeAmount;

      if (freeAmount > 0.01) data.push({ name: '财务自由', value: freeAmount });
      if (safeAmount > 0.01) data.push({ name: '财务安全', value: safeAmount });
      if (otherFundAmount > 0.01) data.push({ name: '其他基金', value: otherFundAmount });

      if (stockAmount > 0.01) data.push({ name: '股票', value: stockAmount });
      if (bankAmount > 0.01) data.push({ name: '现金', value: bankAmount });
      return data;
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
      return totalAmount.toFixed(0);
    },
    sumGains(list) {
      let allGains = 0;
      let allNum = 0;
      list.forEach((val) => {
        allGains += parseFloat(val.gains || 0);
        allNum += parseFloat(val.amount || 0);
      });
      allGains = allGains.toFixed(0);
      let allGainsRate = allNum ? ((allGains * 100) / allNum).toFixed(1) : NaN;
      return [allGains, allGainsRate];
    },
    sumCostGains(list) {
      let allCostGains = 0;
      let allNum = 0;
      list.forEach((val) => {
        allCostGains += parseFloat(val.costGains || 0);
        allNum += parseFloat(val.amount || 0);
      });
      allCostGains = allCostGains.toFixed(0);
      const denominator = allNum - allCostGains;
      let allCostGainsRate = denominator ? ((allCostGains * 100) / denominator).toFixed(1) : NaN;
      return [allCostGains, allCostGainsRate];
    },
    toNullableNumber(value) {
      if (value === null || value === undefined || value === "") {
        return null;
      }

      const numberValue = Number(value);
      return Number.isNaN(numberValue) ? null : numberValue;
    },
    summaryGainCardClass(value) {
      const numberValue = this.toNullableNumber(value);

      if (numberValue === null || numberValue === 0) {
        return "summary-card-soft";
      }

      return numberValue > 0 ? "summary-card-red" : "summary-card-green";
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
        accountType: source.accountType || '',
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

      return ((latestPrice - prevClose) * num).toFixed(0);
    },
    calculateStockCostGain(latestPrice, cost, num) {
      if (cost === null || cost === undefined || cost === "") {
        return 0;
      }

      return ((latestPrice - cost) * num).toFixed(0);
    },
    calculateStockCostGainRate(latestPrice, cost) {
      if (!cost) {
        return 0;
      }

      return (((latestPrice - cost) / cost) * 100).toFixed(1);
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
      chrome.storage.local.get(["fundListM"], (localRes) => {
      chrome.storage.sync.get(
        [
          "RealtimeFundcode",
          "RealtimeIndcode",
          "showNum",
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
          "fundListGroup",
          "stockSortTypeObj",
          "bankSortTypeObj",
          "bankList",
          "stockListM",
        ],
        (res) => {
          if (chrome.runtime.lastError) {
            console.error("storage.sync.get error:", chrome.runtime.lastError);
          }
          // 优先从 local 读取 fundListM（突破 sync 的 8KB 限制）
          if (localRes.fundListM && localRes.fundListM.length) {
            res.fundListM = localRes.fundListM;
          }
          try {
            this.fundList = res.fundList ? res.fundList : this.fundList;
            if (res.fundListM && res.fundListM.length) {
              this.fundListM = res.fundListM;
            } else if (res.fundListGroup && res.fundListGroup.length) {
              this.fundListM = flattenFundListGroup(res.fundListGroup);
              chrome.storage.local.set({
                fundListM: this.fundListM,
              });
            } else {
              for (const fund of this.fundList) {
                let val = {
                  code: fund,
                  num: 0,
                };
                this.fundListM.push(val);
              }
              chrome.storage.local.set({
                fundListM: this.fundListM,
              });
            }
            this.stockListM = (res.stockListM || [])
              .map((item) => this.normalizeStockItem(item))
              .filter(Boolean);
            this.bankList = res.bankList || [];
            this.bankListDft = [...this.bankList];
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
            if (res.showNum) {
              // 解决版本遗留问题，拆分属性
              chrome.storage.sync.set({ showNum: false });
              chrome.storage.sync.set({ showAmount: true });
              chrome.storage.sync.set({ showGains: true });
              this.showAmount = true;
              this.showGains = true;
            } else {
              this.showAmount = res.showAmount !== undefined ? res.showAmount : true;
              this.showGains = res.showGains !== undefined ? res.showGains : true;
            }
            this.RealtimeFundcode = res.RealtimeFundcode;
            this.RealtimeIndcode = res.RealtimeIndcode;
            this.isLiveUpdate = res.isLiveUpdate ? res.isLiveUpdate : false;
            this.showCost = res.showCost !== undefined ? res.showCost : true;
            this.showCostRate = res.showCostRate !== undefined ? res.showCostRate : true;
            this.showGSZ = res.showGSZ !== undefined ? res.showGSZ : true;
            this.BadgeContent = res.BadgeContent ? res.BadgeContent : 1;
            this.showBadge = res.showBadge ? res.showBadge : 1;
            this.grayscaleValue = res.grayscaleValue ? res.grayscaleValue : 0;
            this.opacityValue = res.opacityValue ? res.opacityValue : 0;
            this.sortTypeObj = res.sortTypeObj ? res.sortTypeObj : {};
            this.stockSortTypeObj = res.stockSortTypeObj ? res.stockSortTypeObj : {};
            this.bankSortTypeObj = res.bankSortTypeObj ? res.bankSortTypeObj : {};
            
            if (this.bankSortTypeObj.type && this.bankSortTypeObj.type != "none") {
              this.bankSortType[this.bankSortTypeObj.name] = this.bankSortTypeObj.type;
              this.bankList = [...this.bankListDft].sort((a, b) => {
                let valA = 0;
                let valB = 0;
                const type = this.bankSortTypeObj.name;
                if (type === 'profit') {
                  valA = (parseFloat(a.totalAmount) || 0) - (parseFloat(a.principal) || 0);
                  valB = (parseFloat(b.totalAmount) || 0) - (parseFloat(b.principal) || 0);
                } else {
                  valA = parseFloat(a[type]) || 0;
                  valB = parseFloat(b[type]) || 0;
                }
                if (this.bankSortTypeObj.type === 'asc') {
                  return valA - valB;
                } else {
                  return valB - valA;
                }
              });
            }
          } catch (e) {
            console.error("init storage callback error:", e);
          }

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
      }); // end chrome.storage.local.get
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
    sortBankList(type) {
      for (const key in this.bankSortType) {
        if (this.bankSortType.hasOwnProperty(key) && key != type) {
          this.bankSortType[key] = "none";
        }
      }
      this.bankSortType[type] =
        this.bankSortType[type] == "desc"
          ? "asc"
          : this.bankSortType[type] == "asc"
          ? "none"
          : "desc";
      
      if (this.bankSortType[type] == "none") {
        this.bankList = [...this.bankListDft];
      } else {
        this.bankList = [...this.bankList].sort((a, b) => {
          let valA = 0;
          let valB = 0;
          if (type === 'profit') {
            valA = (parseFloat(a.totalAmount) || 0) - (parseFloat(a.principal) || 0);
            valB = (parseFloat(b.totalAmount) || 0) - (parseFloat(b.principal) || 0);
          } else {
            valA = parseFloat(a[type]) || 0;
            valB = parseFloat(b[type]) || 0;
          }
          if (this.bankSortType[type] === 'asc') {
            return valA - valB;
          } else {
            return valB - valA;
          }
        });
      }
      this.bankSortTypeObj = {
        name: type,
        type: this.bankSortType[type],
      };
      chrome.storage.sync.set({
        bankSortTypeObj: this.bankSortTypeObj,
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
                ? (((displayPrice - prevClose) / prevClose) * 100).toFixed(1)
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
              gszzl: Number(changeRate).toFixed(1),
              gztime: quote.f124 || null,
              num,
              cost,
              accountType: item.accountType || '',
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

          // 为 API 未返回的基金（私募/投顾等）创建 fallback 条目
          let apiCodes = new Set(dataList.map(item => item.fundcode));
          this.fundListM.forEach(item => {
            if (!apiCodes.has(item.code)) {
              let costVal = parseFloat(item.cost) || 0;
              let numVal = parseFloat(item.num) || 0;
              let dwjz = costVal;
              if (item.nav !== undefined && item.nav !== null) {
                dwjz = parseFloat(item.nav) || costVal;
              }
              let fallback = {
                fundcode: item.code,
                name: item.name || item.code,
                jzrq: '--',
                dwjz: dwjz,
                gsz: dwjz,
                gszzl: 0,
                gztime: '--',
                num: numVal,
                cost: costVal,
                accountType: item.accountType || '',
                isPrivate: true,
              };
              fallback.amount = this.calculateMoney(fallback);
              fallback.gains = 0;
              fallback.costGains = this.calculateCost(fallback);
              fallback.costGainsRate = this.calculateCostRate(fallback);
              dataList.push(fallback);
            }
          });

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
        chrome.storage.local.set(
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
        chrome.storage.local.set(
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
      let sum = (val.dwjz * val.num).toFixed(0);
      return sum;
    },
    calculate(val, hasReplace) {
      var sum = 0;
      let num = val.num ? val.num : 0;
      if (hasReplace) {
        sum = ((val.dwjz - val.dwjz / (1 + val.gszzl * 0.01)) * num).toFixed(0);
      } else {
        if (val.gsz != null) {
          sum = ((val.gsz - val.dwjz) * num).toFixed(0);
        }
      }
      return sum;
    },
    calculateCost(val) {
      if (val.cost) {
        let sum = ((val.dwjz - val.cost) * val.num).toFixed(0);
        return sum;
      } else {
        return 0;
      }
    },
    calculateCostRate(val) {
      if (val.cost && val.cost != 0) {
        let sum = (((val.dwjz - val.cost) / val.cost) * 100).toFixed(1);
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

      chrome.storage.local.set(
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
      chrome.storage.local.set(
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
    editNav(el) {
      if (el.isPrivate) {
        this.editingNavCode = el.fundcode;
      }
    },
    saveNav(el) {
      this.editingNavCode = null;
      // 更新 fundListM 中的 nav
      this.fundListM = this.fundListM.map(item => {
        if (item.code === el.fundcode) {
          return { ...item, nav: el.gsz };
        }
        return item;
      });
      // 持久化到 local 存储
      chrome.storage.local.set({ fundListM: this.fundListM });
      // 重新计算收益
      el.dwjz = el.gsz;
      el.amount = this.calculateMoney(el);
      el.costGains = this.calculateCost(el);
      el.costGainsRate = this.calculateCostRate(el);
    },
    dblclickBank(index) {
      this.editingBankIndex = index;
    },
    saveBankEdit() {
      this.editingBankIndex = -1;
      chrome.storage.sync.set({ bankList: this.bankList });
    },
    dltBank(index) {
      this.bankList.splice(index, 1);
      this.editingBankIndex = -1;
      chrome.storage.sync.set({ bankList: this.bankList });
    },
    handleDragStart(e, item) {
      this.dragging = item;
    },
    handleDragOver(e) {
      e.dataTransfer.dropEffect = "move";
    },
    handleDragEnd(e, item) {
      this.dragging = null;
      if (item && item.fundcode) {
        chrome.storage.local.set(
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
    dblclickFund(el) {
      if (this.isEdit || !el.isPrivate) return;
      this.editingFundCode = el.fundcode;
    },
    saveFundEdit(el, index) {
      this.editingFundCode = null;
      let fund = this.fundListM.find((item) => item.code === el.fundcode);
      if (fund) {
        fund.num = parseFloat(el.num) || 0;
        fund.cost = parseFloat(el.cost) || 0;
        if (el.isPrivate) {
          fund.nav = parseFloat(el.gsz) || 0;
        }
      } else {
        this.fundListM.push({
          code: el.fundcode,
          num: parseFloat(el.num) || 0,
          cost: parseFloat(el.cost) || 0,
          accountType: el.accountType || "",
        });
      }

      chrome.storage.local.set({ fundListM: this.fundListM }, () => {
        el.amount = this.calculateMoney(el);
        el.costGains = this.calculateCost(el);
        el.costGainsRate = this.calculateCostRate(el);
        if (el.isPrivate) {
          el.dwjz = parseFloat(el.gsz) || 0;
          el.gains = 0;
        }
        if (this.dataList[index]) {
          this.$set(this.dataList, index, { ...el });
        }
      });
    },
    dblclickStock(el) {
      // 仅保留私募/投顾，股票暂不支持双击编辑
      return;
    },
    saveStockEdit(el, index) {
      this.editingStockCode = null;
      let stock = this.stockListM.find((item) => item.code === el.stockcode);
      if (stock) {
        stock.num = parseFloat(el.num) || 0;
        stock.cost = parseFloat(el.cost) || 0;
      }

      chrome.storage.sync.set({ stockListM: this.stockListM }, () => {
        const latestPrice = parseFloat(el.latestPriceValue) || 0;
        const prevClose = parseFloat(el.dwjz) || latestPrice;
        el.num = parseFloat(el.num) || 0;
        el.cost = parseFloat(el.cost) || 0;
        el.amount = latestPrice * el.num;
        el.gains = (latestPrice - prevClose) * el.num;
        el.costGains = (latestPrice - el.cost) * el.num;
        el.costGainsRate = el.cost > 0 ? (((latestPrice - el.cost) / el.cost) * 100).toFixed(2) : 0;
        
        if (this.stockDataList[index]) {
          this.$set(this.stockDataList, index, { ...el });
        }
      });
    },
  },
};

</script>

<style lang="scss" scoped>
.container {
  --c-bg: #f7f8fa;
  --c-surface: #ffffff;
  --c-border: #e8eaed;
  --c-text: #1f2937;
  --c-text-secondary: #6b7280;
  --c-accent: #3b82f6;
  --c-accent-light: #eff6ff;
  --c-up: #ef4444;
  --c-down: #22c55e;
  min-width: 720px;
  min-height: 680px;
  overflow-x: hidden;
  overflow-y: auto;
  padding: 12px;
  box-sizing: border-box;
  position: relative;
  font-size: 12px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Hiragino Sans GB",
    "Microsoft YaHei", sans-serif;
  color: var(--c-text);
  background: var(--c-bg);
  border: none;
  border-radius: 0;
  box-shadow: none;
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
  margin-top: 8px;
  min-height: 420px;
  min-width: 0;
  padding: 0;
  border-radius: 0;
  border: none;
  background: transparent;
  box-shadow: none;
}

.page-tabs {
  min-height: 620px;
  min-width: 0;
}

.home-pane {
  display: flex;
  flex-direction: column;
  gap: 4px;
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
  /* no min-width, table handles its own scroll */
}

.stock-more-width {
  /* no min-width, table handles its own scroll */
}

.changelog-container {
  min-height: 575px;
}

.table-more-height {
  min-height: 220px;
}
.table-drag {
  cursor: move;
}

.container {
  &.num-width-1,
  &.num-width-2,
  &.num-width-3,
  &.num-width-4,
  &.num-width-5 {
    /* removed fixed min-widths to prevent horizontal scroll */
  }
}

.table-row {
  min-height: 420px;
  max-height: 520px;
  position: relative;
  overflow-y: auto;
  overflow-x: auto;
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
  border-radius: 8px;
  background: var(--c-surface);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
  border: 1px solid var(--c-border);
}
.align-left {
  text-align: left;
}

.center {
  text-align: center;
}

table th {
  padding: 8px 10px;
  white-space: nowrap;
  color: var(--c-text-secondary);
  font-weight: 600;
  font-size: 11px;
}

thead th {
  position: sticky;
  top: 0;
  z-index: 10;
  background: #f9fafb;
  box-shadow: 0 1px 0 var(--c-border);
}

thead th:first-child {
  border-top-left-radius: 8px;
}

thead th:last-child {
  border-top-right-radius: 8px;
}

table td {
  padding: 8px 10px;
  white-space: nowrap;
  color: var(--c-text);
  border-bottom: 1px solid #f3f4f6;
}

.up {
  color: var(--c-up);
  font-weight: bold;
}

.down {
  color: var(--c-down);
  font-weight: bold;
}

tbody tr:hover {
  background: #f9fafb;
}

tbody tr:last-child td,
tbody tr:last-child th {
  border-bottom: none;
}

.btn {
  display: inline-block;
  line-height: 1;
  cursor: pointer;
  background: var(--c-surface);
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 12px;
  color: var(--c-text);
  margin: 0 4px;
  outline: none;
  border: 1px solid var(--c-border);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
  transition: background 0.15s, border-color 0.15s;
}

.btn:hover {
  background: #f9fafb;
  border-color: #d1d5db;
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
  color: var(--c-up);
  border-color: rgba(239, 68, 68, 0.3);
  background: #fef2f2;
  font-weight: bold;
}

.btn-down {
  color: var(--c-down);
  border-color: rgba(34, 197, 94, 0.3);
  background: #f0fdf4;
  font-weight: bold;
}

.btn-total-amount {
  color: var(--c-up);
  border-color: rgba(239, 68, 68, 0.3);
  background-color: #fef2f2;
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
  margin: 8px 0 4px;
  padding: 8px;
  border-radius: 8px;
  border: 1px solid var(--c-border);
  background: var(--c-surface);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
}

.summary-row {
  margin-top: 5px;
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

.summary-row:first-child {
  margin-top: 0;
}

.summary-group {
  margin-top: 8px;
}

.summary-group:first-child {
  margin-top: 0;
}

.summary-group > .summary-row {
  margin-top: 0;
}

.summary-children {
  display: flex;
  flex-direction: column;
  gap: 5px;
  margin-top: 5px;
}

.summary-child-row {
  display: flex;
  align-items: stretch;
}

.tree-branch {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 14px;
  flex-shrink: 0;
  color: #d1d5db;
  font-size: 14px;
  line-height: 1;
  user-select: none;
}

.summary-child-row .summary-row {
  flex: 1;
  min-width: 0;
  margin-top: 0;
}

.summary-child-content {
  flex: 1;
  min-width: 0;
}

.summary-child-content > .summary-row {
  margin-top: 0;
}

.summary-child-content > .summary-children {
  margin-top: 5px;
}

.summary-row-home {
  /* flex inherited from .summary-row */
}

.tab-summary-panel {
  margin: 0 0 8px;
  padding: 8px 10px;
}

.tab-summary-row {
  /* flex inherited from .summary-row */
}

.summary-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
  margin: 4px 0;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid var(--c-border);
  background: var(--c-surface);
}

.summary-table thead th {
  padding: 6px 12px;
  text-align: right;
  font-weight: 600;
  font-size: 11px;
  color: var(--c-text-secondary);
  background: #f9fafb;
  border-bottom: 1px solid var(--c-border);
  position: sticky;
  top: 0;
  z-index: 2;
}

.summary-table thead th:first-child {
  text-align: left;
}

.summary-table tbody td {
  padding: 5px 12px;
  text-align: right;
  font-weight: 500;
  font-size: 12px;
  white-space: nowrap;
  border-bottom: 1px solid #f3f4f6;
}

.summary-table tbody td:first-child {
  text-align: left;
  color: var(--c-text-secondary);
  font-weight: 600;
}

.summary-table tbody tr:last-child td {
  border-bottom: none;
}

.summary-table-total td {
  font-weight: 700 !important;
  font-size: 13px !important;
  background: #f9fafb;
}

/* === Tree view rows === */
.tree-root-node {
  display: inline-flex;
  align-items: center;
  font-weight: 700;
  font-size: 13px;
  color: var(--c-text);
}

.tree-node {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  position: relative;
}

.tree-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}

/* Level-1 rows: draw vertical connector from total row down */
.tree-level-1 td:first-child {
  padding-left: 10px;
  position: relative;
}

.tree-level-1 td:first-child::before {
  content: '';
  position: absolute;
  left: 18px;
  top: 0;
  bottom: 0;
  width: 1px;
  background: var(--c-border);
}

.tree-level-1 td:first-child::after {
  content: '';
  position: absolute;
  left: 18px;
  top: 50%;
  width: 8px;
  height: 1px;
  background: var(--c-border);
}

.tree-level-1.tree-last td:first-child::before {
  bottom: 50%;
}

/* Level-2 rows (sub-rows under fund) */
.tree-level-2 td:first-child {
  padding-left: 28px;
  position: relative;
}

.tree-level-2 td:first-child::before {
  content: '';
  position: absolute;
  left: 36px;
  top: 0;
  bottom: 0;
  width: 1px;
  background: var(--c-border);
  opacity: 0.7;
}

.tree-level-2 td:first-child::after {
  content: '';
  position: absolute;
  left: 36px;
  top: 50%;
  width: 8px;
  height: 1px;
  background: var(--c-border);
  opacity: 0.7;
}

.tree-level-2.tree-last td:first-child::before {
  bottom: 50%;
}

.summary-table-sub td {
  font-size: 11px !important;
  opacity: 0.8;
}

.summary-table .pct {
  font-size: 10px;
  opacity: 0.6;
  margin-left: 2px;
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
  padding: 6px 12px;
  border-radius: 6px;
  border: 1px solid var(--c-border);
  background: var(--c-surface);
  box-shadow: none;
  display: flex;
  flex-direction: row;
  align-items: baseline;
  gap: 6px;
  text-align: left;
  flex: 1;
}

.summary-card-compact {
  padding: 5px 10px;
  overflow: hidden;
}

.summary-card-compact .summary-label {
  white-space: nowrap;
  font-size: 11px;
  flex-shrink: 0;
}

.summary-card-compact .summary-value {
  font-size: 14px;
  white-space: nowrap;
  font-weight: 700;
}

.summary-card-compact .summary-meta {
  font-size: 11px;
  white-space: nowrap;
  opacity: 0.7;
}

.summary-card-soft {
  border-style: dashed;
}

.summary-card-green {
  border-color: rgba(34, 197, 94, 0.25);
  background: #f0fdf4;
  color: #16a34a;
  .summary-value, .summary-meta { color: inherit; }
}

.summary-card-red {
  border-color: rgba(239, 68, 68, 0.25);
  background: #fef2f2;
  color: #dc2626;
  .summary-value, .summary-meta { color: inherit; }
}

.summary-label {
  font-size: 11px;
  font-weight: 600;
  color: var(--c-text-secondary);
}

.summary-value {
  font-size: 14px;
  line-height: 1.2;
  font-weight: 700;
  white-space: nowrap;
}

.summary-meta {
  font-size: 10px;
  font-weight: 500;
  color: var(--c-text-secondary);
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
  min-width: 80px;
  border-radius: 6px;
  padding: 8px 14px;
}

.market-overview {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 8px;
  margin: 0;
  padding: 0;
}

.market-overview .tab-col {
  min-width: 0;
  margin: 0;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid var(--c-border);
  background: var(--c-surface);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
  overflow: hidden;
}

.market-overview .tab-col h5 {
  margin: 0 0 4px;
  font-size: 12px;
  font-weight: 600;
  color: var(--c-text-secondary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.market-overview .tab-col p {
  margin: 1px 0;
  font-size: 12px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
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
  background: var(--c-surface);
  border: 1px solid var(--c-border);
  border-radius: 8px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
  padding: 3px;
  margin-bottom: 10px;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__nav-wrap {
  overflow: visible;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__nav-wrap::after {
  display: none;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__nav-scroll {
  overflow: visible;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__nav {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  gap: 3px;
  float: none;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__item {
  width: auto;
  min-width: 0;
  height: 26px;
  line-height: 26px;
  text-align: center;
  padding: 0 8px;
  font-size: 12px;
  border-radius: 5px;
  flex-shrink: 0;
  white-space: nowrap;
  color: var(--c-text-secondary);
  font-weight: 500;
  float: none;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__active-bar {
  display: none;
}

::v-deep .page-tabs > .el-tabs__header .el-tabs__item.is-active {
  background: var(--c-accent-light);
  color: var(--c-accent);
  font-weight: 600;
  border-radius: 6px;
}

::v-deep .page-tabs > .el-tabs__content {
  overflow: visible;
  min-width: 0;
}

::v-deep .el-tabs__item {
  height: 34px;
  line-height: 34px;
  font-size: 13px;
  font-weight: 600;
  color: var(--c-text-secondary);
}

::v-deep .el-tabs__item.is-active {
  color: var(--c-accent);
}

::v-deep .el-tabs__active-bar {
  height: 2px;
  border-radius: 999px;
  background: var(--c-accent);
}

::v-deep .el-tabs__nav-wrap::after {
  background-color: var(--c-border);
}

::v-deep .el-tabs__content {
  min-height: 420px;
}

.btn-subtotal {
  border-style: dashed;
  color: var(--c-up);
  border-color: rgba(239, 68, 68, 0.3);
  background-color: #fef2f2;
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
  color: var(--c-text-secondary);
  line-height: 1.4;
  padding: 6px 15px;
}


.fundName {
  max-width: 120px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  cursor: pointer;
  user-select: none;
}

.fundName-noclick {
  max-width: 120px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fundName:hover {
  color: var(--c-accent);
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
  font-size: 14px;
  &.num-width-1,
  &.num-width-2,
  &.num-width-3,
  &.num-width-4,
  &.num-width-5 {
    /* removed fixed min-widths */
  }

  &.stock-more-width {
    /* removed fixed min-width */
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
  --c-bg: #0f1117;
  --c-surface: #1a1d27;
  --c-border: rgba(255, 255, 255, 0.1);
  --c-text: rgba(255, 255, 255, 0.87);
  --c-text-secondary: rgba(255, 255, 255, 0.5);
  --c-accent: #60a5fa;
  --c-accent-light: rgba(96, 165, 250, 0.12);
  --c-up: #f87171;
  --c-down: #4ade80;
  color: var(--c-text);
  background-color: var(--c-bg);
  background-image: none;
  border-color: var(--c-border);
  box-shadow: none;
  thead th {
    background-color: #1e2130;
    background-image: none;
    box-shadow: 0 1px 0 var(--c-border);
    color: var(--c-text-secondary);
  }
  .summary-panel {
    background: var(--c-surface);
    border-color: var(--c-border);
    box-shadow: none;
  }
  .summary-card {
    background: rgba(255, 255, 255, 0.06);
    border-color: var(--c-border);
    box-shadow: none;
  }
  .summary-card-green {
    color: #4ade80;
    background: rgba(74, 222, 128, 0.08);
    border-color: rgba(74, 222, 128, 0.2);
  }
  .summary-card-red {
    color: #f87171;
    background: rgba(248, 113, 113, 0.08);
    border-color: rgba(248, 113, 113, 0.2);
  }
  .summary-table {
    background: var(--c-surface);
    border-color: var(--c-border);
  }
  .summary-table thead th {
    background: #1e2130;
    color: var(--c-text-secondary);
    border-bottom-color: var(--c-border);
  }
  .summary-table tbody td {
    color: var(--c-text);
    border-bottom-color: rgba(255, 255, 255, 0.05);
  }
  .summary-table tbody td:first-child {
    color: var(--c-text-secondary);
  }
  .summary-table-total td {
    background: rgba(255, 255, 255, 0.03);
  }
  .holdings-tabs-wrap {
    background: transparent;
  }
  .market-overview .tab-col {
    background: var(--c-surface);
    border-color: var(--c-border);
    box-shadow: none;
  }
  ::v-deep .page-tabs > .el-tabs__header {
    background: var(--c-surface);
    border-color: var(--c-border);
  }
  ::v-deep .page-tabs > .el-tabs__header .el-tabs__item.is-active {
    background: var(--c-accent-light);
    color: var(--c-accent);
  }
  .refresh {
    color: var(--c-accent);
  }
  .btn {
    background-color: rgba(255, 255, 255, 0.08);
    color: var(--c-text);
    border-color: var(--c-border);
  }
  .btn:hover {
    background-color: rgba(255, 255, 255, 0.12);
  }
  .primary {
    border-color: var(--c-accent);
    background-color: rgba(96, 165, 250, 0.2);
    color: var(--c-accent);
  }
  .action-row .btn {
    background-color: rgba(255, 255, 255, 0.08);
  }
  ::v-deep .el-input__inner {
    background-color: rgba(255, 255, 255, 0.08);
    color: var(--c-text);
    border-color: var(--c-border);
  }
  ::v-deep .el-select__input {
    color: var(--c-text);
  }

  ::v-deep tbody tr:hover {
    background: rgba(255, 255, 255, 0.06) !important;
  }

  tbody tr:hover {
    background: rgba(255, 255, 255, 0.06) !important;
  }

  .slt {
    border-color: rgba(74, 222, 128, 0.5);
    background-color: rgba(74, 222, 128, 0.2);
  }

  .btn.red {
    border-color: rgba(248, 113, 113, 0.5);
    background-color: rgba(248, 113, 113, 0.2);
  }

  .btn-up {
    border-color: rgba(248, 113, 113, 0.4);
    background-color: rgba(248, 113, 113, 0.12);
  }

  .btn-down {
    border-color: rgba(74, 222, 128, 0.4);
    background-color: rgba(74, 222, 128, 0.12);
  }

  .btn-total-amount {
    color: #f87171;
    border-color: rgba(248, 113, 113, 0.4);
    background-color: rgba(248, 113, 113, 0.12);
  }

  .btn-subtotal {
    color: #f87171;
    border-color: rgba(248, 113, 113, 0.4);
    background-color: rgba(248, 113, 113, 0.12);
  }

  .tab-col {
    background-color: rgba(255, 255, 255, 0.06);
    border-radius: 6px;
  }

  table {
    background-color: var(--c-surface);
    border-color: var(--c-border);
    border-radius: 8px;
  }

  table {
    margin: 0 auto;
    width: 100%;
    border-collapse: separate;
    border-spacing: 0;
    text-align: right;
    overflow: hidden;
    border-radius: 8px;
    background: var(--c-surface);
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
    /* 让sticky表头在圆角下不被遮挡 */
    overflow: visible;
  }

  table th {
    color: var(--c-text-secondary);
  }

  ::v-deep .el-tabs__item {
    color: var(--c-text-secondary);
  }

  ::v-deep .el-tabs__item.is-active {
    color: var(--c-accent);
  }

  ::v-deep .el-tabs__active-bar {
    background: var(--c-accent);
  }

  ::v-deep .el-tabs__nav-wrap::after {
    background-color: var(--c-border);
  }

  ::placeholder {
    color: rgba($color: #ffffff, $alpha: 0.38);
  }

  ::v-deep .el-select .el-input.is-focus .el-input__inner {
    border-color: var(--c-accent);
  }

  ::v-deep .el-select .el-tag {
    background-color: rgba(255, 255, 255, 0.1);
    color: var(--c-text);
  }

  ::v-deep .el-select-dropdown {
    background-color: #1e2130;
    border: 1px solid var(--c-border);
    .popper__arrow::after {
      border-bottom-color: #1e2130;
    }
    .el-scrollbar {
      background-color: rgba(255, 255, 255, 0.06);
    }
    .el-select-dropdown__item {
      color: var(--c-text);
    }

    .el-select-dropdown__item.hover,
    .el-select-dropdown__item:hover {
      background-color: rgba(255, 255, 255, 0.06);
    }
    .el-select-dropdown__item.selected {
      color: var(--c-accent);
      background-color: rgba(255, 255, 255, 0.06);
    }
    .el-select-dropdown__item.selected::after {
      color: var(--c-accent);
    }
  }

  ::v-deep .el-switch__label.is-active {
    color: var(--c-accent);
  }
  ::v-deep .el-switch__label {
    color: var(--c-text-secondary);
  }

  ::v-deep .hasReplace-tip {
    color: var(--c-text);
    border-color: var(--c-accent);
    background-color: rgba(96, 165, 250, 0.2);
  }

  .fundName:hover {
    color: var(--c-accent);
  }
}
.private-fund-row {
  background-color: rgba(255, 243, 224, 0.5) !important;
}

.darkMode .private-fund-row {
  background-color: rgba(74, 20, 140, 0.3) !important;
}

.small-input {
  width: 60px !important;
  height: 20px !important;
  padding: 0 4px !important;
  font-size: 12px !important;
}
</style>
