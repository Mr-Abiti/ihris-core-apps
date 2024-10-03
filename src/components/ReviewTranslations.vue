<template>
  <v-container v-if="$store.state.user.loggedin" grid-list-xs>
    <v-snackbar v-model="snackbar" timeout="2000">
      {{ snackbarText }}

      <template v-slot:action="{ attrs }">
        <v-btn
          :color="snackbarColor"
          text
          v-bind="attrs"
          @click="snackbar = false"
        >
          Close
        </v-btn>
      </template>
    </v-snackbar>
    <v-dialog
      transition="dialog-top-transition"
      max-width="600"
      v-model="confirmGTrans"
    >
      <template v-slot:default="dialog">
        <v-card>
          <v-toolbar color="primary" dark>Proceed?</v-toolbar>
          <v-card-text>
            This will overwrite all existing {{ language }} translations, do you
            want to proceed?
          </v-card-text>
          <v-card-actions class="justify-end">
            <v-btn text @click="dialog.value = false">
              <v-icon>mdi-close</v-icon>
              No
            </v-btn>
            <v-spacer></v-spacer>
            <v-btn text @click="googleTranslate">
              <v-icon>mdi-check</v-icon>
              Yes
            </v-btn>
          </v-card-actions>
        </v-card>
      </template>
    </v-dialog>
    <ImportTranslations
      :importDialog="importDialog"
      :locale="this.$route.params.locale"
      :language="language"
    ></ImportTranslations>
    <v-card>
      <v-card-title primary-title>
        <v-row>
          <v-col>
            {{ language }} Translations
            <br />
            <br />
            <v-btn class="mx-2" dark color="indigo" small icon to="/">
              <v-icon>mdi-arrow-left-circle</v-icon>
              <v-tooltip activator="parent" location="top">
                Back to Enabled Languages List
              </v-tooltip>
            </v-btn>
          </v-col>
          <v-col v-if="$store.state.user.loggedin">
            <v-card width="300">
              <v-card-title primary-title> Translate with google </v-card-title>
              <v-card-text>
                <label
                  v-if="translationProgress.showTransProgress"
                  style="color: green"
                >
                  Translation on progress
                </label>
                <v-row>
                  <v-col>
                    <v-btn
                      color="primary"
                      :disabled="translationProgress.showTransProgress"
                      small
                      @click="displayTransConf('full')"
                      v-bind="attrs"
                      v-on="on"
                    >
                      <v-icon left>mdi-google-translate</v-icon>
                      <v-tooltip activator="parent" location="bottom">
                        Translate all texts
                      </v-tooltip>
                      Full
                    </v-btn>
                  </v-col>
                  <v-col>
                    <v-btn
                      color="blue"
                      dark
                      :disabled="translationProgress.showTransProgress"
                      small
                      @click="displayTransConf('partial')"
                      v-bind="attrs"
                      v-on="on"
                    >
                      <v-icon left>mdi-google-translate</v-icon>
                      <v-tooltip activator="parent" location="bottom">
                        Only missing translations
                      </v-tooltip>
                      Partial
                    </v-btn>
                  </v-col>
                </v-row>
              </v-card-text>
            </v-card>
          </v-col>
          <v-col>
            <v-card width="300" v-if="$store.state.user.loggedin">
              <v-card-title primary-title> Import/Export </v-card-title>
              <v-card-text>
                <v-row>
                  <v-col>
                    <v-btn
                      color="primary"
                      small
                      @click="exportTranslation"
                      v-if="!exporting"
                    >
                      <v-icon left>mdi-export</v-icon>
                      Export
                    </v-btn>
                    <v-progress-linear :indeterminate="true" height="20" v-else>
                      Preparing Export
                    </v-progress-linear>
                  </v-col>
                  <v-col>
                    <v-btn color="primary" small @click="importDialog = true">
                      <v-icon left>mdi-import</v-icon>
                      Import
                    </v-btn>
                  </v-col>
                </v-row>
              </v-card-text>
            </v-card>
          </v-col>
          <v-col>
            <v-text-field
              v-model="search"
              label="Search"
              class="mx-4"
              clearable
              prepend-icon="mdi-magnify"
            ></v-text-field>
          </v-col>
        </v-row>
      </v-card-title>
    </v-card>
    <v-row>
      <v-col>
        <v-card>
          <v-card-text>
            <v-progress-linear
              v-model="translationProgress.percent"
              height="25"
              buffer-value="0"
              stream
              v-if="translationProgress.showTransProgress"
            >
              <strong>
                {{ translationProgress.translated }}/{{
                  translationProgress.required
                }}
              </strong>
            </v-progress-linear>
            <v-data-table
              :headers="headers"
              :items="translations"
              :search="search"
              :loading="loading"
              loading-text="Loading"
            >
              <template v-slot:item="{ item, index }">
                <tr
                  v-if="item.isCodeSystem"
                  style="cursor: pointer"
                  @click="edit(item)"
                >
                  <td>{{ ++index }}</td>
                  <td>
                    {{ item.name }}
                    <v-chip
                      class="ma-2"
                      color="primary"
                      size="small"
                      variant="flat"
                    >
                      Dropdown
                    </v-chip>
                  </td>
                  <td>{{ item.description }}</td>
                  <td>{{ item.translated }}</td>
                  <td>
                    <v-btn
                      class="ma-1"
                      color="primary"
                      outlined
                      x-small
                      @click.stop="translate(item)"
                    >
                      Translate
                    </v-btn>
                  </td>
                </tr>
                <tr v-else style="cursor: pointer" @click="edit(item)">
                  <td>{{ ++index }}</td>
                  <td>{{ limitTexts(item.en) }}</td>
                  <td>{{ limitTexts(item.text) }}</td>
                </tr>
              </template>
            </v-data-table>
          </v-card-text>
        </v-card>
      </v-col>
      <v-col v-if="selected.key || selected?.resourceType === 'CodeSystem'">
        <v-card>
          <v-toolbar color="secondary" dark height="30">
            Edit Translation
            <v-spacer></v-spacer>
            <v-tooltip top>
              <template v-slot:activator="{ on, attrs }">
                <v-btn
                  fab
                  icon
                  color="white"
                  @click="closeEdit"
                  v-bind="attrs"
                  v-on="on"
                >
                  <v-icon>mdi-close</v-icon>
                </v-btn>
              </template>
              <span>Close translation dialog</span>
            </v-tooltip>
          </v-toolbar>
          <v-card-text>
            WORD:
            <br />
            <i
              ><b>{{ selected.en || `${selected.name} - Dropdown` }}</b></i
            ><br /><br />
            TRANSLATION:
            <v-textarea
              clearable
              clear-icon="mdi-close-circle"
              v-model="newTranslation"
              style="
                background-color: #ffffc2;
                font-family: 'Franklin Gothic Medium', 'Arial Narrow', Arial,
                  sans-serif;
              "
            ></v-textarea>
            <v-spacer></v-spacer>
            <v-btn
              small
              rounded
              color="primary"
              dark
              @click="save"
              v-if="$store.state.user.loggedin"
            >
              <v-icon left>mdi-content-save</v-icon>
              Save
            </v-btn>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>
<script>
import ImportTranslations from "./ImportTranslations.vue";

export default {
  data() {
    return {
      language: "",
      languageCode: "",
      snackbarColor: "green",
      snackbarText: "",
      snackbar: false,
      confirmGTrans: false,
      importDialog: false,
      exporting: false,
      loading: true,
      search: "",
      selected: {},
      newTranslation: "",
      translations: [],
      dropDownList: [],
      transRunType: "",
      headers: [
        {
          text: "SN",
          value: "sn",
        },
        {
          text: "Name",
          value: "name",
        },
        {
          text: "description",
          value: "description",
        },
        {
          text: "Translated",
          value: "translated",
        },
        {
          text: "Actions",
          value: "actions",
          sortable: false,
        },
      ],
      translationProgress: {
        showTransProgress: false,
        required: 0,
        translated: 0,
        percent: 0,
        interval: "",
      },
      options: { itemsPerPage: 10 },
      total: 0,
      codeSystemData: [],
    };
  },
  methods: {
    limitTexts(val) {
      if (val.length < 100) {
        return val;
      }
      return val.substring(0, 100) + "...";
    },
    displayTransConf(type) {
      this.transRunType = type;
      if (type === "full") {
        this.confirmGTrans = true;
      } else {
        this.googleTranslate();
      }
    },
    edit(val) {
      if (val.isCodeSystem) {
        fetch(`/fhir/CodeSystem/${val.id}`).then((response) => {
          response.json().then((data) => {
            this.selected = data;
            let concept = data?.concept;
            if (concept) {
              let filteredData = {};
              concept.map((item) => {
                let display = item.display;
                let translated = item.designation.find(
                  (x) => x.language === this.languageCode
                )?.value;
                filteredData[display] = translated;
              });
              this.newTranslation = JSON.stringify(filteredData, null, 2);
            }
          });
        });
      } else {
        this.selected = val;
        this.newTranslation = val.text;
      }
    },
    closeEdit() {
      this.selected = {};
    },
    save() {
      this.newTranslation = JSON.parse(this.newTranslation);
      if (this.newTranslation) {
        for (const [key, value] of Object.entries(this.newTranslation)) {
          this.selected.concept.map((item) => {
            let display = item.display;
            if (display === key) {
              let languageValue = item.designation.find(
                (x) => x.language === this.languageCode
              );
              if (languageValue) {
                languageValue.value = value;
              }
            }
          });
        }
      }
      if (this.selected?.resourceType === "CodeSystem") {
        fetch(`/translator/updateCodeSystem`, {
          method: "PUT",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(this.selected),
        })
          .then((response) => {
            if (response.status === 200) {
              this.selected = {};
              this.newTranslation = undefined;
              this.setup();
              this.snackbar = true;
              this.snackbarColor = "green";
              this.snackbarText = "Translation Updated";
            }
          })
          .catch((err) => {
            console.log(err);
            this.snackbar = true;
            this.snackbarColor = "red";
            this.snackbarText = "Error Occured";
          });
      } else {
        fetch("/translator/update", {
          method: "PUT",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            locale: this.$route.params.locale,
            path: this.selected.key,
            text: this.newTranslation,
          }),
        })
          .then((response) => {
            if (response.status === 200) {
              this.getTranslations();
              this.snackbar = true;
              this.snackbarColor = "green";
              this.snackbarText = "Translation Updated";
            }
          })
          .catch(() => {
            this.snackbar = true;
            this.snackbarColor = "red";
            this.snackbarText = "Error Occured";
          });
      }
    },
    getTranslations(silent) {
      if (!silent) {
        this.loading = true;
      }
      fetch("/translator/getTranslations/" + this.$route.params.locale)
        .then((response) => {
          response.json().then((trans) => {
            this.loading = false;
            this.translations = trans.translations;
            this.language = trans.language;
          });
        })
        .catch((err) => {
          console.log(err);
        });
    },
    googleTranslate() {
      this.confirmGTrans = false;
      fetch(
        "/translator/translate/en/" +
          this.$route.params.locale +
          "/" +
          this.transRunType
      )
        .then((response) => {
          if (response.status === 200) {
            this.translationProgress.interval = setInterval(() => {
              this.googleTranslateCount();
            }, 1000);
          }
          this.translateAll();
        })
        .catch(() => {
          this.snackbar = true;
          this.snackbarColor = "red";
          this.snackbarText = "Error Occured During Translation";
        });
    },
    googleTranslateCount() {
      this.translationProgress.showTransProgress = true;
      fetch("/translator/translationCount/en/" + this.$route.params.locale)
        .then((response) => {
          response.json().then((count) => {
            this.getTranslations(true);
            this.translationProgress.required = count.from;
            this.translationProgress.translated = count.to;
            this.translationProgress.percent =
              (parseInt(count.to) * 100) / parseInt(count.from);
            if (count.from === count.to || !count.running) {
              clearInterval(this.translationProgress.interval);
              this.translationProgress.showTransProgress = false;
              this.snackbar = true;
              this.snackbarColor = "green";
              this.snackbarText = "Translation completed successfully";
            }
          });
        })
        .catch(() => {
          this.snackbar = true;
          this.snackbarColor = "red";
          this.snackbarText = "Cant get progress";
        });
    },
    exportTranslation() {
      this.exporting = true;
      fetch("/translator/export/" + this.$route.params.locale)
        .then((response) => response.blob())
        .then((blob) => {
          this.exporting = false;
          const url = window.URL.createObjectURL(blob);
          const a = document.createElement("a");
          a.href = url;
          a.download = this.language + ".xlsx";
          document.body.appendChild(a);
          a.click();
          a.remove();
        })
        .catch(() => {
          this.snackbar = true;
          this.snackbarColor = "red";
          this.snackbarText = "Cant download translation";
          this.exporting = false;
        });
    },
    translate(item) {
      this.loading = true;
      let url =
        "/translator/codeSystemTranslations/" +
        this.$route.params.locale +
        "/" +
        item.id;
      fetch(url).then((response) => {
        response
          .json()
          .then(() => {
            this.loading = false;
            this.setup();
          })
          .catch((err) => {
            console.log(err);
            this.loading = false;
          });
      });
    },
    translateAll() {
      this.loading = true;
      fetch("/translator/translateAllCodeSystem/" + this.$route.params.locale)
        .then((response) => {
          response.json().then(() => {
            this.loading = false;
            this.getCodeSystem();
          });
        })
        .catch((err) => {
          this.loading = false;
          console.log(err);
        });
    },
    getCodeSystem() {
      this.loading = true;
      fetch("/fhir/CodeSystem?_count=200&_sort=name")
        .then((response) => {
          response.json().then((code) => {
            code.entry.map((x) => {
              let data = {
                isCodeSystem: true,
                id: x.resource.id,
                name: x.resource.name,
                translated: x.resource?.concept?.[0]?.designation
                  ?.map((designation) => designation.language)
                  .join(", "),
                description: x.resource.description,
              };
              this.translations.push(data);
            });
            this.loading = false;
          });
        })
        .catch((err) => {
          this.loading = false;
          console.log(err);
        });
    },
    setup() {
      this.getCodeSystem();
      this.language = this.$route.params.locale;
      this.languageCode = this.$route.params.locale;
      this.getTranslations();
      fetch("/translator/translationCount/en/" + this.$route.params.locale)
        .then((response) => {
          response.json().then((translation) => {
            if (translation.running) {
              this.translationProgress.showTransProgress = true;
              this.translationProgress.interval = setInterval(() => {
                this.googleTranslateCount();
              }, 1000);
            }
          });
        })
        .catch(() => {
          this.snackbar = true;
          this.snackbarColor = "red";
          this.snackbarText = "Cant get translation progress";
        });
      this.emitter.on("closeImportDialog", () => {
        this.importDialog = false;
      });
    },
  },
  components: {
    ImportTranslations,
  },
  created() {
    this.setup();
  },
  beforeUnmount() {
    clearInterval(this.translationProgress.interval);
  },
  computed: {
    itemsPerPage() {
      return [5, 10, 50, 100];
    },
  },
};
</script>
