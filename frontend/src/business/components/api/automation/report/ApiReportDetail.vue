<template>
  <ms-container v-loading="loading">
    <ms-main-container>
      <el-card>
        <section class="report-container" v-if="this.report.testId">

          <ms-api-report-view-header :debug="debug" :report="report" @reportExport="handleExport" @reportSave="handleSave"/>

          <main v-if="this.isNotRunning">
            <ms-metric-chart :content="content" :totalTime="totalTime"/>
            <div>
              <ms-scenario-results :scenarios="content.scenarios" v-on:requestResult="requestResult"/>
            </div>
            <!--<el-collapse-transition>-->
              <!--<div v-show="isActive" style="width: 99%">-->
                <!--<ms-request-result-tail v-if="isRequestResult" :request-type="requestType" :request="request"-->
                                        <!--:scenario-name="scenarioName"/>-->
              <!--</div>-->
            <!--</el-collapse-transition>-->
            <ms-api-report-export v-if="reportExportVisible" id="apiTestReport" :title="report.testName"
                                  :content="content" :total-time="totalTime"/>
          </main>
        </section>
      </el-card>
    </ms-main-container>
  </ms-container>
</template>

<script>

  import MsRequestResult from "./components/RequestResult";
  import MsRequestResultTail from "./components/RequestResultTail";
  import MsScenarioResult from "./components/ScenarioResult";
  import MsMetricChart from "./components/MetricChart";
  import MsScenarioResults from "./components/ScenarioResults";
  import MsContainer from "@/business/components/common/components/MsContainer";
  import MsMainContainer from "@/business/components/common/components/MsMainContainer";
  import MsApiReportExport from "./ApiReportExport";
  import MsApiReportViewHeader from "./ApiReportViewHeader";
  import {RequestFactory} from "../../definition/model/ApiTestModel";
  import {windowPrint} from "@/common/js/utils";

  export default {
    name: "MsApiReport",
    components: {
      MsApiReportViewHeader,
      MsApiReportExport,
      MsMainContainer,
      MsContainer, MsScenarioResults, MsRequestResultTail, MsMetricChart, MsScenarioResult, MsRequestResult
    },
    data() {
      return {
        activeName: "total",
        content: {},
        report: {},
        loading: true,
        fails: [],
        totalTime: 0,
        isRequestResult: false,
        request: {},
        isActive: false,
        scenarioName: null,
        reportExportVisible: false,
        requestType: undefined,
      }
    },
    activated() {
      this.isRequestResult = false;
    },
    props: {
      reportId: String,
      currentProjectId: String,
      infoDb: Boolean,
      debug: Boolean,
    },
    watch: {
      reportId() {
        this.getReport();
      }
    },
    methods: {
      init() {
        this.loading = true;
        this.report = {};
        this.content = {};
        this.fails = [];
        this.report = {};
        this.isRequestResult = false;
      },
      handleClick(tab, event) {
        this.isRequestResult = false
      },
      active() {
        this.isActive = !this.isActive;
      },
      getReport() {
        this.init();
        if (this.reportId) {
          let url = "/api/scenario/report/get/" + this.reportId + "/" + this.infoDb;
          this.$get(url, response => {
            this.report = response.data || {};
            if (response.data) {
              if (this.isNotRunning) {
                try {
                  this.content = JSON.parse(this.report.content);
                  this.$emit('refresh');
                } catch (e) {
                  throw e;
                }
                this.getFails();
                this.loading = false;
              } else {
                setTimeout(this.getReport, 2000)
              }
            } else {
              this.loading = false;
              this.$error(this.$t('api_report.not_exist'));
            }
          });
        }
      },
      getFails() {
        if (this.isNotRunning) {
          this.fails = [];
          this.totalTime = 0
          this.content.scenarios.forEach((scenario) => {
            this.totalTime = this.totalTime + Number(scenario.responseTime)
            let failScenario = Object.assign({}, scenario);
            if (scenario.error > 0) {
              this.fails.push(failScenario);
              failScenario.requestResults = [];
              scenario.requestResults.forEach((request) => {
                if (!request.success) {
                  let failRequest = Object.assign({}, request);
                  failScenario.requestResults.push(failRequest);
                }
              })

            }
          })
        }
      },
      requestResult(requestResult) {
        this.active();
        this.isRequestResult = false;
        this.requestType = undefined;
        if (requestResult.request.body.indexOf('[Callable Statement]') > -1) {
          this.requestType = RequestFactory.TYPES.SQL;
        }
        this.$nextTick(function () {
          this.isRequestResult = true;
          this.request = requestResult.request;
          this.scenarioName = requestResult.scenarioName;
        });
      },
      handleExport() {
        this.reportExportVisible = true;
        let reset = this.exportReportReset;
        this.$nextTick(() => {
          windowPrint('apiTestReport', 0.57);
          reset();
        });
      },
      handleSave() {
        if (!this.report.name) {
          this.$warning(this.$t('api_test.automation.report_name_info'));
          return;
        }
        if (!this.currentProjectId) {
          this.$warning(this.$t('api_test.select_project'));
          return;
        }
        this.loading = true;
        this.report.projectId = this.currentProjectId;
        let url = "/api/scenario/report/add";
        if (this.infoDb === true) {
          url = "/api/scenario/report/update";
        }
        this.result = this.$post(url, this.report, response => {
          this.$success(this.$t('commons.save_success'));
          this.loading = false;
          this.$emit('refresh');
        }, error => {
          this.loading = false;
        });
      },
      exportReportReset() {
        this.$router.go(0);
      }
    },

    created() {
      this.getReport();
    },

    computed: {
      path() {
        return "/api/test/edit?id=" + this.report.testId;
      },
      isNotRunning() {
        return "Running" !== this.report.status;
      }
    }
  }
</script>
<style>
  .report-container .el-tabs__header {
    margin-bottom: 1px;
  }
</style>

<style scoped>

  .report-container {
    height: calc(100vh - 155px);
    min-height: 600px;
    overflow-y: auto;
  }

  .report-header {
    font-size: 15px;
  }

  .report-header a {
    text-decoration: none;
  }

  .report-header .time {
    color: #909399;
    margin-left: 10px;
  }

  .report-container .fail {
    color: #F56C6C;
  }

  .report-container .is-active .fail {
    color: inherit;
  }

  .export-button {
    float: right;
  }

  .scenario-result .icon.is-active {
    transform: rotate(90deg);
  }

</style>
