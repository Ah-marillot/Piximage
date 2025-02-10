<script setup>
import { ref, onMounted, watch } from "vue";

const props = defineProps({
  sideLength: {
    type: Number,
    required: true,
  },
});

const emit = defineEmits(["update-matrix"]);

const imageFile = ref(null);
const croppedImageMatrix = ref([]);
const imagePosition = ref({ x: 0, y: 0 });
const imageScale = ref(1);
const dragging = ref(false);
const dragStart = ref({ x: 0, y: 0 });
const loading = ref(false);
const canvasPreview = ref(null);

const handleImageUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    loading.value = true;
    const reader = new FileReader();
    reader.onload = (e) => {
      imageFile.value = e.target.result;
      loading.value = false;
      imageToMatrix();
    };
    reader.readAsDataURL(file);
  }
};

const imageToMatrix = () => {
  const canvas = document.createElement("canvas");
  const ctx = canvas.getContext("2d");
  const img = new Image();
  img.src = imageFile.value;

  img.onload = () => {
    canvas.width = 300;
    canvas.height = 300;
    ctx.drawImage(
      img,
      - imagePosition.value.x,
      - imagePosition.value.y,
      img.width * 1/imageScale.value,
      img.height * 1/imageScale.value,
      imagePosition.value.x -150,
      imagePosition.value.y -150,
      imagePosition.value.x +150,
      imagePosition.value.y +150
    );

    const imageData = ctx.getImageData(0, 0, 300, 300);
    const data = imageData.data;
    const matrix = [];
    const rectWidth = Math.floor(300 / props.sideLength);
    const rectHeight = Math.floor(300 / props.sideLength);

    for (let i = 0; i < props.sideLength; i++) {
      const row = [];
      for (let j = 0; j < props.sideLength; j++) {
        let rSum = 0,
          gSum = 0,
          bSum = 0,
          count = 0;
        for (let y = i * rectHeight; y < (i + 1) * rectHeight; y++) {
          for (let x = j * rectWidth; x < (j + 1) * rectWidth; x++) {
            const index = (y * 300 + x) * 4;
            rSum += data[index];
            gSum += data[index + 1];
            bSum += data[index + 2];
            count++;
          }
        }
        const rAvg = Math.floor(rSum / count);
        const gAvg = Math.floor(gSum / count);
        const bAvg = Math.floor(bSum / count);
        row.push([rAvg, gAvg, bAvg]);
      }
      matrix.push(row);
    }

    croppedImageMatrix.value = matrix;
    emit("update-matrix", matrix);

    // Update the canvas preview
    const previewCanvas = canvasPreview.value;
    const previewCtx = previewCanvas.getContext("2d");
    previewCtx.clearRect(0, 0, previewCanvas.width, previewCanvas.height);
    previewCtx.drawImage(canvas, 0, 0);
  };
};

const createCroppedImage = () => {
  const canvas = document.createElement("canvas");
  const ctx = canvas.getContext("2d");
  const img = new Image();
  img.src = imageFile.value;

  img.onload = () => {
    canvas.width = 300;
    canvas.height = 300;
    ctx.drawImage(
      img,
      imagePosition.value.x,
      imagePosition.value.y,
      img.width * imageScale.value,
      img.height * imageScale.value,
      0,
      0,
      300,
      300
    );
    const croppedImage = canvas.toDataURL("image/png");

    // Create a link element
    const link = document.createElement("a");
    link.href = croppedImage;
    link.download = "cropped-image.png";

    // Append the link to the body
    document.body.appendChild(link);

    // Trigger a click on the link
    link.click();

    // Remove the link from the document
    document.body.removeChild(link);
  };
};

const startDrag = (event) => {
  dragging.value = true;
  dragStart.value = {
    x: event.clientX - imagePosition.value.x,
    y: event.clientY - imagePosition.value.y,
  };
};

const onDrag = (event) => {
  if (dragging.value) {
    imagePosition.value = {
      x: event.clientX - dragStart.value.x,
      y: event.clientY - dragStart.value.y,
    };
    updateImagePosition();
  }
};

const stopDrag = () => {
  dragging.value = false;
};

const updateImagePosition = () => {
  const img = document.querySelector(".image-preview img");
  const canvas = canvasPreview.value;
  if (img) {
    img.style.transform = `translate(${imagePosition.value.x}px, ${imagePosition.value.y}px) scale(${imageScale.value})`;
  }
  if (canvas) {
    canvas.style.transform = `translate(${imagePosition.value.x}px, ${imagePosition.value.y}px) scale(${imageScale.value})`;
  }
};

onMounted(() => {
  window.addEventListener("mousemove", (event) => {
    if (dragging.value) {
      onDrag(event);
    }
  });
  window.addEventListener("mouseup", stopDrag);
});

watch(imageScale, updateImagePosition);

watch(imageFile, (newValue) => {
  if (newValue) {
    imageToMatrix();
  }
});

watch(imagePosition, () => {
  imageToMatrix();
});

watch(imageScale, () => {
  imageToMatrix();
});
</script>

<template>
  <div>
    <input type="file" @change="handleImageUpload" />
    <div class="image-preview" @mousedown="startDrag">
      <div v-if="loading" class="loading-spinner"></div>
      <img
        v-if="imageFile && !loading"
        :src="imageFile"
        alt="Uploaded Image"
        :style="{
          transform: `translate(${imagePosition.x}px, ${imagePosition.y}px) scale(${imageScale.value})`,
        }"
        @mousedown.prevent
      />
      <div class="crop-area"></div>
    </div>
    Resize
    <input type="range" min="0.1" max="2" step="0.01" v-model="imageScale" />
    {{ imageScale }}
    <button @click="createCroppedImage">Create Cropped Image And DL</button>
    <div class="canvas-preview" @mousedown="startDrag">
      <canvas
        ref="canvasPreview"
        width="300"
        height="300"
        style="border: 1px solid red"
      ></canvas>
    </div>
  </div>
</template>

<style scoped>
.image-preview,
.canvas-preview {
  width: 400px;
  height: 400px;
  overflow: hidden;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 1px solid #ccc;
}

.image-preview img,
.canvas-preview canvas {
  max-width: none;
  max-height: none;
  cursor: grab;
}

.image-preview img:active,
.canvas-preview canvas:active {
  cursor: grabbing;
}

.crop-area {
  width: 300px;
  height: 300px;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border: 2px dashed #000;
  pointer-events: none;
}

.loading-spinner {
  border: 4px solid rgba(0, 0, 0, 0.1);
  border-left-color: #000;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.canvas-preview {
  margin-top: 20px;
}
</style>
