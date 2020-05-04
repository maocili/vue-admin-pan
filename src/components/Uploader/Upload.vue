<template>
  <div class="app-container">
    <div>{{ path }}</div>
    <el-upload
      class="upload"
      action=""
      drag
      multiple
      :show-file-list="false"
      :http-request="upload"
    >
      <i class="el-icon-upload" />
      <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
    </el-upload>

    <!-- 上传的文件列表 -->
    <div>
      <ul class="upload-file-list">
        <li v-for="(file, fIndex) of fileUploadQueue">
          <div class="file-flx-row">
            <div class="flx-l"><i class="el-icon-document" />{{ file.name }}</div>
            <div class="flx-r">
              <i v-if="file.progress === 100" class="el-icon-upload-success el-icon-circle-check" />
              <!--<i v-if="file.progress === 100" class="el-icon-close" @click="onRemoveFile(file.hash)"></i>-->
            </div>
          </div>

          <div v-show="file.progress < 100" class="progress-bar">
            <div class="progress-out">
              <div class="progress-in" :style="{width: file.progress + '%'}" />
            </div>
            <div class="progress-rate">{{ file.progress }}%</div>
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import sha1 from 'js-sha1'

// 每一分块大小5MB
const CHUNK_SIZE = 5 * 1024 * 1024
// 文件对象的切片方法
const blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice

export default {
  props: {
    path: {
      type: String,
      default: 'root'
    }
  },
  data() {
    return {
      fileUploadQueue: [] // 上传的文件队列
    }
  },
  methods: {
    async upload(fileObj) {
      const file = fileObj.file
      // 加载动画
      const loading = this.$loading({
        lock: true,
        text: '正在加载' + ' : ' + file.name,
        spinner: 'el-icon-loading',
        background: 'rgba(0, 0, 0, 0.7)'
      })

      const hash = await this.getFileHash(file) // 获取文件的hash值
      const axiosArr = [] // axios请求队列
      loading.close()
      const initParams = {
        fileHash: hash,
        fileSize: file.size,
        fileName: file.name,
        filePath: this.path
      }

      // 先初始化上传
      axios.post('/api/file/chunk/init', initParams).then(resp => {
        resp = resp.data

        if (+resp.code === 20000) {
          // 文件分块再上传
          if (resp.data) {
            const chunkCount = +resp.data.ChunkCount

            this.enFileQueue(hash, file.name, file.size, chunkCount)

            for (let i = 0; i < chunkCount; i++) {
              const start = i * CHUNK_SIZE
              const end = Math.min(file.size, start + CHUNK_SIZE)
              const fileChunk = blobSlice.call(file, start, end)

              const result = this.createFormData(fileChunk, i, resp.data.UploadId, hash)
              axiosArr.push(axios.post('/api/file/chunk/upload', result.form, result.config))
            }

            // 待所有片段上传完成，合并文件
            axios.all(axiosArr).then(() => {
              // 如果有uploadI发起上传完成请求
              const mergeData = {
                uploadId: resp.data.UploadId,
                fileName: file.name,
                filePath: this.path
              }

              axios.post('/api/file/chunk/finish', mergeData).catch(err => {
                this.$message({ message: err, type: 'error' })
              })
            }).catch(err => {
              this.$message({ message: err, type: 'error' })
            })
          } else {
            // 不用分块，文件已经上传过，直接显示上传完成
            this.enFileQueue(hash, file.name, file.size, 1, 100)
          }
        } else {
          this.$message({ message: resp.msg, type: 'warning' })
        }
      }).catch(err => {
        this.$message({ message: err, type: 'error' })
      })
    },

    /** 获取文件hash */
    getFileHash(file) {
      return new Promise((resolve, reject) => {
        const chunks = Math.ceil(file.size / CHUNK_SIZE) // 分块总数，向上取整
        // spark = new SparkMD5.ArrayBuffer(),
        const fileReader = new FileReader()
        const hasher = sha1.create()

        let currentChunk = 0 // 当前块数

        // 分块获取文件的hash值
        fileReader.onload = e => {
          hasher.update(e.target.result)
          ++currentChunk // 块数递增

          // 如果当前块数小于总的分块数量，load下一个分块
          if (currentChunk < chunks) {
            loadNextChunk()
          } else {
            const hexHash = hasher.hex()

            resolve(hexHash)
          }
        }

        fileReader.onerror = () => {
          this.$message({ message: '文件读取失败！', type: 'error' })
        }

        // 读取下一个文件片段
        function loadNextChunk() {
          const start = currentChunk * CHUNK_SIZE
          const end = Math.min(file.size, start + CHUNK_SIZE)

          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end))
        }

        loadNextChunk()
      }).catch(err => {
        this.$message({ message: err, type: 'error' })
      })
    },

    /** 构建表单 */
    createFormData(file, chunkIdx, uploadId, fileHash) {
      const form = new FormData()

      form.append('file', file)
      form.append('chunkindex', chunkIdx + 1)
      form.append('uploadid', uploadId)

      const currentFile = this.getFileFromQueue(fileHash)
      const config = {
        headers: {
          'Content-Type': 'multipart/form-data;boundary = ' + new Date().getTime()
        },
        onUploadProgress: resp => {
          if (resp.lengthComputable) {
            const curFile = currentFile.file
            let totalUploaded = 0

            curFile.chunkUploadedArr[chunkIdx] = resp.loaded
            curFile.chunkUploadedArr.forEach(item => {
              if (+item > 0) {
                totalUploaded += Number(item)
              }
            })
            currentFile.file.progress = parseInt((totalUploaded / curFile.size) * 100)
            // this.$set(this.fileUploadQueue, currentFile.index, curFile);
          }
        }
      }

      return { form, config }
    },

    enFileQueue(hash, name, size, chunks, progress = 0) {
      const hashObj = {
        hash: hash,
        name: name,
        size: size,
        progress: progress,
        chunkUploadedArr: new Array(chunks) // 记录每个分块的已上传文件的大小
      }

      this.fileUploadQueue.push(hashObj)
    },

    getFileFromQueue(hash) {
      const fileQueue = this.fileUploadQueue
      let targetFile = null
      let index = 0

      for (let i = 0; i < fileQueue.length; i++) {
        if (fileQueue[i].hash === hash) {
          index = i
          targetFile = fileQueue[i]
          break
        }
      }

      return { file: targetFile, index: index }
    }

    // 移除文件
    // onRemoveFile(hash){
    //   const target = this.getFileHash(hash)
    //
    //   this.fileUploadQueue.splice(target.index, 1);
    // },
  }
}
</script>

<style scoped>
  .upload-file-list{
    padding: 15px;
    list-style: none;
  }
  .upload-file-list li{
    padding: 5px;
    font-size: 14px;
  }
  .file-flx-row{
    display: flex;
    line-height: 1.8;
    cursor: pointer;
    vertical-align: middle;
  }
  .file-flx-row:hover{
    background: #f5f7fa;
  }
  .flx-l{
    flex: 1;
  }
  .file-flx-row:hover .el-icon-close{
    display: block;
  }
  .file-flx-row:hover .el-icon-upload-success{
    display: none;
  }

  .el-icon-upload-success{
    color: #67c23a;
  }
  .el-icon-close{
    position: relative;
    top: 3px;
    display: none;
    color: #606266;
  }
  .progress-bar{
    margin-top: 5px;
    position: relative;
    width: 100%;
    box-sizing: border-box;
  }
  .progress-out{
    position: relative;
    height: 2px;
    border-radius: 100px;
    background-color: #ebeef5;
    overflow: hidden;
  }
  .progress-in{
    position: absolute;
    left: 0;
    top: 0;
    width: 10%;
    height: 100%;
    background-color: #409eff;
    border-radius: 100px;
    line-height: 1;
    white-space: nowrap;
    transition: width .6s ease;
  }
  .progress-rate{
    position: absolute;
    right: 0;
    top: -25px;
    color: #606266;
    font-size: 13px;
  }
</style>

