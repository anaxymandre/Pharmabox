<template>
  <div style="position:relative;">
    <video ref="videoRef" autoplay muted playsinline style="width:100%;border-radius:12px;"></video>
    <div style="margin-top:10px; display:flex; gap:8px;">
      <button @click="stopScan" style="padding:8px 12px;">Fermer</button>
      <button @click="switchCamera" style="padding:8px 12px;">Changer caméra</button>
    </div>
  </div>
</template>

<script>
import { BrowserMultiFormatReader } from '@zxing/browser'
import { ref, onMounted, onBeforeUnmount } from 'vue'

export default {
  emits: ['result', 'close'],
  setup(_, { emit }) {
    const codeReader = new BrowserMultiFormatReader()
    const videoRef = ref(null)
    const currentDeviceId = ref(null)
    const devices = ref([])

    async function startScan() {
      try {
        const foundDevices = await BrowserMultiFormatReader.listVideoInputDevices()
        devices.value = foundDevices
        currentDeviceId.value = foundDevices.find(d => d.label.toLowerCase().includes('back'))?.deviceId || foundDevices[0].deviceId
        await codeReader.decodeFromVideoDevice(currentDeviceId.value, videoRef.value, (result, err) => {
          if (result) {
            emit('result', result.getText())
            stopScan()
          }
        })
      } catch (e) {
        console.error(e)
      }
    }

    function stopScan() {
      codeReader.reset()
      emit('close')
    }

    function switchCamera() {
      if (devices.value.length > 1) {
        const idx = devices.value.findIndex(d => d.deviceId === currentDeviceId.value)
        currentDeviceId.value = devices.value[(idx + 1) % devices.value.length].deviceId
        stopScan()
        setTimeout(startScan, 300)
      }
    }

    onMounted(startScan)
    onBeforeUnmount(stopScan)

    return { videoRef, stopScan, switchCamera }
  }
}
</script>
