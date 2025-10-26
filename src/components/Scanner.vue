<template>
  <div class="scanner-container">
    <video ref="videoRef" autoplay muted playsinline class="scanner-video" v-show="!codeText"></video>
    <canvas ref="canvasRef" class="hidden-canvas"></canvas>

    <div class="overlay" v-if="!codeText"><div class="target"></div></div>

    <!-- Résultat -->
    <div v-if="codeText" class="result-card">
      <h2>📦 Informations détectées</h2>
      <p><strong>Code CIP :</strong> {{ parsed.cip || "—" }}</p>
      <p><strong>Date d’expiration :</strong> {{ parsed.expiration || "—" }}</p>
      <p><strong>Lot :</strong> {{ parsed.lot || "—" }}</p>
      <p><strong>N° de série :</strong> {{ parsed.serial || "—" }}</p>
    </div>

    <p v-else class="hint">Alignez le code Datamatrix dans le carré vert…</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
import { BrowserMultiFormatReader } from "@zxing/browser";

const videoRef = ref(null);
const canvasRef = ref(null);
const codeText = ref("");
const parsed = ref({});
let reader, ctx, loopId, stream;

/**
 * ✅ Décode un Datamatrix pharmaceutique GS1 selon la norme européenne
 * Source : Directive 2011/62/UE + https://meditrust.io/datamatrix-definition
 * Gère les cas sans séparateurs visibles (ASCII 29)
 */
function parseGS1Pharma(raw) {
  const data = {};
  if (!raw) return data;

  // 1. Nettoyage
  const text = raw.replace(/[\u0000-\u001F\u007F]/g, "").trim();

  // 2. On cherche les AI dans n’importe quel ordre
  // Chaque AI est suivi immédiatement par sa donnée
  const aiPatterns = [
    { key: "cip", ai: "01", length: 14, fixed: true }, // GTIN / CIP
    { key: "expiration", ai: "17", length: 6, fixed: true },
    { key: "lot", ai: "10", fixed: false }, // jusqu’à prochain AI
    { key: "serial", ai: "21", fixed: false },
  ];

  let pos = 0;
  while (pos < text.length) {
    const ai = text.slice(pos, pos + 2);
    const pattern = aiPatterns.find((p) => p.ai === ai);
    if (pattern) {
      pos += 2;
      if (pattern.fixed) {
        const val = text.slice(pos, pos + pattern.length);
        if (pattern.ai === "01") data.cip = val;
        if (pattern.ai === "17") {
          const yy = "20" + val.slice(0, 2);
          const mm = val.slice(2, 4);
          const dd = val.slice(4, 6);
          data.expiration = `${dd}/${mm}/${yy}`;
        }
        pos += pattern.length;
      } else {
        // variable-length (lot ou serial)
        let next = text.slice(pos).match(/(01|17|10|21|00|30|37|400|415|7001)/);
        const end = next ? pos + next.index : text.length;
        const val = text.slice(pos, end);
        if (pattern.ai === "10") data.lot = val;
        if (pattern.ai === "21") data.serial = val;
        pos = end;
      }
    } else {
      pos++;
    }
  }

  return data;
}

/** Démarre le scanner */
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

      try {
        const result = await reader.decodeFromCanvas(canvas);
        if (result) {
          const text = result.getText();
          if (text && text !== codeText.value) {
            codeText.value = text;
            parsed.value = parseGS1Pharma(text);
          }
        }
      } catch {
        /* aucun code trouvé */
      }

      loopId = requestAnimationFrame(scan);
    };
    scan();
  } catch (err) {
    console.error("Erreur caméra :", err);
  }
}

/** Réinitialise tout quand on clique sur "Scanner un médicament" */
function resetScanner() {
  codeText.value = "";
  parsed.value = {};
  if (loopId) cancelAnimationFrame(loopId);
  if (reader) reader.reset();
  if (stream) stream.getTracks().forEach((t) => t.stop());
  startScanner();
}

// Option : si ton bouton principal "Scanner un médicament" déclenche un event global
// tu peux écouter un signal via un prop ou un store si besoin
watch(codeText, (newVal) => {
  // tu peux loguer ici pour debug
  console.log("Code brut :", newVal);
});

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
