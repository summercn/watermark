<template>
  <div id="photo-mark-wrap">
    <h2 class="page-title">给照片加上水印</h2>
    <div class="action">
      <div class="item-wrap input">
        <span class="item-label">水印文字：</span>
        <el-input v-model="text" placeholder="请输入水印文字" @change="updateWaterMark" maxlength="15"></el-input>
        <el-button type="primary" @click="generateFile" :disabled="!imgUrl">下载图片</el-button>
      </div>
      <div class="optional-attrs">
        <div class="item-wrap">
          <span class="item-label">字体大小：</span>
          <el-select v-model="fontSize" @change="updateWaterMark" placeholder="请选择透明度">
            <el-option v-for="item in fontSizeOptions" :key="item.value" :label="item.label + 'px'" :value="item.value">
            </el-option>
          </el-select>
        </div>
        <div class="item-wrap">
          <span class="item-label">旋转角度：</span>
          <el-select v-model="degree" @change="updateWaterMark" placeholder="请选择透明度">
            <el-option v-for="item in degreeOptions" :key="item.value" :label="item.label + '度'" :value="item.value">
            </el-option>
          </el-select>
        </div>
        <div class="item-wrap slider">
          <span class="item-label">不透明度：</span>
          <el-slider @change="updateWaterMark" v-model="opacity" :step="10"></el-slider>
          <span class="item-extra">{{opacity + '%'}}</span>
        </div>
      </div>
      <input type="file" ref="uploadBtn" class="hide" @change="upload" accept=".jpg, .jpeg, .png">
    </div>
    <div class="result">
      <div class="image-wrap" @click="chooseFile" v-bind:style="{width: imgUrl ? 'auto' : '400px', height: imgUrl ? 'auto' : '400px'}">
        <canvas v-show="imgUrl" class="water-mark" ref="waterMark" width="400" height="400"></canvas>
        <i class="el-icon-plus" v-if="!imgUrl"></i>
        <i class="el-icon-close" v-if="imgUrl" @click.stop="removeFile"></i>
        <img v-if="imgUrl" :src="imgUrl" ref="image" alt="图片" />
      </div>
    </div>
    <canvas class="hide" ref="sampleMark" width="350" height="150"></canvas>
    <canvas class="hide" ref="downloadCanvas" width="400" height="400"></canvas>
  </div>
</template>

<script>
  export default {
    name: 'photo-mark',
    methods: {
      chooseFile() {
        if (!this.imgUrl)
          this.$refs.uploadBtn.click()
      },
      removeFile() {
        this.imgUrl = ''
      },
      generateFile() {
        this._drawMarkedImage()
        const code = this.$refs.downloadCanvas.toDataURL('image/png')
        const blob = this._base64toBlob(code)
        window.open(URL.createObjectURL(blob))
      },
      _base64toBlob(code) {
        const parts = code.split(';base64,')
        const contentType = parts[0].split(':')[1]
        const raw = window.atob(parts[1])
        const rawLength = raw.length

        let uInt8Array = new Uint8Array(rawLength)

        for (var i = 0; i < rawLength; ++i) {
          uInt8Array[i] = raw.charCodeAt(i)
        }

        return new Blob([uInt8Array], {
          type: contentType
        })
      },
      _drawMarkedImage() {
        const {
          dom,
          ctx
        } = this._getCanvas(this.$refs.downloadCanvas)

        dom.width = this.$refs.waterMark.width
        dom.height = this.$refs.waterMark.height

        ctx.drawImage(this.$refs.image, 0, 0)
        ctx.drawImage(this.$refs.waterMark, 0, 0)
      },
      _fileReader(file, callback) {
        let reader = new FileReader()
        reader.onload = function (e) {
          callback(e.target.result)
        }
        reader.readAsDataURL(file)
      },
      _getCanvas($dom) {
        const dom = $dom || this.$refs.waterMark
        const ctx = dom.getContext('2d')
        return {
          dom,
          ctx
        }
      },
      createMarkPattern() {
        const {
          dom,
          ctx
        } = this._getCanvas(this.$refs.sampleMark)
        const degree = this.degree * Math.PI / 180
        ctx.save()
        ctx.clearRect(0, 0, dom.width, dom.height)
        ctx.font = `${this.fontSize || 20}px 黑体`
        ctx.rotate(`-${degree}`)
        ctx.fillStyle = `rgba(100, 100, 100, ${this.opacity / 100})`

        // adjust draw position to show more words
        ctx.fillText(this.text, 0, 30 + this.degree * 4)
        ctx.restore()
      },
      fillMarkPatterns() {
        const {
          dom,
          ctx
        } = this._getCanvas(this.$refs.waterMark)

        const $image = this.$refs.image

        if (!this.imgUrl || this.loadImageRetryTime >= 100) {
          this.loadImageRetryTime = 0
          return
        }

        // width: 0 => image not load right now
        if (!$image || ($image.width === 0 && this.loadImageRetryTime < 10)) {
          this.loadImageRetryTime++
            return setTimeout(this.fillMarkPatterns, 100)
        }
        this.loadImageRetryTime = 0

        // adjust canvas size and position
        dom.width = $image.width
        dom.height = $image.height
        dom.style.left = $image.offsetLeft + 'px'
        dom.style.top = $image.offsetTop + 'px'

        // canvas fill watermark
        ctx.clearRect(0, 0, dom.width, dom.height)
        ctx.fillStyle = ctx.createPattern(this.$refs.sampleMark, 'repeat')
        ctx.fillRect(0, 0, dom.width, dom.height)
      },
      updateWaterMark() {
        this.createMarkPattern()
        this.fillMarkPatterns()
      },
      upload() {
        const dom = this.$refs.uploadBtn
        const file = dom.files[0]

        if (!dom.value) {
          this.$message({
            message: '未选择文件',
            type: 'warning'
          })
          return
        }
        if (!this.ALLOW_UPLOAD_TYPE.includes(file.type)) {
          this.$message({
            message: '仅支持 jpg, jpeg, png, gif 图片格式',
            type: 'warning'
          })
          return
        }

        this._fileReader(file, fileUrl => {
          this.imgUrl = fileUrl
          this.$nextTick(() => {
            this.updateWaterMark()
          })

        })
      }
    },
    data() {
      return {
        imgUrl: '',
        text: '',
        ALLOW_UPLOAD_TYPE: ['image/png', 'image/jpeg', 'image/jpg', 'image/gif'],
        loadImageRetryTime: 0,
        degree: 20,
        degreeOptions: [{
          value: 5,
          label: 5
        }, {
          value: 10,
          label: 10
        }, {
          value: 15,
          label: 15
        }, {
          value: 20,
          label: 20
        }, {
          value: 25,
          label: 25
        }, {
          value: 30,
          label: 30
        }],
        fontSize: 18,
        fontSizeOptions: [{
          value: 10,
          label: 10
        }, {
          value: 12,
          label: 12
        }, {
          value: 14,
          label: 14
        }, {
          value: 18,
          label: 18
        }, {
          value: 24,
          label: 24
        }, {
          value: 36,
          label: 36
        }],
        opacity: 40,
      }
    },
  }
</script>

<style scoped lang="less">
  .hide {
    display: none;
  }

  #photo-mark-wrap {
    padding-left: 20px;
    text-align: left;
    margin: 0;
  }

  .action {
    margin-bottom: 20px;
  }

  .result {
    position: relative;
  }

  .water-mark {
    position: absolute;
    top: 0;
    left: 0;
  }

  .sample-water-mark {
    display: none;
  }

  .action {
    width: 1000px;
    text-align: left;

    .item-label,
    .item-extra {
      display: block;
      line-height: 44px;
    }
    .item-wrap {
      margin-right: 30px;
      display: flex;

      .el-input,
      .el-slider {
        width: 100px;
        display: block;
        margin-right: 20px;
      }

      &.input {
        .el-input {
          width: 300px;
        }
      }
    }
  }

  .optional-attrs {
    display: flex;
    margin-top: 20px;
  }

  .image-wrap {
    height: 400px;
    max-width: 900px;
    border: 1px solid #d9d9d9;
    display: flex;
    position: relative;

    [class*=" el-icon-"],
    [class^=el-icon-] {
      font-size: 30px;
      cursor: pointer;
    }

    .el-icon-close {
      position: absolute;
      top: 15px;
      right: 20px;
    }

    .el-icon-plus {
      flex: 1;
      margin: auto;
      text-align: center;
    }

    img {
      max-width: 100%;
      margin: auto;
    }
  }
</style>