<template>
  <div class="scanner-container">
    <video ref="videoRef" autoplay muted playsinline class="scanner-video"></video>
    <canvas ref="canvasRef" class="hidden-canvas"></canvas>
    <div class="overlay"><div class="target"></div></div>

    <p v-if="codeText" class="result">📦 {{ codeText }}</p>
    <p v-else class="hint">Alignez le code Datamatrix dans le carré vert…</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import { BrowserMultiFormatReader, BarcodeFormat } from "@zxing/browser";

const videoRef = ref(null);
const canvasRef = ref(null);
const codeText = ref("");
let reader, ctx, loopId;

onMounted(async () => {
  reader = new BrowserMultiFormatReader();

  const constraints = {
    video: {
      facingMode: { ideal: "environment" },
      width: { ideal: 1280 },
      height: { ideal: 720 },
    },
  };

  try {
    const stream = await navigator.mediaDevices.getUserMedia(constraints);
    videoRef.value.srcObject = stream;

    await new Promise((r) => (videoRef.value.onloadedmetadata = r));

    const video = videoRef.value;
    const canvas = canvasRef.value;
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    ctx = canvas.getContext("2d");

    // boucle manuelle : on traite 10 images/s (assez rapide sans saturer le CPU)
    const scan = async () => {
      if (!video || video.readyState !== 4) return;
      // 1. capture frame
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      // 2. prétraitement simple (contraste + gris)
      const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const data = imgData.data;
      for (let i = 0; i < data.length; i += 4) {
        const gray = data[i] * 0.3 + data[i + 1] * 0.59 + data[i + 2] * 0.11;
        // augmenter contraste
        const factor = 1.5;
        const newVal = Math.min(255, Math.max(0, (gray - 128) * factor + 128));
        data[i] = data[i + 1] = data[i + 2] = newVal;
      }
      ctx.putImageData(imgData, 0, 0);

      // 3. décodage
      try {
        const result = await reader.decodeFromCanvas(canvas);
        if (result && result.getText() !== codeText.value) {
          codeText.value = result.getText();
        }
      } catch {
        /* pas de code trouvé, on continue */
      }

      loopId = requestAnimationFrame(scan);
    };
    scan();
  } catch (err) {
    console.error("Erreur caméra :", err);
  }
});

onBeforeUnmount(() => {
  if (reader) reader.reset();
  if (loopId) cancelAnimationFrame(loopId);
});
</script>

<style scoped>
.scanner-container {
  position: relative;
  width: 100%;
  height: 90vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.scanner-video {
  width: 100%;
  max-height: 80vh;
  border-radius: 16px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
  object-fit: cover;
}
.hidden-canvas {
  display: none;
}
.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  pointer-events: none;
}
.target {
  width: 200px;
  height: 200px;
  border: 3px solid #4caf50;
  border-radius: 12px;
  box-shadow: 0 0 20px rgba(76, 175, 80, 0.5);
}
.result {
  margin-top: 12px;
  font-size: 1.2rem;
  color: #2e7d32;
  font-weight: bold;
  text-align: center;
  word-break: break-all;
}
.hint {
  margin-top: 10px;
  font-size: 1rem;
  color: #555;
}
</style>
