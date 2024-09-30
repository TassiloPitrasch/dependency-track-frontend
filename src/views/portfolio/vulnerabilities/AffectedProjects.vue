<template>
  <div>
    <div id="projectsToolbar" class="bs-table-custom-toolbar">
      <span>
        <multiselect
          id="filterEnhancedStatus"
          v-model="filterEnhancedStatus"
          :options="optionsEnhancedStatus"
          :multiple="true"
          :close-on-select="false"
          :clear-on-select="false"
          :preserve-search="true"
          :searchable="false"
          :size=3
          :allow-empty="false"
          :style="{width: '110%'}"
          v-bind:placeholder="$t('message.filter_enhanced_status')"
          selectLabel
          selectedLabel
          deselectLabel
          track-by="name"
          label="name"
        />
      </span>
    </div>
    <bootstrap-table
      ref="table"
      :columns="columns"
      :data="data"
      :options="options"
      @on-load-success="tableLoaded"
      @on-post-body="onPostBody"
    >
    </bootstrap-table>
  </div>
</template>

<script>
import xssFilters from 'xss-filters';
import permissionsMixin from '../../../mixins/permissionsMixin';
import { Multiselect } from 'vue-multiselect';

export default {
  mixins: [permissionsMixin],
  components: {
    Multiselect
  },
  beforeCreate() {
    try {
      let storedFilter = JSON.parse(localStorage.getItem('ProjectListFilterEnhancedStatus'));
      for (let filter of storedFilter) {
          if (! ["Archived", "In Production", "In Development"].includes(filter.name) || ! ["ARCHIVED", "IN_PRODUCTION", "IN_DEVELOPMENT"].includes(filter.value)) {
             throw new Error();
          }
      }
      this.filterEnhancedStatus = storedFilter;
    }
    catch (error) {
      this.filterEnhancedStatus = [{"name": "In Development", "value": "IN_DEVELOPMENT"}];
      localStorage.setItem('ProjectListFilterEnhancedStatus', JSON.stringify(this.filterEnhancedStatus),);
    }
  },
  props: {
    source: String,
    vulnId: String,
    vulnerability: String,
  },
  data() {
    return {
      filterEnhancedStatus: this.filterEnhancedStatus,
      optionsEnhancedStatus: [
        {name: this.$t('message.in_development'), value: "IN_DEVELOPMENT"},
        {name: this.$t('message.in_production'), value: "IN_PRODUCTION"},
        {name: this.$t('message.archived'), value: "ARCHIVED"}],
      labelIcon: {
        dataOn: '\u2713',
        dataOff: '\u2715',
      },
      columns: [
        {
          title: this.$t('message.name'),
          field: 'name',
          sortable: true,
          formatter: (value, row) => {
            const url = this.$router.resolve({
              name: 'Project Vulnerability Lookup',
              params: { uuid: row.uuid, vulnerability: this.vulnerability },
            }).href;

            let html = `<a href="${url}">${xssFilters.inHTMLData(value)}</a>`;
            if (row.dependencyGraphAvailable) {
              const dependencyGraphUrl = this.$router.resolve({
                name: 'Dependency Graph Component Lookup',
                params: {
                  uuid: row.uuid,
                  componentUuids: row.affectedComponentUuids.join('|'),
                },
              }).href;
              html =
                `<a href="${dependencyGraphUrl}"><i class="fa fa-sitemap" aria-hidden="true" style="float:right; padding-top: 4px; cursor:pointer" data-toggle="tooltip" data-placement="bottom" title="Show in dependency graph"></i></a> ` +
                html;
            }

            return html;
          },
        },
        {
          title: this.$t('message.version'),
          field: 'version',
          sortable: true,
        },
        {
          title: this.$t('message.enhanced_status'),
          field: 'enhancedStatus',
          optionsFunc: () => this.optionsEnhancedStatus, // Injecting $optionsEnhancedStatus directly does not work
          formatter(value, row, index) {
            const optionsEnhancedStatus = this.optionsFunc();
            let enhancedStatus = optionsEnhancedStatus.find(({ value }) => value === row.enhancedStatus);
            return enhancedStatus === undefined ? "-" : enhancedStatus.name;
          },
          sortable: true
        },
      ],
      data: [],
      options: {
        search: true,
        showColumns: true,
        showRefresh: true,
        pagination: true,
        silentSort: false,
        sidePagination: 'client',
        queryParamsType: 'pageSize',
        pageList: '[10, 25, 50, 100]',
        pageSize: 10,
        toolbar: '#projectsToolbar',
        icons: {
          refresh: 'fa-refresh',
        },
        url: this.apiUrl(),
      },
    };
  },
  methods: {
    apiUrl: function () {
      let statusFilterURL = this.filterEnhancedStatus.map(status => 'enhancedStatus=' + status.value).join("&");
      let url = `${this.$api.BASE_URL}/${this.$api.URL_VULNERABILITY}/source/${this.source}/vuln/${encodeURIComponent(this.vulnId)}/projects?${statusFilterURL}`;
      return url;
    },
    tableLoaded: function (array) {
      this.$emit('total', array.length);
    },
    refreshTable: function () {
      this.$refs.table.refresh({
        url: this.apiUrl(),
        silent: true,
        pageNumber: 1,
      });
    },
    onPostBody: function () {
      this.$refs.table.hideLoading();
    },
  },
  watch: {
    filterEnhancedStatus() {
      if (localStorage) {
        localStorage.setItem(
          'ProjectListFilterEnhancedStatus',
           JSON.stringify(this.filterEnhancedStatus),
        );
      }
      this.$refs.table.showLoading();
      this.currentPage = 1;
      this.refreshTable();
    },
    vulnId() {
      // update url when vulnId changes, will trigger table refresh
      this.$refs.table.refreshOptions({ ...this.options, url: this.apiUrl() });
    },
  },
};
</script>
