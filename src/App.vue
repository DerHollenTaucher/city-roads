<template>
  <div class="retro-panel">
    <h1 class="retro-label">city-roads retro</h1>
    <div v-if="downloadError" class="error">
      <p>{{ downloadErrorMessage }}</p>
      <button class="retro-btn" @click="retryFetch">Retry</button>
    </div>
    <div v-else>
      <!-- Your main map-input & fetch UI here -->

      <MapInput @loadCity="fetchCityData"/>
      <MapViewer ref="mapViewer" v-if="cityData" :data="cityData" />

      <button v-if="cityData" class="retro-btn" @click="openImageEditor">Edit & Export Image</button>
    </div>
    <ImageEditor
      v-if="showImageEditor"
      :canvas="generatedCanvas"
      @close="showImageEditor = false"
      @save="downloadEditedImage"
    />
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import MapInput from './components/MapInput.vue'
import MapViewer from './components/MapViewer.vue'
import ImageEditor from './components/ImageEditor.vue'

const cityData = ref(null)
const generatedCanvas = ref(null)
const showImageEditor = ref(false)
const downloadError = ref(false)
const downloadErrorMessage = ref('')

// Helper: try data fetch with retries
async function fetchWithRetry(fetchFunc, retries = 3, delay = 1500) {
  let lastErr;
  for (let i = 0; i < retries; ++i) {
    try {
      return await fetchFunc();
    } catch (e) {
      lastErr = e;
      await new Promise(r => setTimeout(r, delay * (i + 1)));
    }
  }
  throw lastErr;
}

function fetchCityData(params) {
  downloadError.value = false;
  fetchWithRetry(() => fetchActualCityData(params))
    .then(data => {
      cityData.value = data
      nextTick(() => {
        // render map to canvas
        const viewer = this.$refs.mapViewer;
        generatedCanvas.value = viewer ? viewer.getCanvas() : null
      })
    })
    .catch(err => {
      downloadError.value = true;
      downloadErrorMessage.value = `Error downloading city map data: ${err.message || err}`;
    });
}
function retryFetch() {
  // re-trigger last fetch with previous params
  fetchCityData(cityData.value?.lastParams);
}
function openImageEditor() {
  if (generatedCanvas.value) showImageEditor.value = true
}
function downloadEditedImage(imgBlob) {
  // e.g. trigger download, or close dialog
  showImageEditor.value = false;
  // Optionally, auto-download the image here
  const url = URL.createObjectURL(imgBlob)
  const a = document.createElement('a')
  a.href = url
  a.download = `city-roads-edited.png`
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  setTimeout(() => URL.revokeObjectURL(url), 100)
}
</script>
<style src="./styles/retro.css"></style>