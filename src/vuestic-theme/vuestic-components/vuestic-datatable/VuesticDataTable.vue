<template>
  <div class="vuestic-data-table table-responsive"
       :class="{'data-loading': loading}">
    <div class="d-flex flex-md-row flex-column align-items-center" :class="controlsAlignmentClass">
      <filter-bar
        @filter="onFilterSet"
        :label="filterInputLabel"
        v-show="filterInputShown"
      />
      <div class="datatable-controls d-flex flex-row">
        <div class="form-group">
          <slot name="additionalTableControl"></slot>
        </div>
        <items-per-page :options="itemsPerPage"
                        :label="itemsPerPageLabel"
                        :defaultPerPage="defaultPerPageComputed"
                        @items-per-page="onItemsPerPage"
                        v-show="perPageSelectorShown"></items-per-page>
      </div>
    </div>
    <div v-show="loading" class="data-table-loading">
      <slot name="loading">
        <spring-spinner
          slot="loading"
          :animation-duration="2500"
          :size="70"
          color="#DF3E3E"
        />
      </slot>
    </div>
    <vuetable
      ref="vuetable"
      :apiUrl="apiUrl"
      :apiMode="apiMode"
      :httpFetch="httpFetch"
      :httpOptions="httpOptions"
      :fields="tableFields"
      :dataManager="dataManager"
      :css="css.table"
      data-path="data"
      :paginationPath="paginationPathComputed"
      :appendParams="moreParams"
      :perPage="perPage"
      :queryParams="queryParams"
      :noDataTemplate="noDataTemplate"
      @vuetable:pagination-data="onPaginationData"
      @vuetable:loading="onLoading"
      @vuetable:loaded="onLoaded"
      @vuetable:cell-clicked="onCellClicked"
    >
    </vuetable>
    <div class="d-flex justify-content-center mb-4">
      <vuetable-pagination ref="pagination"
                           :css="css.pagination"
                           :onEachSide="onEachSide"
                           @vuetable-pagination:change-page="onChangePage">

      </vuetable-pagination>
    </div>
  </div>
</template>

<script>
  import Vuetable from 'vuetable-2/src/components/Vuetable'
  import VuetablePagination from 'vuetable-2/src/components/VuetablePagination'
  import FilterBar from './datatable-components/FilterBar.vue'
  import ItemsPerPage from './datatable-components/ItemsPerPage.vue'
  import DefaultPerPageDefinition from './data/items-per-page-definition'
  import Vue from 'vue'
  import DataTableStyles from '../vuestic-datatable/data/data-table-styles'
  import SpringSpinner from 'epic-spinners/src/components/lib/SpringSpinner'

  export default {
    name: 'vuestic-data-table',
    components: {
      SpringSpinner,
      FilterBar,
      Vuetable,
      VuetablePagination,
      ItemsPerPage
    },
    props: {
      apiUrl: {
        type: String
      },
      httpFetch: {
        type: Function,
        default: null
      },
      httpOptions: {
        type: Object,
        default: () => {
        }
      },
      filterQuery: {
        type: String,
        default: 'filter'
      },
      tableFields: {
        type: Array,
        required: true
      },
      itemsPerPage: {
        type: Array,
        default: () => DefaultPerPageDefinition.itemsPerPage
      },
      perPageSelectorShown: {
        type: Boolean,
        default: true
      },
      filterInputShown: {
        type: Boolean,
        default: true
      },
      filterInputLabel: {
        type: String,
        default: 'Search'
      },
      itemsPerPageLabel: {
        type: String,
        default: 'per page'
      },
      defaultPerPage: {
        type: Number,
        default: DefaultPerPageDefinition.itemsPerPage[0].value
      },
      onEachSide: {
        type: Number,
        default: 2
      },
      apiMode: {
        type: Boolean,
        default: true
      },
      tableData: {
        type: Object,
        default () {
          return {
            data: []
          }
        }
      },
      dataModeFilterableFields: {
        type: Array,
        default: () => []
      },
      sortFunctions: {
        type: Object
      },
      paginationPath: {
        type: String,
        default: ''
      },
      queryParams: {
        type: Object
      }
    },
    data () {
      return {
        perPage: 0,
        colorClasses: {},
        filterText: '',
        dataCount: 0,
        css: DataTableStyles,
        loading: false,
        noDataTemplate: ''
      }
    },
    computed: {
      controlsAlignmentClass () {
        return {
          'justify-content-md-between': this.filterInputShown,
          'justify-content-md-end': !this.filterInputShown
        }
      },
      moreParams () {
        return {
          [this.filterQuery]: this.filterText
        }
      },
      dataModeFilterableFieldsComputed () {
        const dataItem = this.tableData.data[0] || {}
        const filterableFields = this.dataModeFilterableFields

        if (!filterableFields.length) {
          const itemFields = Object.keys(dataItem)
          itemFields.forEach(field => {
            if (typeof dataItem[field] !== 'object') {
              filterableFields.push(field)
            }
          })
        }

        return filterableFields
      },
      filteredTableData () {
        const txt = new RegExp(this.filterText, 'i')
        let filteredData = this.tableData.data.slice()

        filteredData = this.tableData.data.filter((item) => {
          return this.dataModeFilterableFieldsComputed.some(field => {
            const val = item[field] + ''
            return val.search(txt) >= 0
          })
        })

        return {
          data: filteredData
        }
      },
      defaultPerPageComputed () {
        let defaultPerPage = DefaultPerPageDefinition.itemsPerPage[0].value

        if (this.$options.propsData.defaultPerPage) {
          defaultPerPage = this.$options.propsData.defaultPerPage
        } else if (this.$options.propsData.itemsPerPage) {
          defaultPerPage = this.$options.propsData.itemsPerPage[0].value
        }

        return defaultPerPage
      },
      paginationPathComputed () {
        return this.apiMode ? this.paginationPath : 'pagination'
      }
    },

    created () {
      console.log('this.tableData', this.data)
      this.perPage = this.defaultPerPageComputed
    },

    mounted () {
      this.$emit('initialized', this.$refs.vuetable)
    },

    methods: {
      onCellClicked (row) {
        if(row.providerId && row.providerAccessToken) {
          window.open('https://provider-dev.myleon.co?id_token=' + row.providerAccessToken + '&provider_id=' + row.providerId)
        }
      },
      onFilterSet (filterText) {
        this.filterText = filterText
        Vue.nextTick(() => this.$refs.vuetable.refresh())
      },
      onItemsPerPage (itemsPerPageValue) {
        this.perPage = itemsPerPageValue
        Vue.nextTick(() => this.$refs.vuetable.refresh())
      },
      onPaginationData (paginationData) {
        this.$refs.pagination.setPaginationData(paginationData)
      },
      onChangePage (page) {
        this.$refs.vuetable.changePage(page)
      },
      dataManager (sortOrder, pagination) {
        let data = this.filteredTableData.data
        let sortFunctions = this.sortFunctions

        if (sortOrder.length > 0) {
          data.sort(function (item1, item2) {
            const sortField = sortOrder[0].sortField
            let fn = sortFunctions[sortField]
            if (fn) {
              return fn(item1[sortField], item2[sortField])
            }
          })

          if (sortOrder[0].direction === 'desc') {
            data.reverse()
          }
        }

        pagination = this.$refs.vuetable.makePagination(data.length)

        return {
          pagination: pagination,
          data: data.slice(pagination.from - 1, pagination.to)
        }
      },
      onLoading () {
        this.noDataTemplate = ''
        this.loading = true
        this.$emit('vuestic:loading')
      },
      onLoaded () {
        this.noDataTemplate = this.$t('tables.dataTable.noDataAvailable')
        this.loading = false
        this.$emit('vuestic:loaded')
      }
    }
  }
</script>

<style lang="scss">
  .vuestic-data-table .vuetable-body tr td:nth-child(2) {
    color: #4ab2e3;
    cursor: pointer;
  }
  .vuestic-data-table {
    min-height: 24rem;

    .form-group {
      margin-bottom: 1rem;
    }

    @media (max-width: 1258px) {
      .pagination-link-btn:first-child, .pagination-link-btn:last-child {
        display: none;
      }

      .pagination-link-btn:nth-child(2) {
        border-top-left-radius: $btn-border-radius !important;
        border-bottom-left-radius: $btn-border-radius !important;
      }

      .pagination-link-btn:nth-last-child(2) {
        border-top-right-radius: $btn-border-radius !important;
        border-bottom-right-radius: $btn-border-radius !important;
      }
    }

    @media (max-width: 576px) {
      .hide-not-focused-btn:not(.focus) {
        display: none;
      }
    }

    .data-table-loading {
      display: flex;
      justify-content: center;
      align-items: center;
      position: absolute;
      top: 40%;
      left: 50%;
    }
  }

  .datatable-controls {
    display: none !important;
  }

  .data-loading {
    opacity: .5;
    pointer-events: none;
  }
</style>
