<template>
  <div class="container">
    <div
      class="crop-wrapper mr16 fl"
      :style="{ width: canvasWidth, height: canvasHeight }"
    >
      <ImageCropper
        ref="cropper"
        :img="originalImgData"
        :outputSize="option.outputQuality"
        :outputType="option.outputType"
        auto-crop
        :canMove="option.canMove"
        :autoCropWidth="option.cropWidth"
        :autoCropHeight="option.cropHeight"
        @realTime="realTime"
      />
    </div>
    <canvas id="target_canvas" style="display: none"></canvas>
    <div
      class="recover_img_cover fl"
      :style="{ width: canvasWidth, height: canvasHeight }"
    >
      <div
        id="recover_img"
        :style="{ backgroundImage: `url('${recoverImgUrl}')` }"
      ></div>
    </div>
  </div>
</template>

<script>
import ImageCropper from './image-cropper.vue';
import { dataURLtoBlob } from './utils';
// import vRadio from './v-radio.vue';
export default {
  name: 'VueImageHandler',
  components: {
    ImageCropper,
    // vRadio,
  },
  props: {
    imgFile: {
      required: true,
    },
    canvasWidth: {
      type: String,
      default: '380px',
    },
    canvasHeight: {
      type: String,
      default: '252px',
    },
    colorDiff: {
      type: Number,
      default: 20,
    },
    wipeColor: String,
    option: {
      type: Object,
      default() {
        return {
          outputQuality: 1,
          full: false,
          outputType: 'png',
          canMove: true,
          fixedBox: false,
          canMoveBox: true,
          cropWidth: 380,
          cropHeight: 252,
          centerBox: false,
          high: true,
          max: 99999,
        };
      },
    },
  },
  data() {
    return {
      model: false,
      modelSrc: '',
      crap: false,
      img: null,
      imgDataArr: [],
      resultImgDataArr: [],
      drawImgWidth: '',
      drawImgHeight: '',
      COLORS: {
        WHITE: {
          aValue: 255,
          bValue: 255,
          gValue: 255,
          rValue: 255,
        },
        BLACK: {
          aValue: 255,
          bValue: 0,
          gValue: 0,
          rValue: 0,
        },
      },
      clickColorInfo: {},
      recoverImgUrl: '',
      originalImgData: null,
    };
  },
  computed: {},
  watch: {
    imgFile: {
      handler() {
        if (this.imgFile) {
          this.init();
        } else {
          this.clear();
        }
      },
      immediate: true,
    },
    colorDiff() {
      if (this.wipeColor) {
        this.resultImgDataArr = [];
        this.transparentModeCanvas();
      }
    },
    wipeColor() {
      this.resultImgDataArr = [];
      if (!this.wipeColor) {
        this.clickColorInfo = {};
        this.recoverCanvas();
        return;
      } else {
        const colorFieldName = this.wipeColor === 'white' ? 'WHITE' : 'BLACK';
        this.clickColorInfo = this.COLORS[colorFieldName];
      }
      this.transparentModeCanvas();
    },
  },
  created() {
  },
  mounted() {
    this.img = new Image();
  },
  methods: {
    download() {
      this.$refs.cropper.getCropBlob(data => {
        if (window.navigator.msSaveBlob) {
          var blobObject = new Blob([data]);
          window.navigator.msSaveBlob(blobObject, 'demo.png');
        } else {
          this.$nextTick(() => {
            const a = document.createElement('a');
            a.setAttribute('download', 'demo.png');
            a.setAttribute('href', window.URL.createObjectURL(data));
            a.click();
          });
        }
      });
    },
    rotate() {
      this.$refs.cropper.rotateLeft();
    },
    refresh() {
      this.$refs.cropper.refresh();
    },
    handleSkip(callback) {
      callback(this.imgFile);
    },
    init() {
      this.originalImgData = this.imgFile;
      this.clickColorInfo = {};
      this.resultImgDataArr = [];
      this.$nextTick(() => {
        // this.$refs.cropper.reload();
        this.$refs.cropper.refresh();
      });
    },
    recoverCanvas() {
      if (this.imgDataArr.length === 0) {
        return;
      }
      const ctx = document.getElementById('target_canvas').getContext('2d');
      ctx.putImageData(
        new ImageData(this.imgDataArr, this.drawImgWidth, this.drawImgHeight), 0, 0
      );
      this.setCanvasImgToDownloadLink();
    },
    handleCrop() {
      this.$refs.cropper.getCropBlob(data => {
        this.setImgIntoCanvas(data);
      });
    },
    // clear
    clear() {
      this.originalImgData = null;
      this.recoverImgUrl = '';
      this.$refs.cropper.clearCrop();
    },
    // 实时预览函数
    realTime() {
      // 不能用getCropBlob, 否则预览图会闪烁. 原因不明
      this.$refs.cropper.getCropData(data => {
        this.setImgIntoCanvas(dataURLtoBlob(data));
      });
    },
    // 加载图片资源到canvas中
    setImgIntoCanvas(imgFile) {
      const reader = new FileReader();
      reader.readAsDataURL(imgFile);
      reader.onloadend = (e) => {
        const dataURL = e.target.result;
        this.img.src = dataURL;
        this.img.onload = (event) => {
          const ctx = document.getElementById('target_canvas').getContext('2d');
          //将canvas大小设置为和图片一样大
          this.drawImgWidth = this.img.width;
          this.drawImgHeight = this.img.height;
          this.setCanvasSize(this.drawImgWidth, this.drawImgHeight);
          ctx.drawImage(this.img, 0, 0, this.drawImgWidth, this.drawImgHeight);
          this.imgDataArr = ctx.getImageData(0, 0, this.drawImgWidth, this.drawImgHeight).data;
          this.resultImgDataArr = this.imgDataArr.slice(0);
          this.transparentModeCanvas();
        };
      };
    },
    // 设置容差范围内的像素值为透明
    transparentModeCanvas() {
      if (this.imgDataArr.length === 0) {
        return;
      }
      if (this.resultImgDataArr.length === 0) {
        this.resultImgDataArr = this.imgDataArr.slice(0);
      }
      const ctx = document.getElementById('target_canvas').getContext('2d');
      for (
        let pos = this.drawImgWidth * this.drawImgHeight * 4 - 4;
        pos >= 0;
        pos = pos - 4
      ) {
        if (
          this.getColorDiff(this.resultImgDataArr, pos, this.clickColorInfo) <
          this.colorDiff
        ) {
          this.resultImgDataArr[pos] = 0;
          this.resultImgDataArr[pos + 1] = 0;
          this.resultImgDataArr[pos + 2] = 0;
          this.resultImgDataArr[pos + 3] = 0;
        }
      }
      ctx.putImageData(
        new ImageData(
          this.resultImgDataArr,
          this.drawImgWidth,
          this.drawImgHeight
        ),
        0,
        0
      );
      this.setCanvasImgToDownloadLink();
    },
    // 设置canvas宽高
    setCanvasSize(width, height) {
      const targetCanvasEl = document.getElementById('target_canvas');
      targetCanvasEl.width = width;
      targetCanvasEl.height = height;
    },
    //获取图像数据指定位置颜色与指定颜色的色差
    getColorDiff(imgDataArr, pos, colorInfo) {
      const value =
        Math.pow(imgDataArr[pos] - colorInfo.rValue, 2) +
        Math.pow(imgDataArr[pos + 1] - colorInfo.gValue, 2) +
        Math.pow(imgDataArr[pos + 2] - colorInfo.bValue, 2);
      return Math.pow(value, 0.5);
    },
    /**
     * 获取图像数据中指定偏移处的颜色信息
     */
    getColorInfo(imgDataArr, offsetX, offsetY) {
      const pos = this.drawImgWidth * 4 * offsetY + offsetX * 4;
      return {
        rValue: imgDataArr[pos],
        gValue: imgDataArr[pos + 1],
        bValue: imgDataArr[pos + 2],
        aValue: imgDataArr[pos + 3],
      };
    },
    /**
     * 设置canvas图像到下载链接上
     */
    setCanvasImgToDownloadLink() {
      const imgData = document.getElementById('target_canvas').toDataURL();
      this.recoverImgUrl = imgData;
    },
    /**
     * 确认返回最终处理完的图片
     * type: dataUrl ===> 返回base64; blob ===> 返回Blob对象
     */
    getImageUrl(callback) {
      callback(this.recoverImgUrl);
    },
  },
};
</script>

<style lang="css" scoped>
.crop-img-preview-box {
  border: 1px dashed #0784ff;
}
.recover_img_cover {
  position: relative;
  box-sizing: border-box;
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  direction: ltr;
  touch-action: none;
  text-align: left;
  background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQAQMAAAAlPW0iAAAAA3NCSVQICAjb4U/gAAAABlBMVEXMzMz////TjRV2AAAACXBIWXMAAArrAAAK6wGCiw1aAAAAHHRFWHRTb2Z0d2FyZQBBZG9iZSBGaXJld29ya3MgQ1M26LyyjAAAABFJREFUCJlj+M/AgBVhF/0PAH6/D/HkDxOGAAAAAElFTkSuQmCC');
}
.recover_img_cover #recover_img {
  width: 100%;
  height: 100%;
  background-repeat: no-repeat;
  background-position: 50%;
  background-size: contain;
}
.flex {
  display: flex;
}
.mr16 {
  margin-right: 16px;
}
.center {
  display: flex;
  justify-content: center;
  align-items: center;
}
.fl {
  float: left;
}
.container {
  width: fit-content;
}
.container:after {
  content: '';
  display: block;
  width: 0;
  height: 0;
  clear: both;
  visibility: hidden;
}
</style>
