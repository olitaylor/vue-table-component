<template>
    <div class="table-component">
        <div v-if="showFilter && filterableColumnExists" class="table-component__filter">
            <input
                    :class="fullFilterInputClass"
                    type="text"
                    v-model="filter"
                    :placeholder="filterPlaceholder"
            >
            <a
                    v-if="filter"
                    @click="filter = ''"
                    class="table-component__filter__clear"
            >×</a>
        </div>

        <div class="table-component__table-wrapper table-responsive">
            <table :class="fullTableClass">
                <caption v-if="showCaption" class="table-component__table__caption" role="alert" aria-live="polite">
                    {{ ariaCaption }}
                </caption>
                <thead>
                <tr class="bg-blue">
                    <table-column-header
                            @click="changeSorting"
                            v-for="column in columns"
                            :key="column.show"
                            :sort="sort"
                            :column="column"
                    ></table-column-header>
                </tr>
                </thead>
                <tbody>
                <table-row
                        v-for="row in displayedRows"
                        :key="row.vueTableComponentInternalRowId"
                        :row="row"
                        :columns="columns"
                ></table-row>
                </tbody>
            </table>
        </div>

        <div v-if="displayedRows.length === 0" class="table-component__message">
            {{ filterNoResults }}
        </div>


        <div style="display:none;">
            <slot></slot>
        </div>

        <pagination v-if="pagination" :pagination="pagination" @pageChange="pageChange"></pagination>
    </div>
</template>

<script>
    import Column from '../classes/Column';
    import expiringStorage from '../expiring-storage';
    import Row from '../classes/Row';
    import TableColumnHeader from './TableColumnHeader';
    import TableRow from './TableRow';
    import settings from '../settings';
    import isArray from 'lodash/isArray';
    import pick from 'lodash/pick';
    import Pagination from './Pagination';
    import { classList } from '../helpers';

    export default {
        components: {
            TableColumnHeader,
            TableRow,
            Pagination,
        },

        props: {
            data: {default: () => [], type: [Array, Function]},

            showFilter: { default: true },
            showCaption: { default: true },

            sortBy: { default: '', type: String },
            sortOrder: { default: '', type: String },

            cacheKey: { default: null },
            cacheLifetime: { default: 5 },

            tableClass: { default: settings.tableClass },
            filterInputClass: { default: settings.filterInputClass },
            filterPlaceholder: { default: settings.filterPlaceholder },
            filterNoResults: { default: settings.filterNoResults },
        },

        data: () => ({
            columns: [],
            rows: [],
            filter: '',
            sort: {
                fieldName: '',
                order: '',
            },
            pagination: null,

            localSettings: {},
        }),

        created() {
            this.sort.fieldName = this.sortBy;
            this.sort.order = this.sortOrder;

            this.restoreState();
        },

        async mounted() {
            this.mapColumns();

            await this.mapDataToRows();
        },

        watch: {
            filter() {
                if (!this.usesLocalData) {
                    this.mapDataToRows();
                }

                this.saveState();
            },

            data() {
                if (this.usesLocalData) {
                    this.mapDataToRows();
                }
            },
        },

        computed: {
            fullTableClass() {
                return classList('table-component__table', this.tableClass);
            },

            fullFilterInputClass() {
                return classList('table-component__filter__field', this.filterInputClass);
            },

            ariaCaption() {
                if (this.sort.fieldName === '') {
                    return 'Table not sorted';
                }

                return `Table sorted by ${this.sort.fieldName} ` +
                    (this.sort.order === 'asc' ? '(ascending)' : '(descending)');
            },

            usesLocalData() {
                return isArray(this.data);
            },

            displayedRows() {
                if (!this.usesLocalData) {
                    return this.sortedRows;
                }

                if (!this.showFilter) {
                    return this.sortedRows;
                }

                if (!this.columns.filter(column => column.isFilterable()).length) {
                    return this.sortedRows;
                }

                return this.sortedRows.filter(row => row.passesFilter(this.filter));
            },

            sortedRows() {
                if (!this.usesLocalData) {
                    return this.rows;
                }

                if (this.sort.fieldName === '') {
                    return this.rows;
                }

                if (this.columns.length === 0) {
                    return this.rows;
                }

                const sortColumn = this.getColumn(this.sort.fieldName);

                if (!sortColumn) {
                    return this.rows;
                }

                return this.rows.sort(sortColumn.getSortPredicate(this.sort.order, this.columns));
            },

            filterableColumnExists() {
                return this.columns.filter(c => c.isFilterable()).length > 0;
            },

            storageKey() {
                return this.cacheKey
                    ? `vue-table-component.${this.cacheKey}`
                    : `vue-table-component.${window.location.host}${window.location.pathname}${this.cacheKey}`;
            },
        },

        methods: {
            async pageChange(page) {
                this.pagination.currentPage = page;

                await this.mapDataToRows();
            },

            async mapDataToRows() {
                const data = this.usesLocalData
                    ? this.prepareLocalData()
                    : await this.fetchServerData();

                let rowId = 0;

                this.rows = data
                    .map(rowData => {
                        rowData.vueTableComponentInternalRowId = rowId++;
                        return rowData;
                    })
                    .map(rowData => new Row(rowData, this.columns));
            },

            mapColumns() {
                const columnComponents = this.$slots.default.filter(
                    column => column.componentInstance
                );

                this.columns = columnComponents.map(
                    column => new Column(column.componentInstance)
                );

                columnComponents.forEach(column => {
                    Object.keys(this.$options.props).forEach(
                         prop => this.$watch(prop, () => this.mapColumns())
                    );
                });
            },

            prepareLocalData() {
                this.pagination = null;

                return this.data;
            },

            async fetchServerData() {
                const page = this.pagination && this.pagination.currentPage || 1;

                const response = await this.data({
                    filter: this.filter,
                    sort: this.sort,
                    page: page,
                });

                this.pagination = response.pagination;

                return response.data;
            },

            async refresh() {
                await this.mapDataToRows();
            },

            changeSorting(column) {
                if (this.sort.fieldName !== column.show) {
                    this.sort.fieldName = column.show;
                    this.sort.order = 'asc';
                } else {
                    this.sort.order = (this.sort.order === 'asc' ? 'desc' : 'asc');
                }

                if (!this.usesLocalData) {
                    this.mapDataToRows();
                }

                this.saveState();
            },

            getColumn(columnName) {
                return this.columns.find(column => column.show === columnName);
            },

            saveState() {
                expiringStorage.set(this.storageKey, pick(this.$data, ['filter', 'sort']), this.cacheLifetime);
            },

            restoreState() {
                const previousState = expiringStorage.get(this.storageKey);

                if (previousState === null) {
                    return;
                }

                this.sort = previousState.sort;
                this.filter = previousState.filter;

                this.saveState();
            },
        },
    };
</script>
<style>
.table-component {
  display: flex;
  flex-direction: column;
  margin: 4em 0;
}

.table-component__filter {
  align-self: flex-end;
}

.table-component__filter__field {
  padding: 0 1.25em 0 .75em;
  height: 2.5em;
  border: solid 2px #e0e0e0;
  border-radius: 2em;
  font-size: inherit;
}

.table-component__filter__clear {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 2em;
  color: #007593;
  font-weight: bold;
  cursor: pointer;
}

.table-component__filter__field:focus {
  outline: 0;
  border-color: #007593;
}

.table-component__table-wrapper {
  overflow-x: auto;
  margin: 1em 0;
  width: 100%;
  border: solid 1px #ddd;
  border-bottom: none;
}

.table-component__table {
  min-width: 100%;
  border-collapse: collapse;
  border-bottom: solid 1px #ddd;
  table-layout: fixed;
}

.table-component__table__caption {
  position: absolute;
  top: auto;
  left: -10000px;
  overflow: hidden;
  width: 1px;
  height: 1px;
}

.table-component__table th,
.table-component__table td {
  padding: .75em 1.25em;
  vertical-align: top;
  text-align: left;
}

.table-component__table th {
  background-color: #e0e0e0;
  color: #999;
  text-transform: uppercase;
  white-space: nowrap;
  font-size: .85em;
}

.table-component__table tbody tr:nth-child(even) {
  background-color: #f0f0f0;
}

.table-component__table a {
  color: #007593;
}

.table-component__message {
  color: #999;
  font-style: italic;
}

.table-component__th--sort,
.table-component__th--sort-asc,
.table-component__th--sort-desc {
  text-decoration: underline;
  cursor: pointer;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

.table-component__th--sort-asc:after,
.table-component__th--sort-desc:after {
  position: absolute;
  left: .25em;
  display: inline-block;
  color: #bbb;
}

.table-component__th--sort-asc:after {
  content: '↑';
}

.table-component__th--sort-desc:after {
  content: '↓';
}
</style>
