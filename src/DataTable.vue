<template>
	<div class="card material-table">
		<div class="table-header">
			<span class="table-title">{{title}}</span>
			<div class="actions">
				<a v-for="button in customButtons" href="javascript:undefined"
				   class="waves-effect btn-flat nopadding"
				   v-if="button.hide ? !button.hide : true"
				   @click="button.onclick">
					<i class="material-icons">{{button.icon}}</i>
				</a>
				<a href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					v-if="this.printable"
					@click="print">
					<i class="material-icons">print</i>
				</a>
				<a href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					v-if="this.exportable"
					@click="exportExcel">
					<i class="material-icons">description</i>
				</a>
				<a href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					v-if="this.searchable"
					@click="search">
					<i class="material-icons">search</i>
				</a>
			</div>
		</div>
		<div v-if="this.searching">
			<div id="search-input-container">
				<label>
					<input type="search" id="search-input" class="form-control" placeholder="Search data"
						:value="searchInput"
						@input="(e) => {this.searchInput = e.target.value}">
				</label>
			</div>
		</div>
		<table ref="table" :class="computedClass">
			<thead>
				<tr>
					<th v-for="(column, index) in columns"
						@click="sort(index)"
						:class="(sortable ? 'sorting ' : '')
							+ (sortColumn === index ?
								(sortType === 'desc' ? 'sorting-desc' : 'sorting-asc')
								: '') + (column.icon ? 'column-icon' : '' )
							+ (column.numeric ? ' numeric' : '')"
						:style="{width: column.width ? column.width : 'auto'}"
						:data-tooltip='column.label'>
						<div>
							<i class="material-icons" v-if="column.icon">{{column.icon}}</i>
							{{column.label}}
						</div>
					</th>
					<slot name="thead-tr"></slot>
				</tr>
			</thead>

			<tbody>
				<tr v-for="(row, index) in paginated" :class="{ clickable : clickable }" @click="click(row)">
					<td v-for="column in columns" :class=" {numeric : column.numeric }">
						<div v-if="!column.html">
              <span class="md-icon material-icons" :class="addCustomClass(row, column.field)" v-if="column.customIcon"></span>
              <span>{{ collect(row, column.field) }}</span>
            </div>
						<div v-if="column.html" v-html="collect(row, column.field)"></div>
					</td>
					<slot name="tbody-tr" :row="row"></slot>
				</tr>
			</tbody>
		</table>

		<div class="table-footer" v-if="paginate">
			<div class="datatable-length">
				<label>
					<span>Rows per page:</span>
					<select class="browser-default" @change="onTableLength">
						<option v-for="option in perPageOptions" :value="option" :selected="option == currentPerPage">
					    {{ option === -1 ? 'All' : option }}
					  </option>
					</select>
				</label>
			</div>
			<div class="datatable-info">
				{{(currentPage - 1) * currentPerPage ? (currentPage - 1) * currentPerPage : 1}}
					-{{Math.min(processedRows.length, currentPerPage * currentPage)}} of {{processedRows.length}}
			</div>
			<div>
				<ul class="material-pagination">
					<li>
						<a href="javascript:undefined" class="waves-effect btn-flat" @click.prevent="previousPage" tabindex="0">
							<i class="material-icons">chevron_left</i>
						</a>
					</li>
					<li>
						<a href="javascript:undefined" class="waves-effect btn-flat" @click.prevent="nextPage" tabindex="0">
							<i class="material-icons">chevron_right</i>
						</a>
					</li>
				</ul>
			</div>
		</div>
	</div>
</template>

<script>
import Fuse from "fuse.js";

export default {
  props: {
    title: "",
    columns: {},
    rows: {},
    clickable: { default: true },
    customButtons: { default: () => [] },
    perPage: { default: () => [10, 20, 30, 40, 50] },
    defaultPerPage: { default: null },
    sortable: { default: true },
    searchable: { default: true },
    exactSearch: {
      type: Boolean,
      default: false
    },
    paginate: { default: true },
    exportable: { default: true },
    printable: { default: true }
  },

  data: () => ({
    currentPage: 1,
    currentPerPage: 10,
    sortColumn: -1,
    sortType: "asc",
    searching: false,
    searchInput: ""
  }),

  methods: {
    nextPage: function() {
      if (this.processedRows.length > this.currentPerPage * this.currentPage)
        ++this.currentPage;
    },

    previousPage: function() {
      if (this.currentPage > 1) --this.currentPage;
    },

    onTableLength: function(e) {
      this.currentPerPage = e.target.value;
    },

    sort: function(index) {
      if (!this.sortable) return;
      if (this.sortColumn === index) {
        this.sortType = this.sortType === "asc" ? "desc" : "asc";
      } else {
        this.sortType = "asc";
        this.sortColumn = index;
      }
    },

    search: function(e) {
      this.searching = !this.searching;
    },

    click: function(row) {
      if (!this.clickable) {
        return;
      }

      if (getSelection().toString()) {
        // Return if some text is selected instead of firing the row-click event.
        return;
      }

      this.$emit("row-click", row);
    },

    exportExcel: function() {
      const mimeType = "data:application/vnd.ms-excel";
      const html = this.renderTable().replace(/ /g, "%20");

      const documentPrefix =
        this.title != "" ? this.title.replace(/ /g, "-") : "Sheet";
      const d = new Date();

      var dummy = document.createElement("a");
      dummy.href = mimeType + ", " + html;
      dummy.download =
        documentPrefix +
        "-" +
        d.getFullYear() +
        "-" +
        (d.getMonth() + 1) +
        "-" +
        d.getDate() +
        "-" +
        d.getHours() +
        "-" +
        d.getMinutes() +
        "-" +
        d.getSeconds() +
        ".xls";
      dummy.click();
    },

    print: function() {
      let win = window.open("");
      win.document.write(this.renderTable());
      win.print();
      win.close();
    },

    renderTable: function() {
      var table = "<table><thead>";

      table += "<tr>";
      for (var i = 0; i < this.columns.length; i++) {
        const column = this.columns[i];
        table += "<th>";
        table += column.label;
        table += "</th>";
      }
      table += "</tr>";

      table += "</thead><tbody>";

      for (var i = 0; i < this.rows.length; i++) {
        console.group("%cROWS ", "background:#DAF7A6;");
        const row = this.rows[i];
        table += "<tr>";
        for (var j = 0; j < this.columns.length; j++) {
          console.log(this.collect(row, column.field));
          const column = this.columns[j];
          table += "<td>";
          table += this.collect(row, column.field);
          table += "</td>";
        }
        table += "</tr>";
      }

      table += "</tbody></table>";

      return table;
    },

    dig: function(obj, selector) {
      var result = obj;
      const splitter = selector.split(".");

      for (let i = 0; i < splitter.length; i++) {
        if (result == undefined) return undefined;

        result = result[splitter[i]];
      }

      return result;
    },

    collect: function(obj, field) {
      if (typeof field === "function") return field(obj);
      else if (typeof field === "string") return this.dig(obj, field);
      else return undefined;
    },

    addCustomClass: function(obj, field) {
      let classAfter = this.collect(obj, field)
      return field+
      ' class-'+classAfter
    },


  },

  computed: {
    perPageOptions: function() {
      var options = (Array.isArray(this.perPage) && this.perPage) || [
        10,
        20,
        30,
        40,
        50
      ];

      // Force numbers
      options = options.map(v => parseInt(v));

      // Set current page to first value
      this.currentPerPage = options[0];

      // Sort options
      options.sort((a, b) => a - b);

      // And add "All"
      options.push(-1);

      // If defaultPerPage is provided and it's a valid option, set as current per page
      if (options.indexOf(this.defaultPerPage) > -1) {
        this.currentPerPage = parseInt(this.defaultPerPage);
      }

      return options;
    },
    processedRows: function() {
      var computedRows = this.rows;

      if (this.sortable !== false)
        computedRows = computedRows.sort((x, y) => {
          if (!this.columns[this.sortColumn]) return 0;

          const cook = x => {
            x = this.collect(x, this.columns[this.sortColumn].field);
            if (typeof x === "string") {
              x = x.toLowerCase();
              if (this.columns[this.sortColumn].numeric)
                x = x.indexOf(".") >= 0 ? parseFloat(x) : parseInt(x);
            }
            return x;
          };

          x = cook(x);
          y = cook(y);

          return (
            (x < y ? -1 : x > y ? 1 : 0) * (this.sortType === "desc" ? -1 : 1)
          );
        });

      if (this.searching && this.searchInput) {
        const searchConfig = { keys: this.columns.map(c => c.field) };

        // Enable searching of numbers (non-string)
        // Temporary fix of https://github.com/krisk/Fuse/issues/144
        searchConfig.getFn = function(obj, path) {
          if (Number.isInteger(obj[path])) return JSON.stringify(obj[path]);
          return obj[path];
        };

        if (this.exactSearch) {
          //return only exact matches
          (searchConfig.threshold = 0), (searchConfig.distance = 0);
        }

        computedRows = new Fuse(computedRows, searchConfig).search(
          this.searchInput
        );
      }

      return computedRows;
    },

    paginated: function() {
      var paginatedRows = this.processedRows;
      if (this.paginate)
        paginatedRows = paginatedRows.slice(
          (this.currentPage - 1) * this.currentPerPage,
          this.currentPerPage === -1
            ? paginatedRows.length + 1
            : this.currentPage * this.currentPerPage
        );
      return paginatedRows;
    },

    computedClass: function() {
      return (this.class = "responsive-table-not");
    },
  },

  mounted: function() {
    this.currentPerPage = this.currentPerPage;
  }
};
</script>
<style scoped src="materialize-css/dist/css/materialize.min.css"></style>

<style scoped>
  div.material-table {
    padding: 0;
  }

  tr.clickable {
    cursor: pointer;
  }

  #search-input {
    margin: 0;
    border: transparent 0 !important;
    height: 48px;
    color: rgba(0, 0, 0, 0.84);
  }

  #search-input-container {
    padding: 0 14px 0 24px;
    border-bottom: solid 1px #dddddd;
  }

  table {
    table-layout: fixed;
  }

  .table-header {
    height: 64px;
    padding-left: 24px;
    padding-right: 14px;
    -webkit-align-items: center;
    -ms-flex-align: center;
    align-items: center;
    display: flex;
    -webkit-display: flex;
    border-bottom: solid 1px #dddddd;
  }

  .table-header .actions {
    display: -webkit-flex;
    margin-left: auto;
  }

  .table-header .btn-flat {
    min-width: 36px;
    padding: 0 8px;
  }

  .table-header input {
    margin: 0;
    height: auto;
  }

  .table-header i {
    color: rgba(0, 0, 0, 0.54);
    font-size: 24px;
  }

  .table-footer {
    height: 56px;
    padding-left: 24px;
    padding-right: 14px;
    display: -webkit-flex;
    display: flex;
    -webkit-flex-direction: row;
    flex-direction: row;
    -webkit-justify-content: flex-end;
    justify-content: flex-end;
    -webkit-align-items: center;
    align-items: center;
    font-size: 12px !important;
    color: rgba(0, 0, 0, 0.54);
  }

  .table-footer .datatable-length {
    display: -webkit-flex;
    display: flex;
  }

  .table-footer .datatable-length select {
    outline: none;
  }

  .table-footer label {
    font-size: 12px;
    color: rgba(0, 0, 0, 0.54);
    display: -webkit-flex;
    display: flex;
    -webkit-flex-direction: row;
    /* works with row or column */

    flex-direction: row;
    -webkit-align-items: center;
    align-items: center;
    -webkit-justify-content: center;
    justify-content: center;
  }

  .table-footer .select-wrapper {
    display: -webkit-flex;
    display: flex;
    -webkit-flex-direction: row;
    /* works with row or column */

    flex-direction: row;
    -webkit-align-items: center;
    align-items: center;
    -webkit-justify-content: center;
    justify-content: center;
  }

  .table-footer .datatable-info,
  .table-footer .datatable-length {
    margin-right: 32px;
  }

  .table-footer .material-pagination {
    display: flex;
    -webkit-display: flex;
    margin: 0;
  }

  .table-footer .material-pagination li a {
    color: rgba(0, 0, 0, 0.54);
    padding: 0 8px;
    font-size: 24px;
  }

  .table-footer .select-wrapper input.select-dropdown {
    margin: 0;
    border-bottom: none;
    height: auto;
    line-height: normal;
    font-size: 12px;
    width: 40px;
    text-align: right;
  }

  .table-footer select {
    background-color: transparent;
    width: auto;
    padding: 0;
    border: 0;
    border-radius: 0;
    height: auto;
    margin-left: 20px;
  }

  .table-title {
    font-size: 20px;
    color: #000;
  }

  table tr td {
    padding: 0 0 0 56px;
    height: 48px;
    font-size: 13px;
    color: rgba(0, 0, 0, 0.87);
    border-bottom: solid 1px #dddddd;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  table td,
  table th {
    border-radius: 0;
  }

  table tr td a {
    color: inherit;
  }

  table tr td a i {
    font-size: 18px;
    color: rgba(0, 0, 0, 0.54);
  }

  table tr {
    font-size: 12px;
  }

  table th {
    font-size: 12px;
    font-weight: 500;
    color: #757575;
    cursor: pointer;
    white-space: nowrap;
    padding: 0;
    height: 56px;
    padding-left: 56px;
    vertical-align: middle;
    outline: none !important;
  }
  table th div {
    overflow: hidden;
    text-overflow: ellipsis;
  }

  table th.sorting-asc,
  table th.sorting-desc {
    color: rgba(0, 0, 0, 0.87);
  }

  table tbody tr:hover {
    background-color: #eee;
  }

  table th:last-child,
  table td:last-child {
    padding-right: 14px;
  }

  table th:first-child,
  table td:first-child {
    padding-left: 24px;
  }
</style>
<!-- <table-header-icons> -->
<style scoped>
  table th i {
    vertical-align: middle;
  }
  </style>
  <!-- <table-tooltips> -->
  <!-- <table-arrows> -->
  <style scoped>
  table th.sorting:before,
  table th.sorting-asc:before {
    font-family: "Material Icons";
    font-weight: normal;
    font-style: normal;
    font-size: 16px;
    line-height: 1;
    letter-spacing: normal;
    text-transform: none;
    display: inline-block;
    word-wrap: normal;
    -webkit-font-feature-settings: "liga";
    -webkit-font-smoothing: antialiased;
    content: "arrow_back";
    -webkit-transform: rotate(90deg);
    float: right;
    opacity: 0;
    vertical-align: middle;
  }

  table th.sorting:hover:before,
  table th.sorting-asc:hover:before,
  table th.sorting-desc:hover:before {
    opacity: 1;
  }

  table th.sorting-desc:before {
    content: "arrow_forward";
  }

  @media only screen and (min-width: 992px) {
    table th[data-tooltip] {
      position: relative;
    }

    table th[data-tooltip]:after {
      opacity: 0;
      visibility: hidden;
      position: absolute;
      content: attr(data-tooltip);
      padding: 6px 10px;
      bottom: 3.4em;
      left: 50%;
      transform: translateX(-50%) translateY(-2px);
      background: grey;
      color: white;
      white-space: nowrap;
      z-index: 2;
      border-radius: 2px;
      transition: opacity 0.2s cubic-bezier(0.64, 0.09, 0.08, 1),
        transform 0.2s cubic-bezier(0.64, 0.09, 0.08, 1);
    }

    table th[data-tooltip]:hover:after {
      display: block;
      opacity: 1;
      visibility: visible;
      transform: translateX(-50%) translateY(0);
    }
  }
</style>
<!-- <responsive-table> -->
<style scoped>
  @media only screen and (max-width: 992px) {
    table.responsive-table {
      border-bottom: solid 1px #dddddd;
    }
    table.responsive-table:after {
      content: "";
      display: block;
      clear: both;
    }
    table.responsive-table thead tr {
      padding: 0;
    }

    table.responsive-table tr th,
    table.responsive-table tr td {
      min-height: 40px;
      height: auto;
    }
    table.responsive-table tr th {
      padding: 10px;
      width: 100% !important;
    }
    table.responsive-table tr td {
      padding: 10px 0;
    }

    table.responsive-table th.sorting:after,
    table.responsive-table th.sorting-asc:after,
    table.responsive-table th.sorting-desc:after {
      display: inline-block;
      opacity: 0;
    }
    table.responsive-table th.sorting:hover:after,
    table.responsive-table th.sorting-asc:hover:after,
    table.responsive-table th.sorting-desc:hover:after {
      opacity: 1;
    }
    table.responsive-table tr,
    table.responsive-table tr td {
      border: 0;
    }
    table.responsive-table tbody tr {
      border-right: 1px solid #d0d0d0;
    }
  }
</style>
