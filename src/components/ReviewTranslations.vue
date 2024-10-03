<template>
  <v-container grid-list-xs>
    <v-snackbar v-model="snackbar" timeout="2000">
      {{ snackbarText }}

      <template v-slot:action="{ attrs }">
        <v-btn
          :color="snackbarColor"
          text
          v-bind="attrs"
          @click="snackbar = false"
        >
        {{ $t(`App.hardcoded-texts.Close`) }}
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
          <v-toolbar color="primary" dark>{{ $t(`App.hardcoded-texts.Proceed?`) }}</v-toolbar>
          <v-card-text>
            {{ $t(`App.hardcoded-texts.This will overwrite all existing`) }} {{ language }} {{ $t(`App.hardcoded-texts.translations, do you want to proceed?`) }}
          </v-card-text>
          <v-card-actions class="justify-end">
            <v-btn text @click="dialog.value = false">
              <v-icon>mdi-close</v-icon>
              {{ $t(`App.hardcoded-texts.No`) }}
            </v-btn>
            <v-spacer></v-spacer>
            <v-btn text @click="googleTranslate">
              <v-icon>mdi-check</v-icon>
              {{ $t(`App.hardcoded-texts.Yes`) }}
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
            {{ language }} {{ $t(`App.hardcoded-texts.Translations`) }}
            <br />
            <br />
            <v-btn class="mx-2" dark color="indigo" small icon to="/">
              <v-icon>mdi-arrow-left-circle</v-icon>
              <v-tooltip activator="parent" location="top">
                {{ $t(`App.hardcoded-texts.Back to Enabled Languages List`) }}
              </v-tooltip>
            </v-btn>
          </v-col>
          <v-col v-if="$store.state.user.loggedin">
            <v-card width="300">
              <v-card-title primary-title> {{ $t(`App.hardcoded-texts.Translate with google`) }}</v-card-title>
              <v-card-text>
                <label
                  v-if="translationProgress.showTransProgress"
                  style="color: green"
                >
                {{ $t(`App.hardcoded-texts.Translation on progress`) }}
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
                        {{ $t(`App.hardcoded-texts.Translate all texts`) }}
                      </v-tooltip>
                      {{ $t(`App.hardcoded-texts.Full`) }}
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
                        {{ $t(`App.hardcoded-texts.Only missing translations`) }}
                      </v-tooltip>
                      {{ $t(`App.hardcoded-texts.Partial`) }}
                    </v-btn>
                  </v-col>
                </v-row>
              </v-card-text>
            </v-card>
          </v-col>
          <v-col>
            <v-card width="300" v-if="$store.state.user.loggedin">
              <v-card-title primary-title> {{ $t(`App.hardcoded-texts.Import/Export`) }} </v-card-title>
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
                      {{ $t(`App.hardcoded-texts.Export`) }}
                    </v-btn>
                    <v-progress-linear :indeterminate="true" height="20" v-else>
                      {{ $t(`App.hardcoded-texts.Preparing Export`) }}
                    </v-progress-linear>
                  </v-col>
                  <v-col>
                    <v-btn color="primary" small @click="importDialog = true">
                      <v-icon left>mdi-import</v-icon>
                      {{ $t(`App.hardcoded-texts.Import`) }}
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
                <tr @click="edit(item.value)" style="cursor: pointer">
                  <td>{{ ++index }}</td>
                  <td>{{ limitTexts(item.value.en) }}</td>
                  <td>{{ limitTexts(item.value.text) }}</td>
                </tr>
              </template>
            </v-data-table>
          </v-card-text>
        </v-card>
      </v-col>
      <v-col v-if="selected.key">
        <v-card>
          <v-toolbar color="secondary" dark height="30">
            {{ $t(`App.hardcoded-texts.Edit Translation`) }}
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
              <span>{{ $t(`App.hardcoded-texts.Close translation dialog`) }}</span>
            </v-tooltip>
          </v-toolbar>
          <v-card-text>
            {{ $t(`App.hardcoded-texts.WORD:`) }}
            <br />
            <i
              ><b>{{ selected.en }}</b></i
            ><br /><br />
            {{ $t(`App.hardcoded-texts.TRANSLATION:`) }}
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
              {{ $t(`App.hardcoded-texts.Save`) }}
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
      transRunType: "",
      headers: [
        {
          text: "SN",
          value: "sn",
        },
        {
          text: "Word",
          value: "en",
        },
        {
          text: "Translated To",
          value: "text",
        },
      ],
      translationProgress: {
        showTransProgress: false,
        required: 0,
        translated: 0,
        percent: 0,
        interval: "",
      },
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
      this.selected = val;
      this.newTranslation = val.text;
    },
    closeEdit() {
      this.selected = {};
    },
    save() {
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
  },
  components: {
    ImportTranslations,
  },
  created() {
    this.language = this.$route.params.locale;
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
  beforeUnmount() {
    clearInterval(this.translationProgress.interval);
  },
};
</script>
