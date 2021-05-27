<template>
  <v-app>
    <v-main>
      <toolbar />
      <v-container class="mx-auto my-12">
        <v-row>
          <v-col cols="12" sm="6">
            <v-card elevation="7" shaped>
              <v-card-text style="height: 245px">
                <v-row
                  v-if="url"
                  class="d-flex flex-column"
                  dense
                  align="center"
                  justify="center"
                >
                  <img ref="image" :src="url" width="300" />
                </v-row>
                <v-row
                  v-else
                  class="d-flex flex-column"
                  dense
                  align="center"
                  justify="center"
                >
                  <v-icon class="mt-5" size="150"> mdi-cloud-upload </v-icon>
                  <p>Silahkan pilih gambar yang ingin anda proses.</p>
                </v-row>
              </v-card-text>
              <v-card-actions>
                <v-spacer></v-spacer>
                <input
                  @change="onFileChange"
                  type="file"
                  ref="file"
                  accept="image/*"
                  style="display: none"
                />
                <v-btn
                  @click="$refs.file.click()"
                  outlined
                  text
                  color="info"
                  :dark="!proses"
                  :disabled="proses"
                >
                  Upload
                  <v-icon right>mdi-upload</v-icon>
                </v-btn>
                <v-btn
                  @click="recognize"
                  color="blue"
                  :dark="!!url && !proses"
                  :disabled="!url || proses"
                >
                  Proses
                  <v-icon right>mdi-send</v-icon>
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-col>
          <v-col cols="12" sm="6">
            <v-card elevation="10">
              <v-card-text>
                <v-container style="height: 260px">
                  <v-row
                    v-if="proses"
                    class="fill-height"
                    align-content="center"
                    justify="center"
                  >
                    <v-col class="subtitle-1 text-center" cols="12">
                      {{ status }}
                    </v-col>
                    <v-col cols="12">
                      <v-progress-linear :value="progress"></v-progress-linear>
                    </v-col>
                  </v-row>
                  <v-row v-else>
                    <v-form ref="form">
                      <v-row>
                        <v-col cols="12" sm="6" md="6">
                          <v-text-field
                            v-model="textParsing.nik"
                            type="number"
                            label="NIK"
                            required
                          ></v-text-field>
                        </v-col>
                        <v-col cols="12" sm="6" md="6">
                          <v-text-field
                            v-model="textParsing.nama"
                            label="Nama"
                            required
                          ></v-text-field>
                        </v-col>
                        <v-col cols="12" sm="6" md="6">
                          <v-text-field
                            v-model="textParsing.tempatlahir"
                            label="Tempat Lahir"
                            required
                          ></v-text-field>
                        </v-col>
                        <v-col cols="12" sm="6" md="6">
                          <v-text-field
                            v-model="textParsing.tgllahir"
                            label="Tanggal Lahir"
                            required
                          ></v-text-field>
                        </v-col>
                        <v-col cols="12" sm="12" md="12">
                          <v-text-field
                            v-model="textParsing.alamat"
                            label="Alamat"
                            required
                          ></v-text-field>
                        </v-col>
                      </v-row>
                    </v-form>
                  </v-row>
                </v-container>
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<script lang="ts">
import Vue from "vue";
import Toolbar from "../components/Toolbar.vue";

/* eslint-disable */
import { createWorker, PSM, OEM } from "tesseract.js";
import jimp from "jimp";

interface HTMLInputEvent extends Event {
  target: HTMLInputElement & EventTarget;
}

export default Vue.extend({
  name: "Home",

  components: {
    Toolbar,
  },
  data() {
    return {
      url: "",
      proses: false,
      status: "loading",
      progress: 0,
      text: ``,
    };
  },
  computed: {
    textParsing() {
      let nik = "",
        nama = "",
        tempatlahir = "",
        tgllahir = "",
        alamat = "";

      const textArr = this.text?.split("\n");
      try {
        for (const word of textArr) {
          if (word.includes("NIK")) {
            nik = this.nikParsing(word);
          }
          if (word.includes("Nama")) {
            nama = this.namaParsing(word);
          }
          if (word.includes("Tempat")) {
            const ttl = this.ttlParsing(word);
            console.log(ttl);
            tempatlahir = ttl[0].trim();
            tgllahir = ttl[1].trim();
          }
          if (word.includes("Alamat")) {
            alamat = this.alamatParsing(word);
          }
        }
      } catch (error) {
        console.log(error);
      }
      return { nik, nama, tempatlahir, tgllahir, alamat };
    },
  },
  methods: {
    nikParsing(word: string) {
      return word.replace(/\D/g, "");
    },
    namaParsing(word: string) {
      return word
        .replace("Nama", "")
        .replace(/[^a-zA-Z\s]+/g, "")
        .trim();
    },
    ttlParsing(word: string) {
      const wordArr = word.split(word.includes(" : ") ? " : " : " - ");
      return wordArr[1].split(wordArr[1].includes(".") ? "." : ",");
    },
    alamatParsing(word: string) {
      return word
        .replace("Alamat", " ")
        .replace(/[^a-zA-Z\s]+/g, "")
        .trim();
    },
    async onFileChange(event: HTMLInputEvent) {
      const files = event?.target?.files;
      if (!files?.length) return;
      this.url = URL.createObjectURL(files[0]);
    },
    async recognize() {
      this.proses = true;
      const worker = createWorker({
        logger: (m) => {
          this.status = m.status;
          this.progress = m.progress * 100;
        },
      });

      await worker.load();
      await worker.loadLanguage("eng");
      await worker.initialize("eng", OEM.LSTM_ONLY);
      await worker.setParameters({
        tessedit_pageseg_mode: PSM.SINGLE_BLOCK,
      });

      const image = await jimp.read(this.url);
      image.threshold({ max: 255, replace: 255 });

      const newImg = await image.getBase64Async(jimp.MIME_JPEG);

      this.url = newImg;

      const {
        data: { text },
      } = await worker.recognize(newImg);
      console.log(text);
      this.text = text;
      this.proses = false;
    },
  },
});
</script>
