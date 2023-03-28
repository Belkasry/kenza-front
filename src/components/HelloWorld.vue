<template>
  <v-container>
    <v-card class="m-5 mb-6 p-4">
      <v-form @submit.prevent="submitCsv" class="d-flex">
        <v-file-input
          v-model="csvFile"
          :label="'Select a CSV file'"
          :rules="[csvFile => !!csvFile || 'CSV file is required']"
          accept=".csv"
        ></v-file-input>

        <v-btn color="primary" class="float-right mt-3 ml-4" type="submit">Upload</v-btn>
      </v-form>
    </v-card>

    <v-card>
      <v-card-title class="d-flex">
        <v-text-field
          v-model="search"
          append-icon="mdi-magnify"
          label="Search"
          single-line
          class="p-2 mr-4"
          hide-details
        ></v-text-field>
        <v-select
          item-text="filename" item-value="mappings"
          v-model="selectedMapping" :items="availableMappings" class="mt-5 mr-4" label="Select a mapping" clearable
          color="secondary"></v-select>
        <v-btn color="info"
               class="p-2 ml-2" @click="dialogVisible=true">
          <v-icon>application-cog-outline</v-icon>
          Config Export
        </v-btn>
        <v-btn color="success" class="ml-2 p-2" @click="exportToExcel">Export to Excel</v-btn>
      </v-card-title>
      <v-dialog v-model="dialogVisible"
                fullscreen
                :scrim="false"
                transition="dialog-bottom-transition">
        <v-toolbar
          dark
          color="primary"
        >
          <v-btn
            icon
            dark
            @click="dialogVisible = false"
          >
            <v-icon>mdi-close</v-icon>
          </v-btn>
          <v-toolbar-title>Settings</v-toolbar-title>
          <v-spacer></v-spacer>
          <v-toolbar-items>
            <v-btn @click="saveMapping">Save Mapping</v-btn>
            <v-btn color="secondary" @click="dialogVisible = false">Cancel</v-btn>
          </v-toolbar-items>
        </v-toolbar>
        <v-card>
          <v-card-title color="primary">
            Mapping
          </v-card-title>
          <v-card-text>
            <v-row>
              <v-col v-for="(key, index) in templateKeys" :key="index" :cols="4">
                <label>{{ key.label }}</label>
                <v-autocomplete v-model="selectedHeaders[key.key]" :items="getHeadersForTemplateKey(key.key)"
                                label="Select a field to map" clearable></v-autocomplete>
              </v-col>
            </v-row>
          </v-card-text>

        </v-card>
      </v-dialog>
      <v-dialog v-model="dialogSaveMap">
        <v-card>
          <v-card-title>Save mappings</v-card-title>
          <v-card-text>
            <v-text-field v-model="mappingName" label="Mapping name"></v-text-field>
          </v-card-text>
          <v-card-actions>
            <v-btn @click="dialogSaveMap = false">Cancel</v-btn>
            <v-btn @click="saveMappingConfig">
              <v-icon>mdi-save</v-icon>
              Save
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-data-table :search="search" v-if="headers" color="primary" :headers="headers" :items="items"></v-data-table>
    </v-card>
  </v-container>
</template>

<script>
import axios from 'axios';

export default {
  data() {
    return {
      availableMappings: [],
      selectedMapping: null,
      dialogSaveMap: false,
      mappingName: "",
      mappings: [],
      dialogVisible: false,
      csvFile: null,
      search: "",
      file: null,
      headers: [],
      items: [],
      templateKeys: [],
      selectedHeaders: {},
    };
  },
  watch: {
    selectedMapping(val) {
      const selectedHeaders = {};
      for (const key of val) {
        selectedHeaders[key.templateKey] = key.headerKey;
      }
      this.selectedHeaders = selectedHeaders;
    }

  },
  mounted() {
    this.fetchTable();
    this.fetchTemplateKeys();
    this.loadAvailableMappings();
  }
  ,
  methods: {
    exportToExcel() {
      axios.post('https://kenza-amazon.herokuapp.com/public/index.php/api/export', {
        data: this.items,
        mapping: this.selectedMapping,
        template: this.templateKeys,
      }, {
        responseType: 'blob'
      }).then(response => {
        const url = window.URL.createObjectURL(new Blob([response.data]));
        const link = document.createElement('a');
        link.href = url;
        link.setAttribute('download', 'export.xlsx');
        document.body.appendChild(link);
        link.click();
      }).catch(error => {
        console.error(error);
      });
    },
    loadAvailableMappings() {
      axios
        .get("https://kenza-amazon.herokuapp.com/public/index.php/api/mappings")
        .then((response) => {
          this.availableMappings = response.data;
        })
        .catch((error) => {
          console.error(error);
        });
    }
    ,

    getHeadersForTemplateKey(templateKey) {
      return this.headers.filter((header) => !Object.values(this.selectedHeaders).includes(header.key) || header.key === this.selectedHeaders[templateKey]);
    }
    ,
    saveMapping() {
      const mappings = Object.entries(this.selectedHeaders).map(([templateKey, headerKey]) => ({
        templateKey,
        headerKey,
      }));
      this.mappings = mappings; // do something with the mappings
      this.dialogSaveMap = true;
    }
    ,
    async saveMappingConfig() {
      console.log(this.mappings)
      axios
        .post("https://kenza-amazon.herokuapp.com/public/index.php/api/config", {
          name: this.mappingName,
          mappings: this.mappings,
        })
        .then((response) => {
          console.log(response.data);
          this.dialogSaveMap = false;
          this.mappingName = "";
          this.selectedHeaders = {};
        })
        .catch((error) => {
          console.error(error);
        });
    }
    ,
    async fetchTemplateKeys() {
      const response = await axios.get('https://kenza-amazon.herokuapp.com/public/index.php/api/template')
      this.templateKeys = response.data;
    }
    ,
    async fetchTable() {

      let config = {
        method: 'get',
        url: 'https://kenza-amazon.herokuapp.com/public/index.php/api/csv-to-json',
      };

      axios.request(config)
        .then((response) => {
          this.headers = response.data.headers
          this.items = response.data.items
        })
        .catch((error) => {
          console.log(error);
        });

    }
    ,
    async submitCsv() {
      const formData = new FormData()
      formData.append('csv', this.csvFile)

      try {
        const response = await axios.post('https://kenza-amazon.herokuapp.com/public/index.php/api/csv', formData, {
          headers: {
            'Content-Type': 'multipart/form-data'
          }
        })

        this.fetchTable();
      } catch (error) {
        console.error(error)
      }
    }
    ,
    exportData() {
      const exportedItems = this.items.map(item => {
        const exportedItem = {};
        for (const key in this.selectedKeys) {
          const tableKey = this.selectedKeys[key].tableKey;
          const fixedKey = this.selectedKeys[key].fixedKey;
          exportedItem[fixedKey] = item[tableKey];
        }
        return exportedItem;
      });
      // download the exported data as a JSON file

    }
  }

}
;
</script>
<style>
.v-data-table thead {
  font-size: 1.3rem;
  background-color: #e9dd1785;
  color: white;
}
</style>
