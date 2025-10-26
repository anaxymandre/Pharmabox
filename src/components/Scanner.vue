<template>
  <div class="scanner-container">
    <video ref="videoRef" autoplay muted playsinline class="scanner-video" v-show="!codeText"></video>
    <canvas ref="canvasRef" class="hidden-canvas"></canvas>

    <div class="overlay" v-if="!codeText"><div class="target"></div></div>

    <!-- Affichage des infos médicament -->
    <div v-if="codeText" class="result-card">
      <h2>📦 Informations détectées</h2>
      <p><strong>Code CIP :</strong> {{ parsed.cip || "—" }}</p>
      <p><strong>Date d’expiration :</strong> {{ parsed.expiration || "—" }}</p>
      <p><strong>Lot :</strong> {{ parsed.lot || "—" }}</p>
      <p><strong>N° de série :</strong> {{ parsed.serial || "—" }}</p>
      <button @click="resetScan" class="reset-btn">🔄 Scanner un autre médicament</button>
    </div>

    <p v-else class="hint">Alignez le code Datamatrix dans le carré vert…</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import { BrowserMultiFormatReader, BarcodeFormat } from "@zxing/browser";

const videoRef = ref(null);
const canvasRef = ref(null);
const codeText = ref("");
const parsed = ref({});
let reader, ctx, loopId, stream;

/** Fonction pour parser le contenu Datamatrix GS1 */
function parseGS1(text) {
  const data = {};
  const regex = /\((\d{2})\)([^\(]+)/g;
  let match;
  while ((match = regex.exec(text))) {
    const [, ai, value] = match;
    switch (ai) {
      case "01":
        data.cip = value.trim();
        break;
      case "17":
        // format YYMMDD → conversion lisible
        const yy = "20" + value.substring(0, 2);
        const mm = value.substring(2, 4);
        const dd = value.substring(4, 6);
        data.expiration = `${dd}/${mm}/${yy}`;
        break;
      case "10":
        data.lot = value.trim();
        break;
      case "21":
        data.serial = value.trim();
        break;
    }
  }
  return data;
}

async function startScanner() {
  reader = new BrowserMultiFormatReader();

  const constraints = {
    video: {
      facingMode: { ideal: "environment" },
      width: { ideal: 1280 },
      height: { ideal: 720 },
    },
  };

  try {
    stream = await navigator.mediaDevices.getUserMedia(constraints);
    videoRef.value.srcObject = stream;

    await new Promise((r) => (videoRef.value.onloadedmetadata = r));

    const video = videoRef.value;
    const canvas = canvasRef.value;
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;
    ctx = canvas.getContext("2d");

    const scan = async () => {
      if (!video || video.readyState !== 4) return;
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

      const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const data = imgData.data;
      for (let i = 0; i < data.length; i += 4) {
        const gray = data[i] * 0.3 + data[i + 1] * 0.59 + data[i + 2] * 0.11;
        const factor = 1.5;
        const newVal = Math.min(255, Math.max(0, (gray - 128) * factor + 128));
        data[i] = data[i + 1] = data[i + 2] = newVal;
      }
      ctx.putImageData(imgData, 0, 0);

      try {
        const result = await reader.decodeFromCanvas(canvas);
        if (result && result.getText() !== codeText.value) {
          codeText.value = result.getText();
          parsed.value = parseGS1(codeText.value);
        }
      } catch {
        /* pas de code trouvé */
      }

      loopId = requestAnimationFrame(scan);
    };
    scan();
  } catch (err) {
    console.error("Erreur caméra :", err);
  }
}

/** Réinitialiser le scanner */
function resetScan() {
  codeText.value = "";
  parsed.value = {};
  if (loopId) cancelAnimationFrame(loopId);
  if (reader) reader.reset();
  if (stream) {
    stream.getTracks().forEach((t) => t.stop());
  }
  startScanner();
}

onMounted(startScanner);
onBeforeUnmount(() => {
  if (reader) reader.reset();
  if (loopId) cancelAnimationFrame(loopId);
  if (stream) stream.getTracks().forEach((t) => t.stop());
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
.result-card {
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 0 12px rgba(0, 0, 0, 0.2);
  padding: 20px;
  max-width: 90%;
  text-align: left;
  font-size: 1rem;
}
.reset-btn {
  background: #4caf50;
  color: white;
  border: none;
  padding: 10px 16px;
  border-radius: 8px;
  margin-top: 12px;
  font-size: 1rem;
}
.reset-btn:hover {
  background: #43a047;
}
.hint {
  margin-top: 10px;
  font-size: 1rem;
  color: #555;
}
</style>
