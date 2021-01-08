<template>
  <div class="upload-wrapper">
    <h2>大文件分片上传</h2>
    <div class="content-wrapper">
      <!-- 文件选择 -->
      <div class="file-select" @click="choiceFile" @drop.prevent="dragFile" @dragover.prevent>
        <div class="file-select-tips">点击选择文件或者拖拽文件到此</div>
        <div class="file-select-mid">
          <img class="file-select-svg" :src="selectSvg" alt />
          <span>或者</span>
          <img class="file-select-svg" :src="dragSvg" alt />
        </div>
        <div class="file-select-tips">（支持2M到2G的大文件上传）</div>
        <input ref="fileElem" type="file" class="file-input" @change="addFile" />
      </div>
      <!-- 文件信息展示栏 -->
      <transition name="fade">
        <div v-if="showFileInfo" class="file-info">
          <!-- 文件基本信息 -->
          <div class="file-basic-info">
            <div class="left-info">
              <img :src="fileAbbrSvg" class="file-abbr" alt />
              <span class="file-name">{{fileName | ellipsis}}</span>
            </div>
            <div class="right-info">大小 {{fileSize}} {{fileUnit}}</div>
          </div>
          <!-- 上传进度 -->
          <div v-if="showUploadProgress" class="file-upload-progress">
            <span v-if="chunkTotal == 0">0%</span>
            <span v-else>{{progress}}%</span>
            <div class="progress-wrapper">
              <div class="progress-par" :style="`width:${progress}%`">
                <div class="progress-bar"></div>
              </div>
            </div>
            <span>{{chunkDoneTotal}}/{{chunkTotal}}</span>
          </div>
          <div class="file-close">
            <img class="file-close-svg" :src="closeSvg" @click="cancelFile" alt />
          </div>
          <div class="file-done" v-if="chunkDoneTotal === chunkTotal">
            <img class="file-done-svg" :src="doneSvg" alt />
          </div>
        </div>
      </transition>
      <!-- 底部按钮 -->
      <div class="bottom-button">
        <div class="chunk-size-select">
          <span>{{chunkSize | decodeSize}}</span>
        </div>
        <div
          :class="allowUpload ? 'start-upload' : 'start-upload start-upload-noallow'"
          @click="startUpload"
        >开始上传</div>
      </div>
      <!-- 操作提示信息 -->
      <div v-if="tips" :class="bottomTipsClassName">{{tips}}</div>
    </div>
  </div>
</template>

<script>
import fileSvg from '../assets/file.svg'
import imageSvg from '../assets/image.svg'
import zipSvg from '../assets/zip.svg'
import videoSvg from '../assets/video.svg'
import dragSvg from '../assets/drag.svg'
import selectSvg from '../assets/select.svg'
import closeSvg from '../assets/close.svg'
import doneSvg from '../assets/right.svg'
import SparkMD5 from 'spark-md5'
import axios from 'axios'

const baseUrl = ''
axios.defaults.baseURL = baseUrl
const blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
const zipReg = new RegExp(/^.*(?<=(zip|rar|tar))$/)
const imageReg = new RegExp(/^.*(?<=(jpg|jpeg|png))$/)
const videoReg = new RegExp(/^.*(?<=(mp4|avi|rmvb|flv))$/)
const KB = 1024
const MB = 1024 * KB
const GB = 1024 * MB

export default {
  name: 'FileUpload',
  filters: {
    ellipsis (value) {
      if (value.length >= 15) {
        return `${value.substr(0, 3)}...${value.substr(value.length - 7)}`
      } else {
        return value
      }
    },
    decodeSize (value) {
      return `${value / MB} MB`
    }
  },
  computed: {
    progress: function () {
      if (this.chunkTotal === 0) {
        return 0
      } else {
        return parseFloat(
          (this.chunkDoneTotal / this.chunkTotal) * 100
        ).toFixed(2)
      }
    }
  },
  data () {
    return {
      showFileInfo: false,
      fileAbbrSvg: fileSvg,
      selectSvg: selectSvg,
      dragSvg: dragSvg,
      closeSvg: closeSvg,
      doneSvg: doneSvg,
      showChunkSelect: false,
      allowUpload: false,
      chunkSize: 2 * MB, // 分片大小
      chunkTotal: 0, // 分片总数
      chunkDoneTotal: 0, // 上传成功分片数量
      file: null,
      fileHash: '', // 文件哈希
      fileName: '', // 文件名
      fileSize: '', // 文件大小
      fileUnit: '', // 文件大小单位
      fileType: 'file', // file zip image video
      tips: '',
      bottomTipsClassName: 'bottom-tips-success', // success warning error
      showUploadProgress: true
    }
  },
  methods: {
    // 取消文件选择，并重置input框
    cancelFile () {
      this.showFileInfo = false
      this.file = null
      this.$refs.fileElem.value = null
    },
    // 拖拽文件
    dragFile (e) {
      const droppedFiles = e.dataTransfer.files
      if (!droppedFiles) {
        return false
      }
      this.file = droppedFiles[0]
      this.getFile()
    },
    // 点击div，触发选择文件事件
    choiceFile () {
      this.$refs.fileElem.dispatchEvent(new MouseEvent('click'))
    },
    // 手动选择文件
    addFile () {
      if (this.$refs.fileElem.files && this.$refs.fileElem.files.length) {
        this.file = this.$refs.fileElem.files[0]
        this.getFile()
      }
    },
    // 根据文件大小自动匹配合适的分片大小，并计算哈希
    async getFile () {
      if (!this.file) {
        this.updateTips('error', '获取文件失败，请重试')
        return
      }
      this.calFileInfo(this.file.name, this.file.size)
      await this.calFileHash(this.file)
    },
    // 计算文件哈希
    calFileHash (file) {
      const fileSize = file.size
      // 文件大于500M时，分片大小为10M；
      // 文件大于100M时，分片大小为5M；
      if (fileSize > 500 * MB) {
        this.chunkSize = 10 * MB
      } else if (fileSize > 100 * MB) {
        this.chunkSize = 5 * MB
      }

      return new Promise((resolve, reject) => {
        const chunks = Math.ceil(file.size / this.chunkSize)
        this.chunkTotal = chunks // 记录文件分片总数

        const spark = new SparkMD5.ArrayBuffer()
        const fileReader = new FileReader()
        const that = this
        let currentChunk = 0
        that.updateTips('success', '正在读取文件计算哈希中，请耐心等待')

        fileReader.onload = function (e) {
          // 追加数组缓冲区
          spark.append(e.target.result)
          currentChunk++

          if (currentChunk < chunks) {
            loadNext()
          } else {
            const hash = spark.end()
            that.fileHash = hash // 记录文件Hash值
            that.allowUpload = true

            that.updateTips('success', '加载文件成功，文件哈希为' + hash)
            resolve(hash)
          }
        }

        fileReader.onerror = function () {
          that.updateTips('error', '读取切分文件失败，请重试')
          reject(new Error('读取切分文件失败，请重试'))
        }
        // 读取指定的 File 内容
        function loadNext () {
          const start = currentChunk * that.chunkSize
          const end =
                start + that.chunkSize >= file.size
                  ? file.size
                  : start + that.chunkSize

          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end))
        }
        // 初始化调用读取文件分片方法
        loadNext()
      }).catch((err) => {
        this.updateTips('error', err)
      })
    },
    // 点击开始上传
    async startUpload () {
      const that = this
      that.chunkDoneTotal = 0
      // 1.获取文件
      if (!that.file) {
        that.updateTips('error', '获取文件失败，请重试')
        return
      }
      if (!this.allowUpload) {
        that.updateTips('warning', '正在读取文件计算哈希中，请耐心等待')
        return
      }
      // 2.跟后台校验当前文件是否已经上传过 或者 是否需要切换断点续传
      // 返回数据中 res.data 2(文件已上传过) 1(断点续传) 0(从未上传)
      const res = await that.postFileHash()
      if (res && res.type === 2) {
        that.updateTips(
          'warning',
          '检查成功，文件在服务器上已存在，不需要重复上传'
        )
      } else {
        that.updateTips('warning', '分片正在加紧上传中，请勿关闭网页窗口')
        // 3.初始化对应请求队列并执行
        await that.buildFormDataToSend(
          that.chunkTotal,
          that.file,
          that.fileHash,
          res
        )
      }
    },
    // 发送初始文件校验
    async postFileHash () {
      const that = this
      return new Promise((resolve, reject) => {
        axios
          .post('/uploadfile/hash_check', {
            name: that.file.name,
            hash: that.fileHash,
            chunkSize: that.chunkSize,
            total: that.chunkTotal
          })
          .then((res) => {
            if (res.data.success) {
              resolve(res.data.data)
            }
          })
          .catch((err) => {
            reject(err)
            that.updateTips('error', err)
          })
      })
    },
    // 构建HTTP FormData数据并请求，最后请求合并分片
    async buildFormDataToSend (chunkCount, file, hash, res) {
      const that = this
      const chunkReqArr = []
      for (let i = 0; i < chunkCount; i++) {
        if (
          res.type === 0 ||
          (res.type === 1 &&
            res.index &&
            res.index.length > 0 &&
            !res.index.includes(i.toString()))
        ) {
          // 构建需要上传的分片数据
          const start = i * this.chunkSize
          const end = Math.min(file.size, start + this.chunkSize)
          const form = new FormData()
          form.append('file', blobSlice.call(file, start, end))
          form.append('name', file.name)
          form.append('total', chunkCount)
          form.append('chunkSize', this.chunkSize)
          form.append('index', i)
          form.append('size', file.size)
          form.append('hash', hash)
          const chunkReqItem = new Promise((resolve, reject) => {
            axios.post('/uploadfile/chunks_upload', form, {
              onUploadProgress: (e) => {
                // e为ProgressEvent，当loaded===total表明该分片上传完成
                if (e.loaded === e.total) {
                  that.chunkDoneTotal += 1
                }
              }
            }).then(response => {
              const { success } = response.data
              if (success) {
                resolve()
              } else {
                reject(new Error('上传文件分片失败'))
              }
            }).catch(error => {
              reject(new Error(error))
            })
          })
          chunkReqArr.push(chunkReqItem)
        }
      }
      if (chunkReqArr.length <= 6) {
        // 其中一个Promise捕捉到错误时，就会走到catch中，而不会走到then中
        Promise.all(chunkReqArr)
          .then(() => {
            that.updateTips('success', '分片上传完成，正在请求合并分片')
            // 合并chunks
            const data = {
              chunkSize: that.chunkSize,
              name: file.name,
              total: that.chunkTotal,
              hash: that.fileHash
            }
            that.postChunkMerge(data)
          })
          .catch(() => {
            this.updateTips('error', `上传任务中断`)
          })
      } else {
        let hasError = false
        for (let item of chunkReqArr) {
          try {
            await item
          } catch (error) {
            hasError = true
            this.updateTips('error', `由于${error}导致上传任务中断`)
          }
        }
        if (!hasError) {
          that.updateTips('success', '分片上传完成，正在请求合并分片')
          // 合并chunks
          const data = {
            chunkSize: that.chunkSize,
            name: file.name,
            total: that.chunkTotal,
            hash: that.fileHash
          }
          that.postChunkMerge(data)
        } else {
          that.updateTips('error', '分片上传过程中出现错误，请重新上传文件')
        }
      }
    },
    // 发送合并分支请求
    async postChunkMerge (data) {
      const that = this
      axios
        .post('/uploadfile/chunks_merge', data)
        .then((res) => {
          if (res.data.success) {
            that.chunkDoneTotal = that.chunkTotal
            that.updateTips('success', '文件分片上传完结并合并成功')
            // 重置文件信息展示栏
            setTimeout(() => {
              this.cancelFile()
            }, 1000)
          } else {
            that.updateTips('error', res.data.msg)
          }
        })
        .catch((err) => {
          that.updateTips('error', err)
        })
    },
    // 显示文件基本信息
    calFileInfo (fileName, fileSize) {
      this.showFileInfo = true
      this.fileName = fileName
      if (zipReg.test(fileName)) {
        this.fileAbbrSvg = zipSvg
      } else if (imageReg.test(fileName)) {
        this.fileAbbrSvg = imageSvg
      } else if (videoReg.test(fileName)) {
        this.fileAbbrSvg = videoSvg
      }
      if (fileSize >= GB) {
        this.fileSize = parseFloat(fileSize / GB).toFixed(2)
        this.fileUnit = 'GB'
      } else if (fileSize >= MB) {
        this.fileSize = parseFloat(fileSize / MB).toFixed(2)
        this.fileUnit = 'MB'
      } else if (fileSize >= KB) {
        this.fileSize = parseFloat(fileSize / KB).toFixed(2)
        this.fileUnit = 'KB'
      }
    },
    // 更新提示信息
    updateTips (type, msg) {
      this.bottomTipsClassName = `bottom-tips-${type}`
      this.tips = msg
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.upload-wrapper {
  width: 100%;
  height: 100%;
  overflow: hidden;
  background-color: rgb(249, 249, 249);
}
.content-wrapper {
  width: 538px;
  margin: 20px auto;
}
.protocol-select {
  padding: 30px 0;
  display: flex;
  justify-content: space-between;
}
.protocol-item {
  flex: 1;
  font-weight: 600;
  height: 40px;
  line-height: 40px;
  border-radius: 5px;
  border: 2px solid transparent;
  background: #fff;
  box-shadow: 0 0 15px rgb(0, 0, 0, 0.05);
}
.protocol-item-http {
  border: 2px solid rgb(253, 126, 23);
  color: rgb(253, 126, 23);
}
.protocol-item-ws {
  border: 2px solid rgb(155, 82, 244);
  color: rgb(155, 82, 244);
}
.protocol-item:first-of-type {
  margin-right: 20px;
}
.protocol-item:first-of-type:hover {
  cursor: pointer;
  border: 2px solid rgb(253, 126, 23);
  color: rgb(253, 126, 23);
  box-shadow: 0 0 15px rgb(0, 0, 0, 0.1);
}
.protocol-item:last-of-type:hover {
  cursor: pointer;
  border: 2px solid rgb(155, 82, 244);
  color: rgb(155, 82, 244);
  box-shadow: 0 0 15px rgb(0, 0, 0, 0.1);
}
.file-select {
  height: 180px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  border-radius: 5px;
  border: 2px dashed #909090;
}
.file-select-tips {
  color: #909090;
  font-size: 14px;
  padding: 16px 0;
}
.file-select-tips:last-of-type {
  font-size: 12px;
}
.file-select-mid {
  width: 200px;
  display: flex;
  justify-content: space-around;
  align-items: center;
}
.file-select-svg {
  width: 60px;
  height: 60px;
}
.file-input {
  display: none;
}
.bottom-button {
  margin-top: 30px;
  display: flex;
  justify-content: space-between;
}
.chunk-size-select {
  position: relative;
  padding: 0 10px;
  height: 36px;
  line-height: 36px;
  border: 2px dashed rgb(81, 111, 250);
  color: rgb(81, 111, 250);
  background-color: #fff;
  box-shadow: 0 0 15px rgb(0, 0, 0, 0.05);
  border-radius: 5px;
  margin-right: 20px;
  cursor: pointer;
}
.chunk-size-radio {
  position: absolute;
  left: 0;
  top: 50px;
  height: 40px;
  line-height: 40px;
  border-radius: 5px;
  padding: 0 10px;
  background-color: #fff;
  box-shadow: 0 0 15px rgb(0, 0, 0, 0.12);
  white-space: nowrap;
}
.chunk-size-radio-item {
  padding: 0 10px 0 5px;
  color: #333;
}
.chunk-size-radio-item:last-of-type {
  padding: 0 0 0 5px;
}
.start-upload {
  flex: 1;
  height: 40px;
  line-height: 40px;
  color: #fff;
  background-color: rgb(81, 111, 250);
  border-radius: 5px;
  cursor: pointer;
}
.start-upload-noallow {
  cursor: no-drop;
}
.file-info {
  position: relative;
  height: 80px;
  color: #333;
  font-size: 14px;
  box-shadow: 0 0 15px rgb(0, 0, 0, 0.05);
  border-radius: 5px;
  margin-top: 30px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  padding: 0 30px 0 22px;
}
.file-done {
  position: absolute;
  left: 0;
  top: 0;
  width: 22px;
  height: 22px;
  display: flex;
  border-radius: 5px 0 0 0;
  justify-content: left;
  align-items: top;
  background: linear-gradient(135deg, #67c23a 16px, rgb(249, 249, 249) 0);
}
.file-done-svg {
  width: 16px;
  height: 16px;
}
.file-close {
  position: absolute;
  right: 0;
  top: 0;
  width: 30px;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}
.file-close-svg {
  width: 18px;
  height: 18px;
}
.file-basic-info {
  flex: 1;
  line-height: 40px;
  display: flex;
  align-items: center;
}
.left-info {
  flex: 1;
  display: flex;
  align-items: center;
}
.right-info {
  text-align: right;
  padding-left: 10px;
}
.file-abbr {
  width: 20px;
  height: 24px;
  margin-right: 6px;
}
.file-name {
  flex: 1;
  text-align: left;
}
.file-upload-progress {
  flex: 1;
  line-height: 40px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.file-upload-progress span {
  flex: 1;
}
.progress-wrapper {
  width: 356px;
  height: 15px;
  border-radius: 8px;
  margin: 0 16px;
  background-color: #ccc;
}
.progress-par {
  overflow: hidden;
  height: 100%;
  border-radius: 8px;
}
.progress-bar {
  width: 356px;
  height: 100%;
  background: rgb(81, 111, 250);
  border-radius: 8px;
  background-image: repeating-linear-gradient(
    30deg,
    hsla(0, 0%, 100%, 0.1),
    hsla(0, 0%, 100%, 0.1) 15px,
    transparent 0,
    transparent 30px
  );
  -webkit-animation: progressbar 5s linear infinite;
  animation: progressbar 5s linear infinite;
}
.bottom-tips-success {
  height: 38px;
  line-height: 38px;
  margin-top: 30px;
  font-size: 14px;
  border-radius: 5px;
  color: #67c23a;
  background: #f0f9eb;
  border: 1px solid #c2e7b0;
}
.bottom-tips-error {
  height: 38px;
  line-height: 38px;
  margin-top: 30px;
  font-size: 14px;
  border-radius: 5px;
  color: #f56c6c;
  background: #fef0f0;
  border: 1px solid #fbc4c4;
}
.bottom-tips-warning {
  height: 38px;
  line-height: 38px;
  margin-top: 30px;
  font-size: 14px;
  border-radius: 5px;
  color: #e6a23c;
  background: #fdf6ec;
  border: 1px solid #f5dab1;
}
@-webkit-keyframes progressbar {
  0% {
    background-position: 0 0;
  }
  100% {
    background-position: 356px 0;
  }
}
@keyframes progressbar {
  0% {
    background-position: 0% 0;
  }
  100% {
    background-position: 356px 0;
  }
}
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
  opacity: 0;
}
</style>
