# vuetifygentable

Install through npm
```
npm install vuetifygentable
```

Import into your vue project
``` 
import gentable from 'vuetifygentable';
export default {
  components:{
    gentable
  }
}
```

Example, on the vue markup side. 
```
<template>
  <v-row justify="center" align="center">
    <v-col cols="12" sm="8" md="6"> </v-col>
    <gentable
      :url="endPointUrl"
      title="Menus"
      :showContextMenu="true"
      ref="table"
      :serverSideMode="true"
      :makeUrlQuery="makeUrlQuery"
      :minCharSearch="1"
    >
      <v-btn color="blue" dark class="mr-4" slot="button" @click="reload">
        Refresh
      </v-btn>
      

      <template slot="contextMenuItems" slot-scope="rowItem">
        <li>
          <a @click.prevent="editRecord(rowItem)">View</a>
        </li>
      </template>
    </gentable>
  </v-row>
</template>
<script>
import gentable from "vuetifygentable";
export default {
  components: {
    gentable,
  },
  data() {
    return {
      row: {},
      selected: null,
      endPointUrl: "/addresses",
    };
  },
  methods: {
    reload() {
      //refresh
      this.$refs.table.refresh();
    },
    makeUrlQuery(searchTerm) {
      //formulate and advanced query
      return `/?search=${searchTerm}`;
    },
    editRecord(row) {
      //access the row item
      let item = row.rowItem.data;
      this.$axios
        .get("/addresses/" + item.id)
        .then((r) => {
          console.log(r.data);
        })
        .catch((e) => {
          console.log(e);
        });
    },
  },
};
</script>
```

This is compatible with VuetifyTableApi nuget package that was made for this purpose.
In a .net core 5 application, you can use it like this through the nuget package console or through nuget package manager gui.
```
PM> Install-Package VuetifyTableApi 
```
Assuming using .net core, on the controller side, you can use the VuetifyTableApi like so.
This part you can customize as you wish. More documentation it's on it's way for VuetifyTableApi.

```
[HttpGet]
public async Task<ActionResult<Table>> GetAddresses([FromQuery] DataRequest dataRequest)
{
    var tb = new Table();
    tb.skipHeaderVirtualProps = true;
    int maxRecords = dataRequest.maxRecords;
    string search = dataRequest.search;
    var query = _context.Addresses.AsNoTracking();
    if (!string.IsNullOrWhiteSpace(search))
    {
        query = query.Where(t =>
            t.StreetAddress.Contains(search)
            || t.City.Contains(search)
            || t.State.Contains(search)

            );
    }
    query = maxRecords == 0 ? query : query.Take(maxRecords);

        var result = await query.Select(
                    t => new
                    {
                        id = t.Id,
                        t.StreetAddress,
                        t.City,
                        t.State,
                        t.ZipCode,
                        t.CreatedAt,
                        t.UpdatedAt,
                        t.DeletedAt,
                    }
                   ).ToListAsync<object>();

        tb.rows = result;
        tb.SetHeaders<Address>();
    return tb;
}
```
