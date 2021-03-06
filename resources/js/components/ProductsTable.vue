<template>
  <div>
    <modal-trash-box :is-active="isModalActive" :trash-subject="trashObjectName" @confirm="trashConfirm" @cancel="trashCancel"/>
    <b-field grouped group-multiline>
          <b-button type="is-info" disabled>Anzahl Einträge: {{this.total}}</b-button>
          <a :href="this.filterGeneralUrl"><b-button class="btn excel-export">Download Excel</b-button></a>
          <a :href="this.filterEnhancedUrl"><b-button class="btn excel-export">Enhanced Excel</b-button></a>
    </b-field>

    <b-table
      :checked-rows.sync="checkedRows"
      :checkable="checkable"
      :loading="isLoading"
      paginated
      backend-pagination
      :total="total"
      :per-page="perPage"
      :striped="true"
      :hoverable="true"
      @page-change="onPageChange"

      backend-sorting
      :default-sort-direction="defaultSortOrder"
      :default-sort="[sortField, sortOrder]"
       @sort="onSort"

      backend-filtering
      @filters-change="onFilterChange"

      :data="products">

      <template slot-scope="props">

        <b-table-column class="has-no-head-mobile is-image-cell">
          <div v-if="props.row.avatar" class="image">
            <img :src="props.row.avatar" class="is-rounded">
          </div>
        </b-table-column>
        <b-table-column label="Artikel-Nr." field="st_article_nr" sortable searchable>
          {{ props.row.st_article_nr }}
        </b-table-column>
        <b-table-column label="Serial-Nr." field="st_serial_nr" sortable searchable>
          {{ props.row.st_serial_nr }}
        </b-table-column>
        <b-table-column label="Status" field="lifecycle" sortable>
          {{ props.row.lifecycle }}
        </b-table-column>
        <b-table-column label="Produktionsdaten" field="production_data_count">
          {{ props.row.production_data_count }}
        </b-table-column>
        <b-table-column label="Komponenten" field="components_count">
          {{ props.row.components_count }}
        </b-table-column>
        <b-table-column label="Produktionsauftrag" field="production_order_nr" sortable searchable>
          {{ props.row.production_order_nr }}
        </b-table-column>
        <b-table-column label="Erstellt" field="created_at" sortable>
          <small class="has-text-grey is-abbr-like" :title="props.row.created_at">{{ props.row.created_at | moment("DD.MM.YYYY / h:mm:ss")}}</small>
        </b-table-column>
        <b-table-column custom-key="actions" class="is-actions-cell">
          <div class="buttons is-right">
            <button class="button is-small is-danger" type="button" @click.prevent="trashModal(props.row)">
              <b-icon icon="trash-can" size="is-small"/>
            </button>
          </div>
        </b-table-column>
        <b-table-column custom-key="actions" class="is-actions-cell">
          <div class="buttons is-right">
            <button class="button is-small is-success" type="button" @click.prevent="showEditPanel(props.row)">
              <b-icon icon="google-photos" size="is-small"/>
            </button>
          </div>
        </b-table-column>
      </template>

      <section class="section" slot="empty">
        <div class="content has-text-grey has-text-centered">
          <template v-if="isLoading">
            <p>
              <b-icon icon="dots-horizontal" size="is-large"/>
            </p>
            <p>Fetching data...</p>
          </template>
          <template v-else>
            <p>
              <b-icon icon="emoticon-sad" size="is-large"/>
            </p>
            <p>Nothing's here&hellip;</p>
          </template>
        </div>
      </section>

      <template #footer>
        <div class="has-text-right">
          <b-select v-model="perPage">
            <option value="10">10 per page</option>
            <option value="20">20 per page</option>
            <option value="50">50 per page</option>
            <option value="100">100 per page</option>
            <option value="1000">1000 per page</option>
          </b-select>

        </div>
      </template>

    </b-table>
  </div>
</template>

<script>
import ModalTrashBox from '@/components/ModalTrashBox'
import debounce from 'lodash/debounce'
import downloadexcel from "vue-json-excel";
import BButton from "buefy/src/components/button/Button";

export default {
  name: 'ProductsTable',
  components: {BButton, ModalTrashBox, downloadexcel },
  props: {
    dataUrl: {
      type: String,
      default: null
    },
    checkable: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      isModalActive: false,
      trashObject: null,
      products: [],
      isLoading: false,
      isExcelLoading: false,
      perPage: 10,
      checkedRows: [],
      sortField:'',
      sortOrder:'asc',
      defaultSortOrder:'asc',
      page: 1,
      total: 0,
      filterValues: '{}',
      filterGeneralUrl: '',
      filterEnhancedUrl: '',
      excelProducts: [],
      jsonFields: {
        'Artikel-Nr.': 'st_article_nr',
        'Serial-Nr.': 'st_serial_nr',
        'Status': 'lifecycle',
        'Produktionsdaten': 'production_data_count',
        'Komponenten': 'components_count',
        'Produktionsauftrag': 'production_order_nr',
        'Erstellt': 'created_at'
      },
    }
  },
  watch:{
    perPage:function(){
      this.getData();
    }
  },
  computed: {
    trashObjectName () {
      if (this.trashObject) {
        return this.trashObject.st_article_nr+"/"+this.trashObject.st_serial_nr
      }
      return null
    }
  },
  created () {
    this.getData()
    this.getFilteringURL ()
  },
  methods: {
    onPageChange(page) {
        this.page = page
        this.getData ()
    },
    onSort(field, order) {
        this.sortField = field
        this.sortOrder = order
        this.getData()
    },
    onFilterChange: debounce(function (filter) {
      console.warn('filter', Object.entries(filter));
      this.filterValues = '';
      this.filterValues = encodeURIComponent(JSON.stringify(filter));
      this.getData()
      this.getFilteringURL()
    }, 250),
    getData () {
      if (this.dataUrl) {
        this.isLoading = true
        const params = [
                `size=${this.perPage}`,
                `sort_by=${this.sortField}.${this.sortOrder}`,
                `page=${this.page}`,
                `filter=${this.filterValues}`
            ].join('&')

        axios.create({
           headers: {
            'Content-Type': 'application/json',
        }
        })
          .get(this.dataUrl+'?'+params)
          .then(r => {
            this.isLoading = false
            if (r.data && r.data.data) {
              this.perPage = r.data.meta.per_page
              this.total = r.data.meta.total
              this.page = r.data.meta.current_page
              this.products = r.data.data
            }
          })
          .catch( err => {
            this.isLoading = false
            this.$buefy.toast.open({
              message: `Error: ${err.message}`,
              type: 'is-danger',
              queue: false
            })
          })
      }
    },
    getFilteringURL () {
        if(this.dataUrl){
            const paramsGeneral = [
                `enhanced=0`,
                `sort_by=${this.sortField}.${this.sortOrder}`,
                `page=${this.page}`,
                `filter=${this.filterValues}`
            ].join('&')

            const paramsEnhance = [
                `enhanced=1`,
                `sort_by=${this.sortField}.${this.sortOrder}`,
                `page=${this.page}`,
                `filter=${this.filterValues}`
            ].join('&')

            this.filterGeneralUrl = this.dataUrl + '/excel?' + paramsGeneral
            this.filterEnhancedUrl = this.dataUrl + '/enhancedExcel?' + paramsEnhance
        }
    },
    trashModal (trashObject) {
      this.trashObject = trashObject
      this.isModalActive = true
    },
    showEditPanel (editableObject) {
      this.$emit("onSettingShow", editableObject)
    },
    trashConfirm () {
      this.isModalActive = false
      axios
        .delete(`${this.dataUrl}/product/${this.trashObject.id}`)
        .then( r => {
          this.getData()  // update list
          this.$buefy.snackbar.open({
            message: `Deleted ${r.data.data.st_article_nr}/${r.data.data.st_serial_nr} and ${r.data.data.deleted_device_records} Measurements, ${r.data.data.deleted_sub_components} Components.`,
            queue: false
          })
        })
        .catch( err => {
          this.$buefy.toast.open({
            message: `Error: ${err.message}`,
            type: 'is-danger',
            queue: false
          })
        })
    },
    trashCancel () {
      this.isModalActive = false
    },
    startDownload () {
      console.log("Started Download!")
    },
    finishDownload () {
      console.log("Finished Download!")
    }
  }
}
</script>
