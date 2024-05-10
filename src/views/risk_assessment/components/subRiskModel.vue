<!-- 风险模型的页面  
区别：
1.接受父组件传参
2.无初始加载数据（无mounted）
-->

<script>
// import InputSearch from '@/sfc/InputSearch.vue'
import AreaAndmonitorType from '@/sfc/AreaAndmonitorType.vue'
import exceptionApi from '@/api/exceptionApi.js'

import DustRadarChart from '@/views/risk_assessment/components/DustRadarChart.vue'
import FYLineChart from '@/components/chart/FYLineChart.vue'
import index from '@/utils/risk_estimate_common_function/index.js'
import ButtonClick from '@/sfc/ButtonClick.vue'
import dayjs from 'dayjs'
import MonthSelect from '@/sfc/MonthSelect.vue'
export default {
  props: {
    sName: {
      type: String,
      default: ''
    },
    month_1: {
      type: String,
      default: ''
    }
  },
  components: {
    FYLineChart,
    // DateSelectWithShortCuts,
    // InputSearch,
    AreaAndmonitorType,
    DustRadarChart,
    ButtonClick,
    MonthSelect
  },
  data() {
    return {
      isNoData: false,
      loading: false,
      screenLoading: false,
      parentDataFlag: false,
      chartData: [],
      chartData1: {}, //保存查询的结果
      chartData2: {},
      chartData3: {},
      chartData4: {},
      //devId:'',          //设备编号
      begin: '', //开始时间
      end: '', //结束时间

      form: {
        // 站点名称
        name: '',
        // 设备编号
        number: '',

        // month:'',
        // 开始时间
        beginTime: '',
        // 结束时间
        endTime: ''
      },
      month: '',
      // 传递给月份组件的值
      sfc_month: '',
      // 折线图配置项
      option: {},
      // 数据清单
      bill: {
        min: '',
        max: '',
        avg: '',
        online: 100,
        valid: 100,
        exceeding: 0,

        //  典型异常复现率
        exceptionRecurrence: 0,
        // 异常类型据聚集度
        exceptionTypeAggregation: 0
      },
      status: {
        isHasData: false
      },
      // 查询按钮加载效果
      queryButton: false,

      // 风险值排名前十的站点
      top_10_sites_with_risk_values: [{ name: '', riskValue: 0.2 }],
      // 风险值
      weight: ''
    }
  },

  watch: {
    sName() {
      // 当别的页面调用时
      if (this.sName != '' || this.month_1 != '') {
        // 更新站点名字和时间
        this.form.name = this.sName
        this.month = this.month_1
        // 同时更新月起始时间
        this.form.beginTime = dayjs(this.month).startOf().format('YYYY-MM-DD HH:mm:ss')
        this.form.endTime = dayjs(this.month).endOf('month').format('YYYY-MM-DD HH:mm:ss')

        this.parentDataFlag = true

        // 更新统计数据
        this.getAnalysisData()
        // 计算风险值
        this.calRiskValue()
      }
    }
  },

  mounted() {},

  methods: {
    // 查询站点统计信息
    querySiteStaticsInfo(row) {
      this.form.name = row.siteName
      // 更新统计数据
      this.getAnalysisData()
      // 计算风险值
      this.calRiskValue()
    },

    // 格式化月份
    giveMonth(val) {
      //将中国标准时间转为指定格式(该组件返回的标准时间的格式，所以必须的加这个函数)
      this.month = dayjs(val).format('YYYY-MM-DD')
      // 同时更新开始和结束时间
      this.form.beginTime = dayjs(this.month).startOf().format('YYYY-MM-DD HH:mm:ss')
      this.form.endTime = dayjs(this.month).endOf('month').format('YYYY-MM-DD HH:mm:ss')
      console.log(this.form.beginTime, this.form.endTime)
    },

    /**
     * 风险评估按钮
     * @param：
     * @returns：
     */
    riskAssessment() {
      // 更新排名清单
      this.getRiskRank()
      // 更新分析数据
      this.getAnalysisData()
      // 计算风险值
      this.calRiskValue()
    },

    // 更新分析数据
    getAnalysisData() {
      // 日统计数据
      this.fetchDayAnalysisData()
      // 异常数据
      this.fetchExceptionAnalysisData()
    },

    findObjectByPropertyValue(array, property, value) {
      return array.find((obj) => obj[property] === value)
    },

    // 此页面打开时
    ShowDefaultData() {
      // 默认加载展示的内容
      this.updateOriginPage()
    },

    // 加载默认展示的内容
    async updateOriginPage() {
      // 更新排名清单，并得到风险值最高站点
      this.form.name = await this.getRiskRank()

      // 更新日统计数据
      this.getAnalysisData()
      // 计算风险值
      this.calRiskValue()
    },

    // 计算风险值
    calRiskValue() {
      exceptionApi.analysisdataByType(this.month, 'month').then((response) => {
        const chartData = response.data.data

        if (response.data.data.length == 0) {
          return
        }
        // 从对象数组找到该站点所在的元素
        let siteItem = this.findObjectByPropertyValue(chartData, 'name', this.form.name)
        if (Object.keys(siteItem).length === 0) {
          return
        }
        let arr = []
        arr.push(siteItem)
        exceptionApi
          .exceptiondata1({
            siteName: this.form.name,
            beginTime: this.form.beginTime,
            endTime: this.form.endTime
          })
          .then((res) => {
            let table = index.merge(arr, res.data.data, this.form.beginTime, this.form.endTime)
            this.weight = table[0].riskValue
          })
      })
    },

    //  根据目前站点，月份，查折线图日统计数据
    fetchDayAnalysisData() {
      let params = {}
      if (this.form.name) {
        params['siteName'] = this.form.name
      }
      if (this.form.beginTime) {
        params['beginTime'] = this.form.beginTime
      }
      if (this.form.endTime) {
        params['endTime'] = this.form.endTime
      }
      this.screenLoading = true
      this.loading = true
      this.queryButton = true
      exceptionApi
        .analysisdata(this.form.name, this.form.beginTime, this.form.endTime, 'day')
        .then((response) => {
          this.chartData = response.data.data
          this.loading = false
          this.queryButton = false
          if (response.data.data.length == 0) {
            this.isNoData = true
            return
          }
          this.begin = this.chartData[0].lst
          this.end = this.chartData[this.chartData.length - 1].lst

          // 移除空数据状态
          this.isNoData = false
          this.screenLoading = false
          this.setChart()
          //  折线图数据
          let temp = index.calBillData(this.chartData, this.begin, this.end)
          this.bill.min = temp['min']
          this.bill.max = temp['max']

          this.bill.avg = temp['avg']
          this.bill.online = temp['online']
          this.bill.valid = temp['valid']
          this.bill.exceeding = temp['exceeding']
        })
    },
    // 企业异常统计数据
    fetchExceptionAnalysisData() {
      exceptionApi
        .exceptiondata1({
          siteName: this.form.name,
          beginTime: this.form.beginTime,
          endTime: this.form.endTime
        })
        .then((res) => {
          let obj = index.calRecur(res.data.data)
          this.bill.exceptionRecurrence = obj['exceptionRecurrence']
          this.bill.exceptionTypeAggregation = obj['exceptionTypeAggregation']
        })
    },

    // 选择其他值类型时
    setChart() {
      if (this.chartData.length) {
        // x轴日期时间
        let dateList = []
        //颗粒物平均浓度
        let dayAvg = []
        let dataOnline = []
        let dataValid = []
        let dataExceed = []
        this.chartData.forEach((item) => {
          //x轴日期
          dateList.push(item.lst)
          // 历史油烟浓度
          dayAvg.push(item.dayAvg)
          dataOnline.push(item.dayOnline.slice(0, -1))
          dataValid.push(item.dayValid.slice(0, -1))
          dataExceed.push(item.dayExceeding.slice(0, -1))
        })

        this.chartData1 = {
          x: dateList,
          y: dayAvg
        }
        this.chartData2 = {
          x: dateList,
          y: dataOnline
        }
        this.chartData3 = {
          x: dateList,
          y: dataValid
        }
        this.chartData4 = {
          x: dateList,
          y: dataExceed
        }
      }
    }
  }
}
</script>

<template>
  <el-form :inline="true" :model="form">
    <el-form-item class="form-item">
      <AreaAndmonitorType></AreaAndmonitorType>
    </el-form-item>

    <!-- <el-form-item class="form-item">
        <InputSearch
          :site-name="form.name"
          isNeedDefaultSite="0"
          @submit-value="(n) => (form.name = n)"
        ></InputSearch>
      </el-form-item> -->

    <!-- <el-form-item>
        <MonthSelect :month="this.month_1" @submit-value="giveMonth"></MonthSelect>
      </el-form-item> -->

    <!-- <el-form-item>
        <ButtonClick
          content="风险评估"
          type="primary"
          :loading="queryButton"
          @do-search="riskAssessment"
        ></ButtonClick>
      </el-form-item> -->
  </el-form>

  <div v-loading="screenLoading" class="wait-name">
    <div class="chart-container" v-show="!isNoData && !screenLoading">
      <div class="time-text">
        <span>数据统计时段：{{ begin }} ~ {{ end }}</span>
        <span class="site-name">{{ form.name }}</span>
      </div>

      <el-row :gutter="10">
        <el-col :span="14">
          <el-card shadow="never">
            <DustRadarChart
              :name="[
                '数据有效风险',
                '典型异常复现风险',
                '异常类型聚集风险',
                '数据超标风险',
                '数据在线风险'
              ]"
              :yData="[
                bill.valid,
                bill.exceptionRecurrence,
                bill.exceptionTypeAggregation,
                bill.exceeding,
                bill.online
              ]"
            ></DustRadarChart>
          </el-card>
        </el-col>

        <el-col :span="5">
          <el-card shadow="never" class="card-height">
            <template #header>
              <h1
                :class="{
                  'weightColor-low': weight < 0.2,
                  'weightColor-medium': weight >= 0.2 && weight < 0.6,
                  'weightColor-heigh': weight >= 0.6
                }"
              >
                风险值(0~1)：{{ weight }}
              </h1>
            </template>
            <div class="risk-grade">
              <strong>风险等级：</strong>
              <span v-if="weight >= 0.6"> 高风险</span>
              <span v-else-if="weight >= 0.2 && weight < 0.6"> 中风险</span>
              <span v-else> 低风险</span>
            </div>

            <div class="risk-advice">
              <strong>管控建议：</strong>
              <span v-if="weight >= 0.6"> 建议对该站点进行线下执法检查，专项数据对比</span>
              <span v-else-if="weight >= 0.2 && weight < 0.6"> 建议开展常态追踪分析</span>
              <span v-else> 建议引导企业长态保持</span>
            </div>
            <div class="grade-instance">
              <div class="container">
                <div class="block-color heigh"></div>
                <div>高风险(≥0.6)</div>
              </div>
              <div class="container">
                <div class="block-color medium"></div>
                <div>中风险(0.2~0.6)</div>
              </div>
              <div class="container">
                <div class="block-color low"></div>
                <div>低风险(＜0.2)</div>
              </div>
            </div>
          </el-card>
        </el-col>

        <el-col :span="5">
          <el-card shadow="never" class="card-height">
            <template #header><span class="title-16">风险详情</span></template>

            <el-form>
              <el-form-item label="最大值："> {{ bill.max }} mg/m³ </el-form-item>
              <el-form-item label="最小值："> {{ bill.min }} mg/m³ </el-form-item>
              <el-form-item label="数据有效率："> {{ bill.online }}% </el-form-item>
              <el-form-item label="数据在线率："> {{ bill.valid }}% </el-form-item>
              <el-form-item label="数据超标率："> {{ bill.exceeding }}% </el-form-item>
              <el-form-item label="异常类型聚集度：">
                {{ bill.exceptionTypeAggregation * 100 }}%
              </el-form-item>
              <el-form-item label="典型异常复现率：">
                {{ bill.exceptionRecurrence * 100 }}%
              </el-form-item>
            </el-form>
          </el-card>
        </el-col>
      </el-row>

      <el-row :gutter="20">
        <el-col :span="12">
          <el-card shadow="never" class="card-value">
            <FYLineChart title="日均值" :chartData="chartData1" yName="mg/m³" seriesName="日均值">
            </FYLineChart>
          </el-card>
        </el-col>

        <el-col :span="12">
          <el-card shadow="never" class="card-value">
            <FYLineChart title="日在线率" :chartData="chartData2" yName="%" seriesName="日在线率">
            </FYLineChart>
          </el-card>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="12">
          <el-card shadow="never" class="card-value">
            <FYLineChart title="日有效率" :chartData="chartData3" yName="%" seriesName="日有效率">
            </FYLineChart>
          </el-card>
        </el-col>

        <el-col :span="12">
          <el-card shadow="never" class="card-value">
            <FYLineChart title="日超标率" :chartData="chartData4" yName="%" seriesName="日超标率">
            </FYLineChart>
          </el-card>
        </el-col>
      </el-row>
    </div>
  </div>
  <el-empty description="暂无数据" v-show="isNoData" :image-size="200" />
</template>

<style scoped>
.el-form {
  margin: 10px;
}
.form-item {
  margin-top: 10px;
}
.chart-container {
  margin-left: 10px;
}
.time-text {
  font-size: 14px;
  color: #333333;
  letter-spacing: 1px;
}
.el-card {
  margin-top: 15px;
  border-radius: 9px;
}
.chart-container {
  width: 98%;
  /* height: 600px; */
}
.card-value {
  /* min-width:900px; */
  /* padding:0px */
}
.card-height {
  height: 540px;
}
.el-header {
  background-color: #010408;
  color: #333;
  line-height: 60px;
}
.risk-advice {
  margin: 40px 0px;
}
.container {
  display: flex;
  margin-bottom: 10px;
}
.grade-instance {
  margin-top: 80px;
}
.block-color {
  width: 1em;
  height: 1em;
  margin-right: 10px;
  margin-top: 3px;
}
.heigh {
  background-color: red;
}
.medium {
  background-color: #fadc19;
}
.low {
  background-color: #9fdb1d;
}

.el-text {
  align-self: left;
}
.el-form-item {
  margin-bottom: 20px;
}
:deep().el-form-item__content {
  justify-content: flex-end;
}
.title-16 {
  font-size: 16px;
  font-weight: bold;
}
.weightColor-low {
  color: #9fdb1d;
}
.weightColor-medium {
  color: #dabe09;
}
.weightColor-heigh {
  color: red;
}
.risk-grade {
  display: flex;
  margin-top: 10px;
}
:deep().el-table__header-wrapper {
  color: red;
}
/* .wait-name {
      width: 500px;
      height: 600px;
    } */
.table-class {
  /* border: 1px solid blue; */
  /* margin: 20px 0px 20px 0px; */
  height: 540px;
}

.table-button {
  letter-spacing: 1px;
  text-decoration: underline;
  border-radius: 0px;
}
.site-name {
  margin-left: 20px;
}
</style>
