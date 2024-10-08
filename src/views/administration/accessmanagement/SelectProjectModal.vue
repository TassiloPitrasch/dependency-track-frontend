<template>
  <b-modal
    id="selectProjectModal"
    size="lg"
    hide-header-close
    no-stacking
    :title="$t('admin.select_project')"
  >
    <div id="projectsToolbar" class="bs-table-custom-toolbar">
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
          :style="{ width: '110%' }"
          v-bind:placeholder="$t('message.filter_enhanced_status')"
          selectLabel
          selectedLabel
          deselectLabel
          track-by="name"
          label="name"
        />
    </div>
    <bootstrap-table
      ref="table"
      :columns="columns"
      :data="data"
      :options="options"
    >
    </bootstrap-table>
    <template v-slot:modal-footer="{ cancel }">
      <b-button size="md" variant="secondary" @click="cancel()">{{
        $t('message.cancel')
      }}</b-button>
      <b-button
        size="md"
        variant="primary"
        @click="$emit('selection', $refs.table.getSelections())"
        >{{ $t('message.select') }}</b-button
      >
    </template>
  </b-modal>
</template>

<script>
import xssFilters from 'xss-filters';
import permissionsMixin from '../../../mixins/permissionsMixin';
import common from '../../../shared/common';
import { Switch as cSwitch } from '@coreui/vue';
import { Multiselect } from 'vue-multiselect';
import router from '@/router';

export default {
  mixins: [permissionsMixin],
  components: {
    cSwitch,
    Multiselect
  },
  props: {
    teamUuid: String,
  },
  beforeCreate () {
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
          field: 'state',
          checkbox: true,
          align: 'center',
        },
        {
          title: this.$t('message.project_name'),
          field: 'name',
          sortable: true,
          formatter(value, row) {
            // TODO: Close modal when link is clicked.
            const href = router.resolve({
              name: 'Project',
              params: { uuid: row.uuid },
            }).href;
            return `<a href="${href}">${xssFilters.inHTMLData(value)}</a>`;
          },
        },
        {
          title: this.$t('message.version'),
          field: 'version',
          sortable: true,
          formatter(value) {
            return xssFilters.inHTMLData(common.valueWithDefault(value, ''));
          },
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
        sidePagination: 'server',
        queryParamsType: 'pageSize',
        pageList: '[10, 25, 50, 100]',
        pageSize: 10,
        icons: {
          refresh: 'fa-refresh',
        },
        toolbar: '#projectsToolbar',
        responseHandler: function (res, xhr) {
          res.total = xhr.getResponseHeader('X-Total-Count');
          return res;
        },
        url: this.apiUrl(),
      },
    };
  },
  methods: {
    apiUrl: function () {
      let url = `${this.$api.BASE_URL}/${this.$api.URL_PROJECT}?notAssignedToTeamWithUuid=${this.teamUuid}`;
      this.filterEnhancedStatus.forEach(status => url += ('&enhancedStatus=' + status.value));
      return url;
    },
    refreshTable: function () {
      this.$refs.table.refresh({
        url: this.apiUrl(),
        silent: true,
      });
    },
  },
  watch: {
    // TODO: Might need a refresh/reload of stored options if shown again, as filter can be changed in the modals of other projects
    filterEnhancedStatus() {
      if (localStorage) {
        localStorage.setItem(
          'ProjectListFilterEnhancedStatus',
           JSON.stringify(this.filterEnhancedStatus),
        );
      }
      this.refreshTable();
    },
  },
};
</script>
