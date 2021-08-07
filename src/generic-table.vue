<template>
    <div :url=apiUrl :serverSideMode="serverSideMode" :startLoaded="startLoaded" :itemsPerPage="itemsPerPage" :maxRecords="maxRecords">

        <v-data-table :headers="computedHeaders"
                      :items="rows"
                      :items-per-page="ItemsPerPageComputed"
                      class="elevation-1"
                      :search="search"
                      :custom-filter="customFilter"
                      :loading="isLoading"
                      loading-text="Loading... Please wait"
                      
                      >
            <!--top item-key="id" -->
            <template v-slot:top>
                <div>
                    <v-toolbar flat color="white">
                        <v-toolbar-title>{{title}}</v-toolbar-title>
                        <v-divider class="mx-4"
                                   inset
                                   vertical>
                        </v-divider>
                        <!--search bar-->
                        <v-col cols="6">
                            <v-card-title>
                                <v-text-field v-model="search"
                                              append-icon="mdi-magnify"
                                              label="Search"
                                              single-line
                                              hide-details ></v-text-field>
                            </v-card-title>
                        </v-col>
                        <!--/search bar-->
                        <v-spacer></v-spacer>
                        <slot name="button"/>
                    </v-toolbar>
                </div>
            </template>
            <!--/top-->
            <!-- Body -->
            <template v-slot:body="{ items }">
                <tbody>
                    <tr v-for="item in items" :key="item.id" @contextmenu.prevent="selectItem(item,$event)" :class="{'selectedRow': item === selectedItem, }">
                        <td v-for="valuePair in inspectItem(item)" :class="valuePair.prop" >{{ valuePair.value }}</td>
                    </tr>
                </tbody>
            </template>
            <!-- /Body -->
        </v-data-table>

        <vue-context ref="menu" >
            <template slot-scope="child">
                <slot name="contextMenuItems" :rowItem="child" />
            </template>
        </vue-context>
    </div>
</template>

<script>
import Vuetify, { VDataTable } from 'vuetify/lib';
    import 'vue-context/src/sass/vue-context.scss';
    import VueContext from 'vue-context';
    
    export default {
        components: {
            VDataTable
        },
        //inheritAttrs: false,
        props:[
            "title",
            "topRightButton",
            "showContextMenu",
            "maxRecords",
            "itemsPerPage",
            "serverSideMode",
            "startLoaded",
            "minCharSearch"
        ],
        data() {
            return {
                search: '',
                headers: [],
                rows: [],
                selectedItem: null,
                apiUrl: null,
                isLoading: false,
            }
        },
        methods: {
            refresh(maxRecords){
                this.fetchItemsByTerm({search:this.search,maxRecords,setHeaders:true});
            },
            handleClick: function (item, event) {
                if(this.showContextMenu ){
                    this.$refs.menu.open(event, item);
                }
            },
            selectItem(item, event) {
                this.selectedItem = item
                this.handleClick(item, event);
            },
            async fetchItemsByTerm(term){
                let {search,maxRecords,setHeaders} = term;
                this.isLoading = true;
                let query = "/?search="+search;
                if(maxRecords){
                    query+="&maxRecords="+maxRecords;
                }
                return await this.$axios.get(this.$attrs.url+query).then((r) => {
                    this.rows = r.data.rows
                    if(setHeaders){
                        this.headers = r.data.headers;
                    }
                    return true;
                }).catch((e) => {
                    console.error(e);
                    return false;
                }).finally(e=>{
                    this.isLoading = false;
                });
            },
            async searchItems(search){
                let minSearch = this.minCharSearch?this.minCharSearch:1;
                if(search && search.length >= minSearch){
                    return await this.fetchItemsByTerm({search,maxRecords:10});
                } else if(search.length == 0) {
                    return await this.fetchItemsByTerm({search,maxRecords:10});
                }
            },
            inspectItem(item){
                let newObject = {};
                let i = 0;
                this.headers.forEach(element => {
                    i++;
                    var key = element.value;
                    if(element.shouldDeleteColumn){
                         delete item[key];
                         newObject[i]={ prop: " d-none",value: item[key]}
                    }
                    newObject[i]={ prop: element.align,value: item[key]}
                });
                return newObject;
            },
            customFilter(value, search, item) {
                 if(!this.serverSideMode){
                    return value != null &&
                        search != null &&
                        typeof value === 'string' &&
                        value.toString().indexOf(search) !== -1;
                } else {
                    return true;//used to display all records
                }
            },
        },
        watch:{
            headers(newValue,OldValue){

            },
            rows(newValue,OldValue){

            },
            search(newValue,OldValue){
                if(this.serverSideMode){
                    this.searchItems(newValue);
                }
            },
            isLoading(newValue,OldValue){

            }
        },
        computed:{
            initialMaxRecords: function(){
                return this.maxRecords != undefined || this.maxRecords != null ? this.maxRecords:25;
            },
            ItemsPerPageComputed:function(){
                return this.itemsPerPage != undefined || this.itemsPerPage  != null ? this.itemsPerPage : this.initialMaxRecords;
            },
            computedHeaders () {
                return this.headers.filter(t=> !t.shouldDeleteColumn);  
            }
        },
        components: {
            VueContext
        },
        created() {
            if(this.maxRecords !== null || this.maxRecords !== undefined)
            this.refresh(this.maxRecords);
                else
            this.refresh();
        }
    }
</script>

<style >
    .selectedRow {
        background-color: lightgray;
        font-weight: bold;
    }
    .d-none{display:none!important}
</style>