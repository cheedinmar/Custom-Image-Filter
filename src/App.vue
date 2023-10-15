<template>
  <div class="flex flex-col space-y-4 p-5">
    <input type="file" @change="handleImageUpload" accept="image/*">

    <!-- Radio buttons for selecting filters -->
    <div class="flex space-x-4">
      <input type="radio" id="grayscale" value="grayscale" v-model="selectedFilter">
      <label for="grayscale">Grayscale</label>
      <input type="radio" id="sepia" value="sepia" v-model="selectedFilter">
      <label for="sepia">Sepia</label>
      <input type="radio" id="invert" value="invert" v-model="selectedFilter">
      <label for="invert">Invert Colors</label>
      <input type="radio" id="brightness" value="brightness" v-model="selectedFilter">
      <label for="brightness">Brightness</label>
      <input type="radio" id="contrast" value="contrast" v-model="selectedFilter">
      <label for="contrast">Contrast</label>
      <input type="radio" id="blur" value="blur" v-model="selectedFilter">
      <label for="blur">Blur</label>
      <input type="radio" id="vignette" value="vignette" v-model="selectedFilter">
      <label for="vignette">Vignette</label>

      <!-- Add more filters as needed -->
    </div>

    <div class="flex space-x-16">
      <img v-if="resultImageUrl" :src="resultImageUrl" alt="Processed Image" class="w-[500px] h-[500px]">
      <div class="w-48 h-48" :class="processed ? 'block' : 'hidden'">
        <canvas ref="imageCanvas" :width="newWidth" :height="newHeight"></canvas>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      imageInput: null,
      imageCanvas: null,
      resultImageUrl: null,
      processed: null,
      ctx: null,
      imageData: null,
      originalImageData: null,
      newWidth: 500,
      newHeight: 500,
      selectedFilter: 'grayscale',
    };
  },
  methods: {
    handleImageUpload(e) {
      this.processed = false;
      this.imageInput = e.target;
      const file = this.imageInput.files[0];
     
      if (file) {
        const reader = new FileReader();
        reader.onload = (event) => {
          const image = new Image();
          image.src = event.target.result;
          image.onload = () => {
            this.imageCanvas = this.$refs.imageCanvas;
            this.ctx = this.imageCanvas.getContext('2d');
            this.imageCanvas.width = this.newWidth;
            this.imageCanvas.height = this.newHeight;
            this.ctx.drawImage(image, 0, 0, this.newWidth, this.newHeight);
            
            this.originalImageData = this.ctx.getImageData(0, 0, this.newWidth, this.newHeight);
            console.log(this.originalImageData, 'originalImageData')
            
            this.imageData = new ImageData(new Uint8ClampedArray(this.originalImageData.data), this.newWidth, this.newHeight);
            
            this.resultImageUrl = event.target.result;

            this.applyFilter(this.selectedFilter);
          };
        };

        reader.readAsDataURL(file);
      }
    },
   applyFilter(filter) {
      if (!this.imageData) {
        alert('Please upload an image first.');
        return;
      }

      const buffer = this.imageData.data.buffer;
      console.log(buffer)
      const dataView = new Uint8Array(buffer);
      console.log(dataView)

      switch (filter) {
        case 'grayscale':
          for (let i = 0; i < dataView.length; i += 4) {
            const avg = (dataView[i] + dataView[i + 1] + dataView[i + 2]) / 3;
            dataView[i] = dataView[i + 1] = dataView[i + 2] = avg;
          }
          break;
        case 'sepia':
          for (let i = 0; i < dataView.length; i += 4) {
            const r = dataView[i];
            const g = dataView[i + 1];
            const b = dataView[i + 2];
            dataView[i] = Math.min(255, 0.393 * r + 0.769 * g + 0.189 * b);
            dataView[i + 1] = Math.min(255, 0.349 * r + 0.686 * g + 0.168 * b);
            dataView[i + 2] = Math.min(255, 0.272 * r + 0.534 * g + 0.131 * b);
          }
          break;
        case 'invert':
          for (let i = 0; i < dataView.length; i += 4) {
            dataView[i] = 255 - dataView[i];
            dataView[i + 1] = 255 - dataView[i + 1];
            dataView[i + 2] = 255 - dataView[i + 2];
          }
          break;
        case 'brightness':
          const brightnessValue = 50;
          for (let i = 0; i < dataView.length; i += 4) {
            dataView[i] = Math.min(255, dataView[i] + brightnessValue);
            dataView[i + 1] = Math.min(255, dataView[i + 1] + brightnessValue);
            dataView[i + 2] = Math.min(255, dataView[i + 2] + brightnessValue);
          }
          break;
        case 'contrast':
          const contrastValue = 1.5;
          for (let i = 0; i < dataView.length; i += 4) {
            dataView[i] = Math.min(255, (dataView[i] - 128) * contrastValue + 128);
            dataView[i + 1] = Math.min(255, (dataView[i + 1] - 128) * contrastValue + 128);
            dataView[i + 2] = Math.min(255, (dataView[i + 2] - 128) * contrastValue + 128);
          }
          break;
        case 'blur':
          const blurRadius = 5;
          const imageDataCopy = new Uint8Array(buffer);
          for (let y = 0; y < this.newHeight; y++) {
            for (let x = 0; x < this.newWidth; x++) {
              let r = 0, g = 0, b = 0;
              for (let j = -blurRadius; j <= blurRadius; j++) {
                for (let i = -blurRadius; i <= blurRadius; i++) {
                  const pixelIndex = (y + j) * this.newWidth * 4 + (x + i) * 4;
                  if (pixelIndex >= 0 && pixelIndex < dataView.length) {
                    r += imageDataCopy[pixelIndex];
                    g += imageDataCopy[pixelIndex + 1];
                    b += imageDataCopy[pixelIndex + 2];
                  }
                }
              }
              const pixelIndex = y * this.newWidth * 4 + x * 4;
              dataView[pixelIndex] = r / ((2 * blurRadius + 1) ** 2);
              dataView[pixelIndex + 1] = g / ((2 * blurRadius + 1) ** 2);
              dataView[pixelIndex + 2] = b / ((2 * blurRadius + 1) ** 2);
            }
          }
          break;
        case 'vignette':
          const centerX = this.newWidth / 2;
          const centerY = this.newHeight / 2;
          const maxDistance = Math.sqrt(centerX ** 2 + centerY ** 2);
          for (let y = 0; y < this.newHeight; y++) {
            for (let x = 0; x < this.newWidth; x++) {
              const dx = centerX - x;
              const dy = centerY - y;
              const distance = Math.sqrt(dx ** 2 + dy ** 2);
              const strength = 1 - (distance / maxDistance);
              const pixelIndex = y * this.newWidth * 4 + x * 4;
              dataView[pixelIndex] *= strength;
              dataView[pixelIndex + 1] *= strength;
              dataView[pixelIndex + 2] *= strength;
            }
          }
          break;
        case 'invertColors':
          for (let i = 0; i < dataView.length; i += 4) {
            dataView[i] = 255 - dataView[i];
            dataView[i + 1] = 255 - dataView[i + 1];
            dataView[i + 2] = 255 - dataView[i + 2];
          }
          break;
        default:
          alert('Invalid filter selected.');
      }

      this.ctx.putImageData(this.imageData, 0, 0);
      this.processed = true;
    }

  },
  watch: {
    selectedFilter(newFilter) {
      this.imageData = new ImageData(new Uint8ClampedArray(this.originalImageData.data), this.newWidth, this.newHeight);
      this.applyFilter(newFilter);
    },
  },
};
</script>
