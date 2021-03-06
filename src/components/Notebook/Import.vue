<template>
  <div>
    <a-modal
        v-model="showDialog"
        title="Import Notebook"
        onOk="handleOk"
        :maskClosable="false"
      >
      <template slot="footer">
        <a-button key="back" @click="handleCancel">Cancel</a-button>
        <a-button key="submit" type="primary" :loading="loading" @click="handleOk">Import</a-button>
      </template>

      <a-form layout="vertical">
        <a-form-item>
          <a-radio-group
            v-model="importType"
            buttonStyle="solid"
            name="import-type"
            @change="changeImportType"
            style="display: block; margin: 0 auto; width: 238px;"
          >
            <a-radio-button value="url">Provide a URL</a-radio-button>
            <a-radio-button value="file">Upload a File</a-radio-button>
          </a-radio-group>
        </a-form-item>

        <div
          v-if="importType === 'file'"
          id="notebook-import-drag-drop"
          class="ant-upload ant-upload-drag"
          v-bind:class="{'hover': isFileDragHover}"
          style="margin-bottom: 16px; display: block;"
          @click="showFileDialog"
          @change="handleChange"
          @drop.prevent="handleChange"
          @dragover.prevent="handleDragHover"
          @dragleave.prevent="handleDragHover"
        >
          <input
            type="file"
            name="nbFileInput"
            ref="nbFileInput"
            style="display: none;"
          />
          <p class="ant-upload-drag-icon" style="margin-bottom: 10px;">
            <a-icon type="inbox" />
          </p>
          <p class="ant-upload-text" style="font-size: 13px;">
            <span v-if="sourceNotebookFile">{{ this.sourceNotebookFile.name }}</span>
            <span v-else>{{ $t("message.notebooks.import_click_or_drag") }}</span>
          </p>
        </div>

        <a-form-item
          v-if="importType === 'url'"
          label="Notebook URL"
        >
          <a-input placeholder="Notebook URL"  v-model="notebookURL"/>
        </a-form-item>

        <a-form-item
          label="Notebook Name"
        >
          <a-input placeholder="Enter Notebook Name"  v-model="name"/>
        </a-form-item>

        <!-- <a-form-item
          label="Default Interpreter"
        >
          <a-select
            showSearch
            style="width: 180px"
            :defaultValue="interpreters[0]"
            v-model="defaultInterpreter"
          >
            <a-select-option
              v-for="interpreter in interpreters"
              v-bind:key="interpreter.id"
            >
              {{interpreter.id}}
            </a-select-option>
          </a-select>
        </a-form-item> -->
      </a-form>
    </a-modal>
  </div>
</template>

<script>
import { EventBus } from '@/services/event-bus'

export default {
  name: 'ImportNotebook',
  data () {
    return {
      showDialog: false,
      loading: false,
      isFileDragHover: false,

      name: '',
      defaultInterpreter: null,

      importType: 'file',
      sourceNotebookFile: null,
      sourceNotebookJSON: null,
      fileUploaded: false
    }
  },
  computed: {
    interpreters () {
      return this.$store.state.InterpreterStore.interpreters
    }
  },
  mounted () {
    EventBus.$on('showImportNotebookDialog', () => {
      this.showDialog = true
    })
  },
  methods: {
    showFileDialog () {
      this.$refs.nbFileInput.click()
    },
    handleChange (e) {
      this.handleDragHover(e)

      let files = e.target.files || e.dataTransfer.files
      let validated = files.length !== 0 && this.beforeUpload(files[0])
      if (validated) {
        this.sourceNotebookFile = files[0]

        let reader = new FileReader()
        reader.readAsText(this.sourceNotebookFile)
        reader.onloadend = () => {
          this.sourceNotebookJSON = JSON.parse(reader.result)
          this.fileUploaded = true
        }
      } else {
        this.sourceNotebookFile = null
        this.sourceNotebookJSON = null
        this.fileUploaded = false
      }
    },
    beforeUpload (file) {
      const isJSON = (file.type === 'application/json')
      if (!isJSON) {
        this.$message.error(this.$i18n.t('message.notebooks.import_json_type_error'), 4)
      }
      const isLt1M = file.size / 1024 / 1024 < 1
      if (!isLt1M) {
        this.$message.error(this.$i18n.t('message.notebooks.import_json_size_error'), 4)
      }
      return isJSON && isLt1M
    },
    handleOk (e) {
      this.loading = true

      this.sourceNotebookJSON.name = this.name
      this.$root.executeCommand('notebook', 'import-json', this.sourceNotebookJSON)

      // let that = this
      setTimeout(() => {
        this.showDialog = false
        this.loading = false

        this.resetForm()

        this.$message.success(this.$i18n.t('message.notebooks.import_success'), 4)
        // Pending - validation
        // Pending open - imported Notebook
      }, 1000)
    },
    handleCancel (e) {
      this.showDialog = false
      this.resetForm()
    },
    resetForm () {
      this.name = ''
      this.defaultInterpreter = null

      this.importType = 'file',
      this.sourceNotebookFile = null
      this.sourceNotebookJSON = null
      this.fileUploaded = false
    },
    changeImportType (e) {
      this.importType = e.target.value
    },
    handleDragHover (e) {
      this.isFileDragHover = e.type === 'dragover'
    }
  }
}
</script>

<style lang="scss">
.hover {
  border-color: #505050 !important;
}
</style>
