<template>
  <div class="scanner-container">
    <video ref="videoRef" autoplay muted playsinline class="scanner-video"></video>
    <div class="overlay"><div class="target"></div></div>

    <p v-if="codeText" class="result">📦 {{ codeText }}</p>
    <p v-else class="hint">Visez le Datamatrix dans le carré vert…</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import { BrowserMultiFormatReader, BarcodeFormat } from "@zxing/browser";

const videoRef = ref(null);
const codeText = ref("");
let reader;

onMounted(async () => {
  reader = new BrowserMultiFormatReader();

  try {
    // 🔧 Contraintes vidéo optimisées pour mobile
    const constraints = {
      video: {
        facingMode: { ideal: "environment" },
        width: { ideal: 1280 },
        height: { ideal: 720 },
        focusMode: "continuous",
      },
    };

    const stream = await navigator.mediaDevices.getUserMedia(constraints);
    videoRef.value.srcObject = stream;

    // 🔍 Spécifie qu'on ne lit que le format Datamatrix
    const hints = new Map();
    hints.set(0, [BarcodeFormat.DATA_MATRIX]);

    // 🚀 Décodage en continu
    reader.decodeFromVideoDevice(
      null,
      videoRef.value,
      (result, err) => {
        if (result) {
          const text = result.getText();
          if (text && text !== codeText.value) {
            codeText.value = text;
            console.log("Code détecté :", text);
          }
        }
      },
      { formats: [BarcodeFormat.DATA_MATRIX] }
    );
  } catch (err) {
    console.error("Erreur d’accès à la caméra :", err);
  }
});

onBeforeUnmount(() => {
  if (reader) reader.reset();
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
  height: auto;
  border-radius: 16px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
  object-fit: cover;
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
  font-size: 1.1rem;
  color: #2e7d32;
  font-weight: bold;
  text-align: center;
  word-break: break-all;
}

.hint {
  margin-top: 8px;
  font-size: 1rem;
  color: #555;
}
</style>
