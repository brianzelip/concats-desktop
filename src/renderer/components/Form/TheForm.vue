<template>
  <form name="concatsForm">
    <TheResetBtn
      class="reset"
      v-if="fileHasBeenProcessed"
      v-on:reset-app="resetApp"
    ></TheResetBtn>

    <transition
      mode="out-in"
      name="fade"
    >
      <component
        :is="currentSelector"
        v-bind="currentSelectorProps"
        v-on:file-input="handleInputFile"
        v-on:user-selected-headers-change="updateUserSelectedHeaders"
        v-on:user-selected-headers-submitted="setCsvOutput"
      ></component>
    </transition>
  </form>
</template>

<script>
const fs = require("fs");
const { dialog } = require("electron").remote;
import { ipcRenderer } from "electron";
import CSV from "csvtojson";

import TheFileSelector from "./TheFileSelector.vue";
import TheHeadersSelector from "./TheHeadersSelector.vue";
import TheOutput from "./TheOutput.vue";
import TheResetBtn from "./TheResetBtn.vue";

export default {
  data() {
    return {
      csvInput: "",
      csvInputHeaders: [],
      userSelectedHeaders: [],
      csvAsJson: [],
      csvOutput: "",
      submitted: false,
      currentSelector: "TheFileSelector"
    };
  },
  computed: {
    fileHasBeenProcessed() {
      if (this.csvAsJson.length > 0) {
        this.$emit("file-has-been-processed");
        return true;
      }
      return false;
    },
    headersHaveBeenSubmitted() {
      return this.csvInputHeaders.length > 0 && this.csvOutput.length > 0;
    },
    currentSelectorProps() {
      return this.currentSelector === "TheFileSelector"
        ? {}
        : this.currentSelector === "TheHeadersSelector"
        ? { headers: this.csvInputHeaders }
        : { csvOutput: this.csvOutput };
    }
  },
  components: {
    TheFileSelector,
    TheHeadersSelector,
    TheOutput,
    TheResetBtn
  },
  methods: {
    handleInputFile(file) {
      const vm = this;
      const reader = new FileReader();

      const convertFileToJson = fileContent => {
        vm.csvInput = fileContent;
        CSV({ delimiter: ["\t", ","] })
          .fromString(fileContent)
          .on("header", header => {
            vm.setCsvInputHeaders(header);
          })
          .then(json => {
            vm.setCsvAsJson(json);
          })
          .then(() => {
            vm.currentSelector = "TheHeadersSelector";
          });
      };

      if (typeof file === "string") {
        convertFileToJson(file);
      } else if (typeof file === "object") {
        reader.readAsText(file);
        reader.onload = function(event) {
          convertFileToJson(event.target.result);
        };
      }
    },
    setCsvInputHeaders(data) {
      this.csvInputHeaders = data;
    },
    setCsvAsJson(data) {
      this.csvAsJson = data;
    },
    updateUserSelectedHeaders(data) {
      this.userSelectedHeaders = data;
    },
    setCsvOutput() {
      this.csvOutput = this.csvAsJson
        .reduce((acc, obj) => {
          const keys = this.userSelectedHeaders;
          var concattedString = "";
          keys.forEach((key, index) => {
            index !== keys.length - 1
              ? (concattedString = concattedString.concat(`${obj[key]} `))
              : (concattedString = concattedString.concat(obj[key]));
          });
          acc.push(concattedString);
          return acc;
        }, [])
        .join("\n");
      this.currentSelector = "TheOutput";
      this.$emit("user-selected-headers-submitted");
      this.saveFile(this.csvOutput);
    },
    saveFile(data) {
      dialog.showSaveDialog(
        {
          filters: [{ name: "csv", extensions: ["csv"] }]
        },
        fileName => {
          if (fileName === undefined) return;
          fs.writeFile(fileName, data);
        }
      );
    },
    resetApp() {
      this.csvInput = "";
      this.csvInputHeaders = [];
      this.userSelectedHeaders = [];
      this.csvAsJson = [];
      this.csvOutput = "";
      this.submitted = false;
      this.currentSelector = "TheFileSelector";
      this.$emit("reset");
    }
  },
  mounted() {
    ipcRenderer.on("reset-app", () => {
      this.resetApp();
    });
    ipcRenderer.on("file-input", (e, filePath) => {
      this.handleInputFile(filePath);
    });
  }
};
</script>

<style scoped>
form {
  display: flex;
  margin: 1rem;
}
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
.reset {
  position: absolute;
  top: 1rem;
  left: 1rem;
}
</style>
