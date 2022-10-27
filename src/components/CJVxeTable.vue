<template>
  <div
    :style="{ padding: padding + 'px' }"
    class="flex-column"
    :class="{ 'scroll-bold': scrollBold }"
  >
    <div
      v-if="$slots['bar-left'] || $slots['bar-right']"
      class="f-mgb clearfix"
    >
      <div class="f-fl">
        <slot name="bar-left" />
      </div>
      <div class="f-fr">
        <slot name="bar-right" />
      </div>
    </div>

    <div class="flex-item f-oh">
      <vxe-table
        ref="xTable"
        :scroll-y="scrollY"
        :row-key="rowKey"
        v-bind="vxeTableConfig"
        v-on="$listeners"
      >
        <slot />
      </vxe-table>
    </div>
    <p class="f-mgt-10 asyncPageLoading" v-if="asyncPageLoading">
      <Icon type="ios-loading" size="14" class="ani-rotate"></Icon>
      分页数据加载中...
    </p>
    <vxe-pager
      v-else
      ref="xPage"
      class="f-mgt-10"
      :current-page="pageIndex"
      :page-size="pageSize"
      :total="total"
      @page-change="handlePageChange"
    />
    <!-- v-if="!hidePager" -->
    <!-- :page-sizes="[2]" -->
  </div>
</template>

<script>
import { removeClass } from "@/libs/tools";
export default {
  name: "CJVxeTable",
  props: {
    observe: {
      type: Boolean,
      default: true,
    },
    config: {
      type: Object,
      default: () => ({}),
    },
    checkboxConfig: {
      type: Object,
      default: () => ({}),
    },
    tooltipConfig: {
      type: Object,
      default: () => ({}),
    },
    scrollY: {
      type: Object,
      default: () => ({}),
    },
    renderApi: {
      type: Function,
      required: true,
    },
    renderCountApi: {
      type: Function,
      default: () => {},
    },
    renderParams: {
      type: Object,
      default: () => ({}),
    },
    padding: {
      type: Number,
      default: 15,
    },
    defaultPageSize: {
      type: Number,
      default: 20,
    },
    scrollBold: {
      type: Boolean,
      default: true,
    },
    rowKey: {
      type: Boolean,
      default: false,
    },
    afterLoadData: {
      type: Function,
    },
  },
  data() {
    const { defaultPageSize } = this;
    return {
      tableData: [],
      tableLoading: false,
      asyncPageLoading: false, //异步分页loading
      pageIndex: 1,
      pageSize: defaultPageSize || 20,
      total: 0,
      tableVm: null,
      pageVm: null,
      // firstLoad: true
    };
  },
  computed: {
    vxeTableConfig() {
      const { config, tableLoading, tableData, checkboxConfig, tooltipConfig } =
        this;
      const extendConfig = {
        checkboxConfig: { ...(config.checkboxConfig || {}), ...checkboxConfig },
        tooltipConfig: { ...(config.tooltipConfig || {}), ...tooltipConfig },
      };
      return {
        height: "100%",
        showOverflow: true,
        autoResize: true,
        loading: tableLoading,
        data: tableData,
        ...config,
        ...extendConfig,
      };
    },
  },
  watch: {
    $route(val) {
      this.$nextTick(() => {
        const tips =
          document.querySelectorAll(".vxe-table--tooltip-wrapper") || [];
        tips.forEach((item) => {
          if (item.className.indexOf("is--visible") > -1) {
            removeClass.call(item, "is--visible");
          }
        });
      });
    },
    renderParams: {
      immediate: true,
      handler() {
        if (!this.observe) return;
        this.tableData = [];
        this.pageIndex = 1;
        this.$nextTick(() => this.renderTable());
      },
    },
  },
  methods: {
    handlePageChange(pageInfo) {
      const { currentPage, pageSize, type } = pageInfo;
      if (type === "size") {
        this.pageSize = pageSize;
        this.pageIndex = 1;
      } else {
        this.pageIndex = currentPage;
      }

      this.renderTable(pageInfo);
    },
    // 异步分页数据
    async renderPage() {
      if (!this.renderParams.isAsync) return;
      this.asyncPageLoading = true;
      const params = {
        ...this.renderParams,
        pageIndex: this.pageIndex,
        pageSize: this.pageSize,
      };
      const {
        data: { count },
      } = await this.renderCountApi(params);
      this.total = count;
      this.asyncPageLoading = false;
    },
    async renderTable(pageInfo = {}) {
      this.tableLoading = true;

      let { pageIndex, pageSize } = pageInfo;

      if (pageIndex) {
        this.pageIndex = pageIndex;
      } else {
        pageIndex = this.pageIndex;
      }

      if (pageSize) {
        this.pageSize = pageSize;
      } else {
        pageSize = this.pageSize;
      }
      const params = {
        ...this.renderParams,
        pageIndex,
        pageSize,
      };
      //  分页切换不调用 renderPage
      if (!("type" in pageInfo)) {
        // 在 renderApi 前调用，体验会好很多
        this.renderPage();
      }
      const { code, data } = await this.renderApi(params);
      // 获取数据失败，提示用户刷新页面
      if (code !== 0 || !data) {
        // this.$Message.error('数据获取异常，请重新加载页面')
        return;
      }
      const { list, count } = data;
      this.tableData = list;
      // 异步分页的total不在这里赋值
      if (!this.renderParams.isAsync) {
        this.total = count;
      }
      if (this.afterLoadData instanceof Function) {
        this.afterLoadData(list);
      }
      // 滚动回表格顶部
      if (this.$refs.xTable.clearScroll) {
        this.$refs.xTable.clearScroll().then(() => {
          const el = this.$refs.xTable.$el.querySelector(
            ".vxe-table--body-wrapper"
          );
          if (el) {
            el.scrollTop = 0;
          }
        });
      }

      this.tableLoading = false;
    },
  },
  mounted() {
    this.tableVm = this.$refs.xTable;
    this.pageVm = this.$refs.xPage;
  },
};
</script>
<style lang="scss" scoped>
.scroll-bold {
  & ::v-deep {
    ::-webkit-scrollbar-thumb {
      border-radius: 0px;
    }
    ::-webkit-scrollbar {
      width: 5px;
      height: 10px;
    }
    ::-webkit-scrollbar-track {
      border-radius: 0px;
    }
  }
}
.asyncPageLoading {
  text-align: right;
  font-size: 12px;
  color: #606266;
}
</style>
