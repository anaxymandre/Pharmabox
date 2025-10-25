<template>
  <div class="scanner-container">
    <video ref="videoRef" autoplay muted playsinline class="scanner-video"></video>
    <div class="overlay">
      <div class="target"></div>
    </div>
    <p v-if="codeText" class="result">📦 {{ codeText }}</p>
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

  // 🔍 Optimisation : ne scanner que le Datamatrix (plus rapide)
  const formats = [BarcodeFormat.DATA_MATRIX];

  try {
    // Accès à la caméra arrière avec contraintes optimisées
    const constraints = {
      video: {
        facingMode: { ideal: "environment" },
        width: { ideal: 1280 },
        height: { ideal: 720 },
        focusMode: "continuous", // auto-focus continu
        torch: false, // on peut activer la lampe plus tard
      },
    };

    const stream = await navigator.mediaDevices.getUserMedia(constraints);
    videoRef.value.srcObject = stream;

    // 🔄 Boucle de scan continue avec throttling (moins de CPU)
    reader.decodeFromVideoDevice(
      null,
      videoRef.value,
      (result, err) => {
        if (result) {
          const text = result.getText();
          // éviter les doublons (scanner ne s’arrête pas)
          if (text !== codeText.value) codeText.value = text;
        }
      },
      { formats }
    );
  } catch (err) {
    console.error("Erreur accès caméra :", err);
  }
});

onBeforeUnmount(() => {
  if (reader) {
    reader.reset();
  }
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
  font-size: 1.2rem;
  color: #2e7d32;
  font-weight: bold;
  text-align: center;
  word-break: break-all;
}
</style>
