<template>
  <div class="scanner-root">
    <div class="header">
      <strong>Scanner Datamatrix</strong>
      <div>
        <button @click="toggleTorch" :disabled="!torchAvailable">{{ torchOn ? 'Torch ON' : 'Torch OFF' }}</button>
        <button @click="flipCamera">Changer caméra</button>
        <button @click="capturePhoto">Prendre photo</button>
        <button @click="close">Fermer</button>
      </div>
    </div>

    <div class="video-wrap">
      <video ref="video" playsinline autoplay muted></video>
      <canvas ref="canvas" style="display:none"></canvas>
      <div class="overlay">Placez le code dans le rectangle</div>
    </div>

    <div v-if="message" class="message">{{ message }}</div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { BrowserMultiFormatReader, BarcodeFormat, DecodeHintType } from '@zxing/browser'

export default {
  emits: ['result','close'],
  setup(_, { emit }) {
    const video = ref(null)
    const canvas = ref(null)
    const reader = new BrowserMultiFormatReader()
    let track = null
    let devices = []
    let currentDeviceIndex = 0
    let running = false
    const message = ref('Initialisation...')
    const torchAvailable = ref(false)
    const torchOn = ref(false)
    const attempts = { continuous: 0 }

    // Hints: force DATA_MATRIX to prioritize Datamatrix
    const hints = new Map()
    hints.set(DecodeHintType.POSSIBLE_FORMATS, [BarcodeFormat.DATA_MATRIX])

    // Preferred constraints - request higher resolution but keep reasonable
    const baseConstraints = {
      audio: false,
      video: {
        facingMode: { ideal: 'environment' },
        width: { ideal: 1280 },
        height: { ideal: 720 },
      }
    }

    async function listDevices() {
      try {
        const devs = await BrowserMultiFormatReader.listVideoInputDevices()
        devices = devs
      } catch (e) {
        console.warn('listVideoInputDevices failed', e)
      }
    }

    async function start() {
      await listDevices()
      message.value = 'Initialisation caméra...'
      running = true
      await openCameraWithIndex(currentDeviceIndex)
      // start continuous decode using ZXing on the video stream as fallback fast path
      try {
        reader.decodeFromVideoDevice(null, video.value, (result, err) => {
          if (result) {
            // success
            emit('result', result.getText())
            stop()
          } else {
            // ignore NotFound exceptions
          }
        }, { hints })
      } catch (e) {
        console.warn('decodeFromVideoDevice failed', e)
      }
      // start preprocessing loop which will attempt decode on processed frames (slower but more robust)
      requestAnimationFrame(processLoop)
    }

    async function openCameraWithIndex(index) {
      if (track) {
        try { track.stop() } catch(e){}
        track = null
      }
      // choose deviceId if available
      const deviceId = (devices && devices[index] && devices[index].deviceId) ? devices[index].deviceId : null
      const constraints = deviceId ? { video: { deviceId: { exact: deviceId }, width: { ideal: 1280 }, height: { ideal: 720 }, facingMode: 'environment' } } : baseConstraints
      try {
        const stream = await navigator.mediaDevices.getUserMedia(constraints)
        video.value.srcObject = stream
        // get the video track to permit torch/advanced constraints
        track = stream.getVideoTracks()[0]
        // check for torch support
        try {
          const capabilities = track.getCapabilities ? track.getCapabilities() : {}
          torchAvailable.value = !!capabilities.torch
        } catch (e) {
          torchAvailable.value = false
        }
        // try apply continuous focus if available
        try {
          if (track.applyConstraints) {
            await track.applyConstraints({
              advanced: [
                { focusMode: 'continuous' },
                { width: 1280, height: 720 }
              ]
            }).catch(()=>{})
          }
        } catch(e){}
        message.value = 'Caméra prête — placez la boîte devant la caméra'
      } catch (e) {
        message.value = 'Erreur accès caméra : ' + (e.message || e)
        console.error(e)
      }
    }

    async function toggleTorch() {
      if (!track) return
      try {
        torchOn.value = !torchOn.value
        await track.applyConstraints({ advanced: [{ torch: torchOn.value }] })
      } catch (e) {
        message.value = 'Torch non supportée : ' + (e.message || e)
      }
    }

    function flipCamera() {
      if (!devices || devices.length < 2) {
        message.value = 'Aucune autre caméra détectée'
        return
      }
      currentDeviceIndex = (currentDeviceIndex + 1) % devices.length
      openCameraWithIndex(currentDeviceIndex)
    }

    // pre-processing helpers
    function applyPreprocessing(ctx, w, h) {
      // get image data
      const img = ctx.getImageData(0,0,w,h)
      const data = img.data

      // 1) convert to grayscale + contrast stretching + optional sharpening
      // simple contrast stretch: find min/max luminance
      let min=255, max=0
      const lum = new Uint8ClampedArray(w*h)
      for (let i=0, j=0; i<data.length; i+=4, j++) {
        const r = data[i], g = data[i+1], b = data[i+2]
        const l = (0.2126*r + 0.7152*g + 0.0722*b) | 0
        lum[j] = l
        if (l < min) min = l
        if (l > max) max = l
      }
      const range = Math.max(1, max-min)
      // normalize and threshold-ish (contrast stretch)
      for (let i=0, j=0; i<data.length; i+=4, j++) {
        let v = lum[j]
        // normalize
        v = ((v - min) * 255 / range) | 0
        // lighten slightly / gamma correction cheap
        v = Math.pow(v/255, 0.9) * 255
        data[i] = data[i+1] = data[i+2] = v
      }
      ctx.putImageData(img,0,0)

      // 2) optional unsharp mask (simple)
      // We implement a quick and cheap sharpen: for each pixel mix with neighbors
      // Not perfect but helps with focus
      const tmp = ctx.getImageData(0,0,w,h)
      const td = tmp.data
      const out = new Uint8ClampedArray(td.length)
      const kernel = [0,-1,0,-1,5,-1,0,-1,0] // small sharpen kernel
      for (let y=1; y<h-1; y++) {
        for (let x=1; x<w-1; x++) {
          let idx = (y*w + x)*4
          let sum = 0
          let k=0
          for (let ky=-1; ky<=1; ky++){
            for (let kx=-1; kx<=1; kx++){
              const id = ((y+ky)*w + (x+kx))*4
              const gray = td[id] // already grayscale
              sum += gray * kernel[k++]
            }
          }
          const v = Math.min(255, Math.max(0, sum))
          out[idx] = out[idx+1] = out[idx+2] = v
          out[idx+3] = 255
        }
      }
      // write back inner pixels
      for (let y=1; y<h-1; y++) {
        for (let x=1; x<w-1; x++) {
          const idx = (y*w + x)*4
          td[idx] = out[idx]; td[idx+1] = out[idx+1]; td[idx+2] = out[idx+2]
        }
      }
      ctx.putImageData(tmp,0,0)
    }

    // Try decode from a canvas element using ZXing. Returns text or null
    async function tryDecodeFromCanvas(c) {
      try {
        // create an Image element from canvas to feed ZXing decodeFromImageElement
        const img = new Image()
        const url = c.toDataURL('image/png')
        return await new Promise((resolve) => {
          img.onload = async () => {
            try {
              // the library offers decodeFromImageElement - use it if available
              if (reader && reader.decodeFromImageElement) {
                const r = await reader.decodeFromImageElement(img)
                resolve(r ? r.getText() : null)
              } else if (reader.decodeFromImage) {
                const r = await reader.decodeFromImage(img)
                resolve(r ? r.getText() : null)
              } else {
                // as fallback, try decodeFromVideoDevice already running reduces errors
                resolve(null)
              }
            } catch(e){
              resolve(null)
            }
          }
          img.onerror = () => resolve(null)
          img.src = url
        })
      } catch(e) {
        return null
      }
    }

    // processLoop: capture frames, preprocess, try multiple scales/regions
    async function processLoop() {
      if (!running || !video.value || video.value.readyState < 2) {
        requestAnimationFrame(processLoop)
        return
      }
      attempts.continuous++
      const vid = video.value
      const cw = Math.min(1280, vid.videoWidth)
      const ch = Math.min(720, vid.videoHeight)
      if (cw === 0 || ch === 0) {
        requestAnimationFrame(processLoop)
        return
      }
      const ctx = canvas.value.getContext('2d')
      canvas.value.width = cw
      canvas.value.height = ch
      // center crop a bit to focus on center rectangle (users hold phone and box in center)
      const cropW = Math.floor(cw * 0.7)
      const cropH = Math.floor(ch * 0.5)
      const sx = Math.max(0, Math.floor((vid.videoWidth - cropW)/2))
      const sy = Math.max(0, Math.floor((vid.videoHeight - cropH)/2))
      ctx.drawImage(vid, sx, sy, cropW, cropH, 0, 0, cropW, cropH)

      // preprocess
      applyPreprocessing(ctx, cropW, cropH)

      // try decode on whole crop
      const res1 = await tryDecodeFromCanvas(canvas.value)
      if (res1) { emit('result', res1); stop(); return }
      // try scaled up version (super-res trick: upscale to help small codes)
      const scaleCanvas = document.createElement('canvas')
      const scale = 2
      scaleCanvas.width = cropW * scale
      scaleCanvas.height = cropH * scale
      const sctx = scaleCanvas.getContext('2d')
      // draw scaled up (imageSmoothingQuality high)
      sctx.imageSmoothingEnabled = true
      sctx.imageSmoothingQuality = 'high'
      sctx.drawImage(canvas.value, 0, 0, scaleCanvas.width, scaleCanvas.height)
      // optionally re-apply slight sharpen
      applyPreprocessing(sctx, scaleCanvas.width, scaleCanvas.height)
      const res2 = await tryDecodeFromCanvas(scaleCanvas)
      if (res2) { emit('result', res2); stop(); return }

      // try small tiles (left/right/top/bottom) as sometimes code is at one edge
      const tiles = [
        [0,0,Math.floor(cropW/2),Math.floor(cropH/2)],
        [Math.floor(cropW/2),0,Math.floor(cropW/2),Math.floor(cropH/2)],
        [0,Math.floor(cropH/2),Math.floor(cropW/2),Math.floor(cropH/2)],
        [Math.floor(cropW/2),Math.floor(cropH/2),Math.floor(cropW/2),Math.floor(cropH/2)]
      ]
      for (const t of tiles) {
        const [tx,ty,tw,th] = t
        // draw tile into canvas
        const tileCanvas = document.createElement('canvas')
        tileCanvas.width = tw; tileCanvas.height = th
        const tctx = tileCanvas.getContext('2d')
        tctx.drawImage(vid, sx+tx, sy+ty, tw, th, 0, 0, tw, th)
        applyPreprocessing(tctx, tw, th)
        const r = await tryDecodeFromCanvas(tileCanvas)
        if (r) { emit('result', r); stop(); return }
      }

      // continue loop
      requestAnimationFrame(processLoop)
    }

    // capturePhoto fallback: user explicitly takes a photo (better focus possible)
    async function capturePhoto() {
      if (!track) {
        message.value = 'Caméra non prête'
        return
      }
      try {
        // use ImageCapture if available
        let blob = null
        if (typeof window.ImageCapture === 'function') {
          try {
            const ic = new ImageCapture(track)
            const bmp = await ic.grabFrame() // returns ImageBitmap
            // draw ImageBitmap to canvas
            const ctx = canvas.value.getContext('2d')
            canvas.value.width = bmp.width
            canvas.value.height = bmp.height
            ctx.drawImage(bmp, 0, 0)
            applyPreprocessing(ctx, canvas.value.width, canvas.value.height)
            // try decode
            const res = await tryDecodeFromCanvas(canvas.value)
            if (res) { emit('result', res); stop(); return }
          } catch(e){
            console.warn('ImageCapture fallback failed', e)
          }
        }
        // fallback: draw video frame
        const ctx2 = canvas.value.getContext('2d')
        canvas.value.width = video.value.videoWidth
        canvas.value.height = video.value.videoHeight
        ctx2.drawImage(video.value, 0, 0)
        applyPreprocessing(ctx2, canvas.value.width, canvas.value.height)
        const res2 = await tryDecodeFromCanvas(canvas.value)
        if (res2) { emit('result', res2); stop(); return }
        message.value = 'Photo prise mais code non décodé, essayez d\'éclairer/rapprocher'
      } catch (e) {
        message.value = 'Erreur capture photo: ' + (e.message || e)
      }
    }

    function stop() {
      running = false
      try { reader.reset() } catch(e){}
      if (track) {
        try { track.stop() } catch(e){}
        track = null
      }
      emit('close')
    }

    function close() {
      stop()
    }

    onMounted(() => {
      start()
    })
    onBeforeUnmount(() => {
      stop()
    })

    return { video, canvas, message, torchAvailable, torchOn, toggleTorch, flipCamera, capturePhoto, close }
  }
}
</script>

<style scoped>
.scanner-root { color:#111; background:#fff; padding:8px; max-width:720px; margin: 0 auto; }
.header { display:flex; justify-content:space-between; align-items:center; gap:8px; margin-bottom:8px; }
.video-wrap { position:relative; display:flex; justify-content:center; align-items:center; }
video { width:100%; max-width:680px; border-radius:10px; background:black; }
.overlay { position:absolute; border: 2px dashed rgba(255,255,255,0.8); width:60%; height:40%; pointer-events:none; border-radius:8px; }
.message { margin-top:10px; color:#b33; }
button { margin-left:6px }
</style>
