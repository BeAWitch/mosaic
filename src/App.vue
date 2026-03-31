<template>
  <div class="mosaic-container">
    <div class="header">
      <h2>图片马赛克生成器</h2>
    </div>

    <div class="controls">
      <el-row :gutter="20">
        <el-col :span="6">
          <el-upload
            class="upload-demo"
            drag
            action=""
            :auto-upload="false"
            :show-file-list="false"
            :on-change="handleFileChange"
          >
            <el-icon class="el-icon--upload"><upload-filled /></el-icon>
            <div class="el-upload__text">
              拖拽图片到此处或 <em>点击上传</em>
            </div>
          </el-upload>
        </el-col>

        <el-col :span="18" class="sliders">
          <div class="slider-item">
            <span class="label">网格大小 (Grid Spacing)</span>
            <el-slider v-model="params.gridSpacing" :min="10" :max="100" @input="debouncedProcess" />
          </div>
          <div class="slider-item">
            <span class="label">最大边框比例 (Max Border Ratio)</span>
            <el-slider v-model="params.maxBorderRatio" :min="0" :max="0.5" :step="0.01" @input="debouncedProcess" />
          </div>
          <div class="slider-item">
            <span class="label">边框厚度系数 (Border Multiplier)</span>
            <el-slider v-model="params.borderMultiplier" :min="0.1" :max="1.0" :step="0.05" @input="debouncedProcess" />
          </div>
          <el-button type="primary" @click="downloadImage" :disabled="!isImageLoaded">下载结果图</el-button>
        </el-col>
      </el-row>
    </div>

    <div class="preview" v-if="isImageLoaded">
      <div class="canvas-wrapper" v-loading="isProcessing">
        <canvas ref="resultCanvas"></canvas>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import { UploadFilled } from '@element-plus/icons-vue'

const resultCanvas = ref(null)
const isImageLoaded = ref(false)
const isProcessing = ref(false)

const params = reactive({
  gridSpacing: 30,
  maxBorderRatio: 0.4,
  borderMultiplier: 0.5
})

let originalImage = null
let debounceTimer = null

// RGB转HSV
const rgbToHsv = (r, g, b) => {
  r /= 255; g /= 255; b /= 255;
  const max = Math.max(r, g, b), min = Math.min(r, g, b);
  let h, s, v = max;
  const d = max - min;
  s = max === 0 ? 0 : d / max;
  if (max === min) {
    h = 0; // achromatic
  } else {
    switch (max) {
      case r: h = (g - b) / d + (g < b ? 6 : 0); break;
      case g: h = (b - r) / d + 2; break;
      case b: h = (r - g) / d + 4; break;
    }
    h /= 6;
  }
  return [h * 360, s, v];
}

// HSV转RGB
const hsvToRgb = (h, s, v) => {
  let r, g, b;
  let i = Math.floor(h / 60);
  let f = h / 60 - i;
  let p = v * (1 - s);
  let q = v * (1 - f * s);
  let t = v * (1 - (1 - f) * s);

  switch (i % 6) {
    case 0: r = v, g = t, b = p; break;
    case 1: r = q, g = v, b = p; break;
    case 2: r = p, g = v, b = t; break;
    case 3: r = p, g = q, b = v; break;
    case 4: r = t, g = p, b = v; break;
    case 5: r = v, g = p, b = q; break;
  }
  return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
}

const handleFileChange = (file) => {
  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      originalImage = img;
      isImageLoaded.value = true;
      processImage();
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file.raw);
}

const debouncedProcess = () => {
  if (!isImageLoaded.value) return;
  isProcessing.value = true;
  if (debounceTimer) clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    processImage();
  }, 300); // 300ms debounce
}

const processImage = () => {
  if (!originalImage || !resultCanvas.value) {
    isProcessing.value = false;
    return;
  }

  isProcessing.value = true;
  // RequestAnimationFrame allows the UI loading state to render before blocking thread
  requestAnimationFrame(() => {
    const canvas = resultCanvas.value;
    const ctx = canvas.getContext('2d', { willReadFrequently: true });
    
    const width = originalImage.width;
    const height = originalImage.height;
    canvas.width = width;
    canvas.height = height;

    // 1. 获取原图数据 (先在离屏canvas获取或直接获取)
    const offscreen = document.createElement('canvas');
    offscreen.width = width;
    offscreen.height = height;
    const offCtx = offscreen.getContext('2d');
    offCtx.drawImage(originalImage, 0, 0, width, height);
    const imageData = offCtx.getImageData(0, 0, width, height);
    const data = imageData.data;

    // 2. 清空画布（背景设黑，相当于全部画上黑边）
    ctx.fillStyle = '#000000';
    ctx.fillRect(0, 0, width, height);

    const gs = params.gridSpacing;
    const nCols = Math.ceil(width / gs);
    const nRows = Math.ceil(height / gs);

    for (let r = 0; r < nRows; r++) {
      for (let c = 0; c < nCols; c++) {
        let startX = c * gs;
        let startY = r * gs;
        let endX = Math.min(startX + gs, width);
        let endY = Math.min(startY + gs, height);
        let regionW = endX - startX;
        let regionH = endY - startY;

        let sumSin = 0;
        let sumCos = 0;
        let sumS = 0;
        let sumV = 0;
        let pixelCount = 0;

        for (let y = startY; y < endY; y++) {
          for (let x = startX; x < endX; x++) {
            const i = (y * width + x) * 4;
            const hsv = rgbToHsv(data[i], data[i+1], data[i+2]);
            const h = hsv[0], s = hsv[1], v = hsv[2];
            
            const rad = h * Math.PI / 180;
            sumSin += Math.sin(rad);
            sumCos += Math.cos(rad);
            sumS += s;
            sumV += v;
            pixelCount++;
          }
        }

        if (pixelCount === 0) continue;

        let meanSin = sumSin / pixelCount;
        let meanCos = sumCos / pixelCount;
        let meanH = Math.atan2(meanSin, meanCos) * 180 / Math.PI;
        if (meanH < 0) meanH += 360;
        
        let meanS = sumS / pixelCount;
        let meanV = sumV / pixelCount; // 0~1

        // 边框逻辑
        let borderRatio = Math.min(params.maxBorderRatio, (1 - meanV) * params.borderMultiplier);
        let borderWidth = Math.max(1, Math.floor(borderRatio * gs));

        borderWidth = Math.min(borderWidth, Math.floor(regionH / 2), Math.floor(regionW / 2));

        // 内部颜色绘制 (v = 1 即 100%明度)
        const rgb = hsvToRgb(meanH, meanS, 1);
        
        let innerX = startX + borderWidth;
        let innerY = startY + borderWidth;
        let innerW = regionW - 2 * borderWidth;
        let innerH = regionH - 2 * borderWidth;

        if (innerW > 0 && innerH > 0) {
          ctx.fillStyle = `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`;
          ctx.fillRect(innerX, innerY, innerW, innerH);
        }
      }
    }
    
    isProcessing.value = false;
  });
}

const downloadImage = () => {
  if (!resultCanvas.value) return;
  const link = document.createElement('a');
  link.download = 'mosaic-result.png';
  link.href = resultCanvas.value.toDataURL();
  link.click();
}
</script>

<style scoped>
.mosaic-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: var(--el-font-family);
}
.header {
  text-align: center;
  margin-bottom: 20px;
}
.controls {
  margin-bottom: 20px;
  background: #f5f7fa;
  padding: 20px;
  border-radius: 8px;
}
.sliders {
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.slider-item {
  margin-bottom: 15px;
}
.slider-item .label {
  display: block;
  margin-bottom: 5px;
  font-size: 14px;
  color: #606266;
}
.preview {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
}
.canvas-wrapper {
  overflow: auto;
  max-height: 80vh;
  max-width: 100%;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  border-radius: 4px;
  background-color: #e4e7ed;
  display: flex;
  justify-content: center;
}
canvas {
  display: block;
  max-width: 100%;
  height: auto;
  object-fit: contain;
}
</style>