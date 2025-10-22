<template>
  <div style="padding:16px; font-family: Arial, sans-serif;">
    <h1>Pharmabox</h1>
    <p>Scannez le Datamatrix d'une boîte de médicament.</p>

    <div style="margin-top:16px;">
      <button @click="showScanner = true" style="padding:10px 16px; font-size:16px;">
        Scanner un médicament
      </button>
    </div>

    <Scanner v-if="showScanner" @close="showScanner=false" @result="handleResult" />

    <div v-if="lastResult" style="margin-top:20px;">
      <h3>Résultat brut :</h3>
      <pre>{{ lastResult }}</pre>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import Scanner from './components/Scanner.vue'

export default {
  components: { Scanner },
  setup() {
    const showScanner = ref(false)
    const lastResult = ref(null)
    function handleResult(v) { lastResult.value = v; showScanner.value = false }
    return { showScanner, lastResult, handleResult }
  }
}
</script>
