<template>
  <div class="bg-indigo-accent-1 px-4">
    <v-card class="pt-3 px-2 mt-2" style="background: #561dc7;
    color: white; margin-top: 20px !important;">
      <v-form @submit.prevent="submitCsv" class="d-flex">
        <v-file-input
          v-model="csvFile"
          color="deep-purple-accent-4"
          label="CSV"
          placeholder="Select your files"
          prepend-icon="mdi-paperclip"
          variant="outlined"
          :show-size="1000"
          :rules="[csvFile => !!csvFile || 'CSV file is required']"
          accept=".csv"
        ></v-file-input>

        <v-btn color="primary" class="float-right mt-2 mx-3" type="submit">
          <v-icon>mdi-upload</v-icon>
          Upload
        </v-btn>
      </v-form>
      <div class="d-flex justify-end flex-sm-row flex-column">
        <v-btn color="yellow"
               class="p-2 ml-2 mt-2" @click="dialogFige=true">
          <v-icon>mdi-application-cog-outline</v-icon>
          Header Excel Template
        </v-btn>
        <v-btn color="info"
               class="p-2 ml-2 mt-2" @click="dialogVisible=true">
          <v-icon>mdi-set-center-right</v-icon>
          Mapping
        </v-btn>
        <v-btn color="success" class="ml-2 p-2 mt-2" @click="exportToExcel">
          <v-icon>mdi-file-excel</v-icon>
          Export to Excel
        </v-btn>
      </div>
      <v-dialog v-model="dialogFige"
                fullscreen
                :scrim="false"
                transition="dialog-bottom-transition">
        <v-toolbar
          dark
          color="primary"
        >
          <v-btn
            icon
            @click="dialogFige = false"
          >
            <v-icon>mdi-close</v-icon>
          </v-btn>
          <v-toolbar-title>Settings Valeurs figés</v-toolbar-title>
          <v-spacer></v-spacer>
          <v-toolbar-items>
            <v-btn @click="configValFige">Save</v-btn>
            <v-btn color="secondary" @click="dialogFige = false">Cancel</v-btn>
          </v-toolbar-items>
        </v-toolbar>
        <v-card style="background: #f0f7ff">
          <v-card-title>
            <span class="headline">Edit Template Keys</span>
          </v-card-title>
          <v-card-text>
            <v-row>
              <v-col>
                <v-text-field label="Key" v-model="computedKey" disabled></v-text-field>
              </v-col>
              <v-col>
                <v-text-field label="Label" v-model="newKey.label"></v-text-field>
              </v-col>
              <v-col>
                <v-switch label="Figé" v-model="newKey.fige"></v-switch>
              </v-col>
              <v-text-field
                label="Valeur Figée"
                v-if="newKey.fige"
                v-model="newKey.default"
              ></v-text-field>
              <v-spacer
                v-else
              ></v-spacer>
              <v-col>
                <v-btn color="primary" icon @click="addRow">
                  <v-icon>mdi-plus</v-icon>
                </v-btn>
              </v-col>
            </v-row>
            <v-row v-for="(key, index) in templateKeys" :key="index">
              <v-col>
                <v-text-field v-model="key.key" readonly></v-text-field>
              </v-col>
              <v-col>
                <v-text-field v-model="key.label"></v-text-field>
              </v-col>
              <v-col>
                <v-switch v-model="key.fige" color="success"></v-switch>
              </v-col>
              <v-text-field
                v-if="key.fige"
                v-model="key.default"
                label="Valeur Figée"
              ></v-text-field>
              <v-spacer
                v-else
              ></v-spacer>
              <v-col>
                <v-btn color="error" icon @click="deleteRow(index)">
                  <v-icon>mdi-delete</v-icon>
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>

      </v-dialog>
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
        <v-card class="pb-4">
          <v-card-title color="primary">
            Mapping
          </v-card-title>
          <v-card-text class="py-5">
            <v-row>
              <v-col v-for="(key, index) in templateKeys" :key="index" :cols="4">
                <label>{{ key.label }}</label>
                <v-autocomplete v-if="!key.fige" v-model="selectedHeaders[key.key]"
                                :items="getHeadersForTemplateKey(key.key)"
                                label="Select a field to map" clearable></v-autocomplete>
                <v-text-field label="Valeur figée" v-else :model-value="key.default" disabled></v-text-field>
              </v-col>
            </v-row>
          </v-card-text>
          <div class="mb-9"></div>
        </v-card>
      </v-dialog>
      <v-dialog v-model="dialogSaveMap">
        <v-card>
          <v-card-title>Save mappings</v-card-title>
          <v-card-text>
            <v-text-field v-model="mappingName" label="Mapping name"></v-text-field>
          </v-card-text>
          <v-card-actions>
            <v-btn @click="dialogSaveMap = false" color="light">Cancel</v-btn>
            <v-btn @click="saveMappingConfig" color="success">
              <v-icon>mdi-save</v-icon>
              Save
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <div class="d-flex flex-grow-1 justify-center my-4 flex-sm-row flex-column">
        <v-text-field
          variant="outlined"
          v-model="search"
          prepend-inner-icon="mdi-magnify"
          label="Search"
          hide-details
          class="mx-2"
          density="compact"
        ></v-text-field>
        <v-btn class="mr-2 mt-1" color="red" @click="items=[];headers=[];"><v-icon>mdi-delete</v-icon>Reset</v-btn>
        <div class="mr-2 mt-1">
        <v-btn icon @click="scroll(1)" class="mr-2 mt-1" size="x-small">
          <v-icon>mdi-page-first</v-icon>
        </v-btn>
        <v-btn icon @click="scroll(2)" class="mr-2 mt-1" size="x-small">
          <v-icon>mdi-chevron-left</v-icon>
        </v-btn>
        <v-btn icon @click="scroll(3)" class="mr-2 mt-1" size="x-small">
          <v-icon>mdi-chevron-right</v-icon>
        </v-btn>
        <v-btn icon @click="scroll(4)" class="mr-2 mt-1" size="x-small">
          <v-icon>mdi-page-last</v-icon>
        </v-btn>
        </div>
      </div>
    </v-card>
    <v-data-table :search="search"
                  v-if="headers" color="primary"
                  :headers="headers"
                  :items="items"
                  density="compact"></v-data-table>

  </div>
</template>

<script>
import axios from 'axios';
import {VDataTable} from "vuetify/labs/components";

export default {
  components: {
    VDataTable
  },
  computed: {
    "computedKey"() {
      return this.newKey.label
        .toLowerCase()
        .replace(/[\s]+/g, '_')
        .replace(/[^a-z0-9_]/g, '');
    }
  },
  data() {
    return {
      newKey: {
        key: '',
        label: '',
        fige: false,
        default: ''
      },
      dialogFige: false,
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
      api_path: "http://127.0.0.1:8001/api"
    };
  },

  mounted() {
    this.fetchTable();
    this.fetchTemplateKeys();
    this.loadAvailableMappings();
  }
  ,
  methods: {
    scroll(direction) {
      const table = this.$el.querySelector(".v-table__wrapper");
      let scrollGoal = 0;
      if (direction == 2)
        scrollGoal = (table.scrollWidth - table.clientWidth) / 3;
      else if (direction == 3)
        scrollGoal = (2 * (table.scrollWidth - table.clientWidth)) / 3;
      else if (direction == 4)
        scrollGoal = table.scrollWidth - table.clientWidth;
      table.scroll({
        left: scrollGoal,
        behavior: "smooth",
      });
      // table.classList.add("d-none");
      // const tableWidth = table.offsetWidth;
      // const tableScrollWidth = table.scrollWidth;
      // const scrollAmount = tableWidth / 2;
      // let newScrollX = this.scrollX + direction * scrollAmount;
      // if (newScrollX < 0) {
      //   newScrollX = 0;
      // } else if (newScrollX > tableScrollWidth - tableWidth) {
      //   newScrollX = tableScrollWidth - tableWidth;
      // }
      // this.scrollX = newScrollX;
    },
    addRow() {
      this.templateKeys.push({
        key: this.newKey.key,
        label: this.newKey.label,
        fige: this.newKey.fige,
        default: this.newKey.default
      });
      this.newKey = {key: '', label: '', fige: false, default: ''};
    },
    deleteRow(index) {
      this.templateKeys.splice(index, 1);
    },
    configValFige() {
      axios.post(this.api_path + '/template', {data: this.templateKeys})
        .then(response => {
          console.log(response.data.message);
        })
        .catch(error => {
          console.error(error.response.data.message);
        });

    },
    exportToExcel() {
      axios.post(this.api_path + '/export', {
        data: this.items,
        mapping: this.selectedMapping.mappings,
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
    async loadAvailableMappings() {
      await axios
        .get(this.api_path + "/mappings")
        .then((response) => {
          this.availableMappings = response.data;
          this.selectedMapping = this.availableMappings.find(e => e.filename == "#mapping_name");
          const selectedHeaders = {};
          for (const key of this.selectedMapping.mappings) {
            selectedHeaders[key.templateKey] = key.headerKey;
          }
          this.selectedHeaders = selectedHeaders;
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
    async saveMapping() {
      const mappings = Object.entries(this.selectedHeaders).map(([templateKey, headerKey]) => ({
        templateKey,
        headerKey,
      }));
      this.mappings = mappings; // do something with the mappings
      await axios
        .post(this.api_path + "/config", {
          name: "#mapping_name",
          mappings: this.mappings,
        })
        .then((response) => {
          console.log(response.data);
          this.dialogSaveMap = false;
          this.dialogVisible = false;
          this.mappingName = "";
          this.selectedHeaders = {};
        })
        .catch((error) => {
          console.error(error);
        });
      this.loadAvailableMappings();
    },
    async fetchTemplateKeys() {
      const response = await axios.get(this.api_path + '/template')
      this.templateKeys = response.data;
    },
    async fetchTable() {
      let config = {
        method: 'get',
        url: this.api_path + '/csv-to-json',
      };

      await axios.request(config)
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
      Array.from(this.csvFile).forEach(file => {
        formData.append('csv', file);
      });
      await axios.post(this.api_path + '/csv', formData, {
        headers: {
          'Content-Type': 'multipart/form-data'
        }
      })
        .catch((error) => {
          console.log(error);
        });

      this.fetchTable();

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
.v-data-table {
  margin-top: 10px;
  font-size: 0.7rem;
  color: #222;
}

.v-table > .v-table__wrapper > table > tbody > tr > th, .v-table > .v-table__wrapper > table > thead > tr > th, .v-table > .v-table__wrapper > table > tfoot > tr > th {
  background: mediumpurple;
  color: white;
}
</style>
