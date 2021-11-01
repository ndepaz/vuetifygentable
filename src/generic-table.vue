<template>
  <div
    :url="apiUrl"
    :serverSideMode="serverSideMode"
    :itemsPerPage="itemsPerPage"
    :maxRecords="maxRecords"
    :httpMethod="httpMethod"
  >
    <v-data-table
      :headers="computedHeaders"
      :items="rows"
      :items-per-page="ItemsPerPageComputed"
      class="elevation-1"
      :search="search"
      :custom-filter="customFilter"
      :loading="isLoading"
      loading-text="Loading... Please wait"
      :item-key="itemKey"            
      :expanded.sync="expanded"
      :single-expand="singleExpand"
    >
      <!--top item-key="id" -->
      <template v-slot:top>
        <div>
          <v-toolbar flat>
            <v-toolbar-title v-if="title">{{ title }}</v-toolbar-title>
            <v-divider class="mx-4" inset vertical> </v-divider>
            <!--search bar-->
                <v-col cols="4">
              <v-card-title>
                <v-text-field
                  v-model="search"
                  append-icon="mdi-magnify"
                  label="Search"
                  single-line
                  hide-details
                  @keyup.enter="searchItems"
                ></v-text-field>
              </v-card-title>
            </v-col>
            <!--/search bar-->
            <v-spacer></v-spacer>
            <slot name="button" />
          </v-toolbar>
        </div>
      </template>
      <!--/top-->
      <!-- Body -->
      <template v-slot:item="{ item, expand, isExpanded }">
          <tr
            @click="rowClicked(item)"
            @contextmenu.prevent="selectItem(item, $event)"
            :class="selectedRowClass(item,selectedItem)+' grab'"
          >
            <td v-if="expandable">
              <v-btn
              fab
              icon
              x-small
              elevation="2"
               @click="expand(!isExpanded)"><v-icon>mdi-chevron-double-down</v-icon></v-btn>
            </td>
            <td v-for="valuePair in inspectItem(item)" :class="valuePair.prop">
              {{ valuePair.value }}
            </td>
          </tr>
      </template>
      <!-- /Body -->
      <template v-slot:expanded-item="{ headers, item }">
        <slot v-bind="{ headers, item }" name="expandable"></slot>
      </template>
    </v-data-table>

    <vue-context ref="menu">
      <template slot-scope="child">
        <slot name="contextMenuItems" :rowItem="child" />
      </template>
    </vue-context>
  </div>
</template>

<script>
import Vuetify, { VDataTable } from "vuetify/lib";
import "vue-context/src/sass/vue-context.scss";
import VueContext from "vue-context";

export default {
  components: {
    VDataTable,
    VueContext,
  },
  props: {
    selectedRowItems:{
      type:Array,
    },
    onLeftClick:{
      type:Function,
    },
    itemKey:{
      default:"id"
    },
    title:{
      type:String,
    },
    showContextMenu:{
    },
    maxRecords:{
      type:Number,
      default: 10
    },
    itemsPerPage:{
      type:Number,
    },
    serverSideMode:{
      type:Boolean,
      default: false
    },
    minCharSearch:{
      type:Number,
      default: 1
    },
    urlQuery:{
      type:String,
    },
    makeUrlQuery:{
      type:Function,
    },
    value:{
    },
    httpMethod:{
      type: String,
      required: false,
      default: "get"
    },
    expandable:{
      type:Boolean,
      default: false,
    }
  },
  data() {
    return {
      searchValue: "",
      headers: [],
      itemRows: [],
      selectedItem: null,
      apiUrl: null,
      isLoading: false,
      SearchResultsCount:  null,
      query:null,
      termsList:[],
      setHeaders: true,
      expanded: [],
      singleExpand: true,
      selectedRows: [],
    };
  },
  methods: {
    toggleSelection(keyID) {
      if (this.selectedRows.includes(keyID)) {
        this.selectedRows = this.selectedRows.filter(
          selectedKeyID => selectedKeyID !== keyID
        );
      } else {
        this.selectedRows.push(keyID);
      }
      this.$emit('update:selectedRowItems',this.selectedRows);
    },
    rowClicked(item){
      if(this.onLeftClick){
        if(this.selectedRowItems){
          this.toggleSelection(item[this.itemKey]);
        } else {
          this.selectedItem = item;
          this.onLeftClick(this.selectedItem);
        }
      }
    },
    selectedRowClass(item,selectedItem){
      if(this.selectedRowItems){
        return this.selectedRows.indexOf(item[this.itemKey])>-1?'cyan':'';
      }
      let className = item === selectedItem?'selectedRow':'';
      return className;
    },
    clearInputSearch(){
      this.search = "";
    },
    clearTermsSearch(){
      this.terms = [];
    },
    selectItem(item, event) {
      this.selectedItem = item;
      this.handleClick(item, event);
    },
    handleClick: function (item, event) {
      if (this.showContextMenu) {
        this.$refs.menu.open(event, item);
      }
    },
    refresh(maxRecords) {
      this.setDefaultQuery();
      this.fetchItemsByTerm();
    },
    async searchItems(searchTerm) {
      this.search=  typeof searchTerm == "string" ? searchTerm :this.search;
      this.setDefaultQuery();
      return await this.fetchItemsByTerm();
    },
    setDefaultQuery(){
      let query ="/?search=" + this.search;
      query += "&maxRecords=" + this.maxRecords;
      this.query = query;
    },
    fetchItemsByTerm() {
      this.isLoading = true;
      let url = this.$attrs.url;
      if(this.httpMethod == 'get'){
        url+=this.query;
      }

      let axiosOptions = {
        method:this.httpMethod,
        url
      };
      if(this.httpMethod == 'post'){
        axiosOptions.data = this.terms.length > 0 ?this.terms: this.defaultSearchTerms;
      }
      
      return new Promise((resolve,reject)=>{
        this.$axios(axiosOptions)
        .then((r) => {
          this.rows = r.data.rows;
          this.headers = r.data.headers;
          if(this.expandable){
            this.headers.unshift({ text: 'EXPAND', align: 'left', sortable: false, value: 'expand' })
          }
          this.$emit('OnDataGet',r);
          resolve(r)
          return true;
        })
        .catch((e) => {
          this.$emit('OnDataGetFail',e);
          reject(e);
          return false;
        })
        .finally((e) => {
          this.isLoading = false;
        });
      });
    },
    inspectItem(item) {
      let newObject = {};
      let i = this.expandable? 1: 0; 
      for(i = i;i<this.headers.length;i++) {
        let element = this.headers[i];

        var key = element.value;
        var objectKey = Object.keys(item).find(itemKey => itemKey.toLowerCase() === key.toLowerCase());
        
        if (element.shouldDeleteColumn) {
          delete item[key];
          newObject[i] = { prop: " d-none", value: item[objectKey] };
        }
        newObject[i] = { prop: element.align, value: item[objectKey] };
      }
      return newObject;
    },
    customFilter(value, search, item) {
      let found = 0;
      let typeName = typeof(value);
      
      if(Array.isArray(value) && this.arrayFilter(value, search, item)){
        found++;
      }
      if(typeName == "number" && this.numberFilter(value, search, item)){
        found++;
      }
      if(typeName == "string" && this.stringFilter(value, search, item)){
        found++;
      }
      
      return found > 0;
    },
    getByValue4(arr, value) {
      var o;

      for (var i=0, iLen=arr.length; i<iLen; i++) {
        o = arr[i];

        for (var p in o) {
          if (o.hasOwnProperty(p)) {
            let propValue = o[p];
            if(typeof(propValue) == "string" && propValue.toString().toLowerCase().indexOf(value.toLowerCase()) !== -1){
              return propValue;
            } else if (propValue == value){
              return propValue;
            }
          }
        }
      }
    },
    arrayFilter(value, search) {
         return (
          value != null &&
          search != null &&
          this.getByValue4(value,search) != undefined
        );
    },
    numberFilter(value, search) {
         return (
          value != null &&
          search != null &&
          value ==search
        );
    },
    stringFilter(value, search) {
      return(
          value != null &&
          search != null &&
          value.toString().toLowerCase().indexOf(search.toLowerCase()) !== -1
        );
    },
  },
  
  computed: {
    terms: {
        get: function() {
          
          return this.termsList;
        },
        set: function(newValue) {
          this.termsList = newValue;
          
          this.$emit("input",this.termsList );
        }
    },
    defaultSearchTerms:{
      get: function() {
          return [
              {FieldName:"*",ComparisonOperator: "contains", SearchValue:this.search,ClauseOperator:"And",},
          ];
      },
    },
    search:{
      get: function() {
          return this.searchValue;
        },
        set: function(newValue) {
          this.searchValue = newValue;
        }
    },
    rows:{
      get: function() {
          return this.itemRows;
        },
        set: function(newValue) {
          this.itemRows = newValue;
        }
    },
    ItemsPerPageComputed: function () {
      return this.itemsPerPage != undefined || this.itemsPerPage != null
        ? this.itemsPerPage
        : this.maxRecords;
    },
    computedHeaders() {
      return this.headers.filter((t) => !t.shouldDeleteColumn);
    },
    
  },
  watch: {
    headers(newValue, OldValue) {},
    isLoading(newValue, OldValue) {}, 
    searchValue(newValue, OldValue) {}, 
    value(newValue){
      this.terms =newValue;
      
    }
  },
  created() {
    this.terms = this.value;
    
    this.refresh(this.maxRecords);
  },
}
</script>

<style>
.selectedRow {
  background-color: lightgray;
  font-weight: bold;
}
.d-none {
  display: none !important;
}
.grab {
  cursor: pointer;
}
</style>