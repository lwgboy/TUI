<template>
  <div class="t-upload">
    <div
      v-if="$slots.default"
      class="t-upload__trigger"
      @click="fireUpload"
      ref="trigger">
      <slot/>
    </div>

    <div
      v-if="!$slots.default && drag"
      class="t-upload__trigger"
      @click="fireUpload"
      ref="trigger">
      <div class="t-upload__drop-area">
        <i class="fa fa-cloud-upload-alt"></i>
        <p>拖进文件 或 <span>点击上传</span></p>
      </div>
    </div>
    <div
      class="t-upload__tip"
      v-if="$slots.tip">
      <slot name="tip"/>
    </div>
    <div
      class="t-upload__file-list"
      v-if="showFileList && !listType">
      <div
        class="t-upload__file-info"
        v-for="(f, idx) in fileList"
        :key="idx">
        <div class="t-upload__file-name">
          <span class="t-upload__name">
            <i class="fa fa-file-alt"></i>
            {{ f.file.name }}
          </span>
          <span class="t-upload__alt">
            <i class="fa fa-check-circle" v-show="f.uploadSuccess"/>
            <t-tooltip
              content="重新上传"
              theme="dark"
              position="right">
              <i class="fa fa-exclamation-circle" v-show="f.uploadError" @click="uploadFile(f)"/>
            </t-tooltip>
            <i class="fa fa-arrow-alt-circle-up" v-show="f.prepearUpload" @click="uploadFile(f)"/>
            <i class="fa fa-times" v-show="!f.loading" @click="removeFile(f)"/>
          </span>
        </div>
        <transition name="fly-top">
          <div
            class="t-upload__file-progress"
            v-if="f.loading">
            <t-progress
              :percentage="f.percent"
              hide-percentage
              v-if="f.loading"/>
         </div>
        </transition>
      </div>
    </div>
    <div
      class="t-upload__file-list t-upload__file-list--pic"
      v-if="showFileList && listType === 'pic' && fileList.length > 0">
      <div
        class="t-upload__file-info"
        v-for="(f, idx) in fileList"
        :key="idx">
        <div class="t-upload__pic" @click="preview(f)">
          <img :src="f.uri">
        </div>
        <div class="t-upload__file-name">
          <span class="t-upload__name">
            <i class="fa fa-file-alt"></i>
            {{ f.file.name }}
          </span>
          <span class="t-upload__alt">
            <i class="fa fa-check-circle" v-show="f.uploadSuccess"/>
            <t-tooltip
              content="重新上传"
              ref="reupload"
              theme="dark"
              position="right">
              <i class="fa fa-exclamation-circle" v-show="f.uploadError" @click="uploadFile(f)"/>
            </t-tooltip>
            <i class="fa fa-arrow-alt-circle-up" v-show="f.prepearUpload" @click="uploadFile(f)"/>
            <i class="fa fa-times" v-show="!f.loading" @click="removeFile(f)"/>
          </span>
        </div>
        <transition name="fly-top">
          <div
            class="t-upload__file-progress"
            v-if="f.loading">
            <t-progress
              :percentage="f.percent"
              v-if="f.loading"/>
          </div>
        </transition>
      </div>
    </div>
    <input
      type="file"
      style="display: none"
      ref="upload"
      :name="name"
      @change="onFileLoad"
      :multiple="multiple"
      :webkitdirectory="webkitdirectory"
      :accept="accept"
      :disaebled="disabled"
    >

    <t-modal :show-close="false" :show.sync="isPreview" hide-on-click style="text-align: center">
      <template slot="body">
        <img :src="previewUri" style="width: 100%">
      </template>
    </t-modal>
  </div>
</template>

<script>
import Uploader from './uploader'
import TFile from './tfile'
export default {
  name: 't-upload',

  data () {
    return {
      fileList: [],
      isPreview: false,
      previewUri: '',
      uploader: new Uploader(
        this.action,
        this.method,
        this.name,
        this.header,
        this.withCredentials
      ),
      uploadQueue: []
    }
  },

  props: {
    showFileList: {
      type: Boolean,
      default: true
    },
    action: {
      type: String,
      required: true
    },
    method: {
      type: String,
      default: 'post'
    },
    //  input name
    name: {
      type: String,
      default: 'file'
    },
    headers: {
      type: Object
    },

    withCredentials: Boolean,
    autoUpload: {
      type: Boolean,
      default: true
    },

    multiple: Boolean,
    drag: Boolean,
    webkitdirectory: Boolean,
    accept: String,
    disabled: Boolean,

    beforeUpload: Function,
    onUploadSuccess: Function,
    onUploadError: Function,
    beforeRemove: Function,
    onRemove: Function,
    onFileListChange: Function,
    onPreview: Function,
    onExceed: Function,

    listType: String,

    limit: Number
  },

  mounted () {
    this.drag && this.bindDragTrigger()
  },

  //  TODO: 2.validate; => default config params
  methods: {
    fireUpload () {
      !this.disabled && this.$refs.upload.click()
    },
    bindDragTrigger () {
      this.$refs.trigger.addEventListener('dragover', this.onDragOver)
      this.$refs.trigger.addEventListener('drop', this.onDrop)
    },
    removeDragTrigger () {
      this.$refs.trigger.removeEventListener('dragover', this.onDragOver)
      this.$refs.trigger.removeEventListener('drop', this.onDrop)
    },
    preview (tfile) {
      this.previewUri = tfile.uri
      this.isPreview = true
      this.onPreview && this.onPreview(tfile)
    },
    onDragOver (e) {
      e.preventDefault()
    },
    onDrop (e) {
      // Prevent default behavior (Prevent file from being opened)
      e.preventDefault()

      if (e.dataTransfer.items) {
        // Use DataTransferItemList interface to access the file(s)
        for (let i = 0; i < e.dataTransfer.items.length; i++) {
          let entry = e.dataTransfer.items[i].webkitGetAsEntry()
          // If dropped items aren't files, reject them
          if (entry.isFile) {
            let file = e.dataTransfer.items[i].getAsFile()
            this.loadFile(file)
          } else if (entry.isDirectory) {
            this.readDirectoryFiles(entry)
          }
        }
      } else {
        // Use DataTransfer interface to access the file(s)
        for (let i = 0; i < e.dataTransfer.files.length; i++) {
          this.loadFile(e.dataTransfer.files[i])
        }
      }

      this.removeDragData(e)
    },
    readDirectoryFiles (directoryEntry) {
      let dirReader = directoryEntry.createReader()
      dirReader.readEntries(entries => {
        entries.forEach(entr => {
          if (entr.isFile) {
            entr.file(file => {
              this.loadFile(file)
            })
          } else if (entr.isDirectory) {
            this.readDirectoryFiles(entr)
          }
        })
      })
    },
    removeDragData (e) {
      if (e.dataTransfer.items) {
        // Use DataTransferItemList interface to remove the drag data
        e.dataTransfer.items.clear()
      } else {
        // Use DataTransfer interface to remove the drag data
        e.dataTransfer.clearData()
      }
    },
    onFileLoad (e) {
      let len = e.target.files.length
      for (let i = 0; i < len; i++) {
        this.loadFile(e.target.files[i])
      }
    },

    loadFile (file) {
      this.uploadQueue.push(file)
    },

    // run file validate && auto upload
    run (file) {
      let tfile = new TFile(file)

      this.validate(tfile)
        .then(tfile => {
          this.fileList.push(tfile)
          this.autoUpload ? this.uploadFile(tfile) : (tfile.prepearUpload = true)
        })
        .catch(err => {
          console.warn(err)
        })
    },

    validate (file) {
      return new Promise((resolve, reject) => {
        //  component validator
        if (this.limit === this.fileList.length) {
          this.onExceed && this.onExceed(file)
          reject(new Error('list size exceed !'))
        }

        //  user validator
        !this.beforeUpload && resolve(file)

        if (this.beforeUpload([...this.fileList], file)) {
          resolve(file)
        }
      })
    },

    uploadFile (tfile) {
      this.uploader.upload(tfile, this.handleProgressChange)
        .then((resp, xhr) => {
          this.onUploadSuccess && this.onUploadSuccess(resp, tfile, xhr)
        })
        .catch((err, xhr) => {
          this.onUploadError && this.onUploadError(err, tfile, xhr)
        })
    },

    removeFile (tfile) {
      if ((this.beforeRemove && this.beforeRemove([...this.fileList], tfile)) || !this.beforeRemove) {
        this.fileList.splice(this.fileList.indexOf(tfile), 1)
        this.onRemove([...this.fileList], tfile)
      }
    },

    handleProgressChange (e, file, uploadCircle) {
      switch (uploadCircle) {
        case 'start':
          file.prepearUpload = false
          file.uploadSuccess = false
          file.uploadError = false
          file.loading = true
          file.uploadPercentage = 0
          break
        case 'success':
          file.uploadSuccess = true
          setTimeout(() => {
            file.loading = false
          }, 1000)
          break
        case 'error':
          file.uploadError = true
          setTimeout(() => {
            file.loading = false
          }, 500)
          break
        default:
          file.percent = e.loaded / e.total * 100
      }
    }
  },
  watch: {
    uploadQueue (q) {
      if (q.length > 0) {
        setTimeout(() => {
          this.run(q.pop())
        }, 50)
      }
    },
    fileList (list) {
      this.onFileListChange && this.onFileListChange(list)
    }
  },

  computed: {
    header () {
      return Object.assign({}, {
        'Content-Type': 'multipart/form-data; charset=utf-8;'
      }, this.headers)
    }
  },

  beforeDestroy () {
    this.$refs.trigger && this.removeDragTrigger()
  }
}
</script>
