<template>
  <div class="retro-panel" style="z-index:1000;position:fixed;top:6vh;left:0;right:0;margin:auto;max-width:700px;">
    <h2 style="font-family:'VT323',monospace">Edit & Export</h2>
    <canvas ref="editCanvas" :width="canvasWidth" :height="canvasHeight" style="border:2px solid #92ffb9;border-radius:8px;margin:auto;display:block"></canvas>
    <div style="margin:1.3em 0;">
      <label class="retro-label">Filter:</label>
      <select v-model="selectedFilter" @change="applyAll">
        <option v-for="f in filters" :value="f.value">{{ f.label }}</option>
      </select>
      <label class="retro-label" style="margin-left:1em;">Text:</label>
      <input v-model="customText" placeholder="Type here" style="width:100px;" />
      <select v-model="selectedFont" @change="applyAll" class="font-sample">
        <option v-for="(font, k) in fontOptions" :value="font" :style="{fontFamily: font}">{{ k }}</option>
      </select>
      <input v-model.number="fontSize" type="number" min="8" max="64" style="width:55px;" @input="applyAll" />
      <input type="color" v-model="fontColor" @input="applyAll"/>
      <br>
      <label class="retro-label" style="margin-top:1em;">Add Watermark:</label>
      <input v-model="watermarkText" placeholder="Your watermark" style="width:110px;" />
      <select v-model="watermarkFont" class="font-sample">
        <option v-for="(font, k) in fontOptions" :value="font" :style="{fontFamily: font}">{{ k }}</option>
      </select>
      <input type="color" v-model="watermarkColor"/>
      <button class="retro-btn" @mousedown.stop.prevent="moveMark = !moveMark">{{moveMark?'Place':'Move'}}</button>
      <input type="file" accept="image/*" @change="onWatermarkImage"/>
      <span style="font-size:11px;">(Text and/or upload)</span>
    </div>
    <div style="margin:1.7em 0;text-align:center">
      <button class="retro-btn" @click="onExport">Download</button>
      <button class="retro-btn" style="margin-left:1.5em" @click="$emit('close')">Cancel</button>
    </div>
  </div>
</template>
<script setup>
import { ref, watch, onMounted } from 'vue'
// Props: canvas (source to edit)
const props = defineProps(['canvas'])
const emit = defineEmits(['close','save'])
// -- image state
const editCanvas = ref(null)
const canvasWidth = ref(1200)
const canvasHeight = ref(900)
const baseImage = ref(null)
// Editable overlays
const filters = [
  { label:'None', value:'none' },
  { label:'Grayscale', value:'grayscale' },
  { label:'Sepia', value:'sepia' },
  { label:'Invert', value:'invert' },
  { label:'Brightness+', value:'brightness' },
  { label:'Contrast+', value:'contrast' },
]
const selectedFilter = ref('none')
const customText = ref('')
const selectedFont = ref("'VT323', 'IBM Plex Mono', monospace")
const fontOptions = {
  'Retro (VT323)': "'VT323', monospace",
  'Pixel (Roboto Mono)': "'Roboto Mono', monospace",
  'Typewriter (IBM Plex)': "'IBM Plex Mono', monospace",
  'Gothic (UnifrakturCook)': "'UnifrakturCook', cursive",
  'Fraktur': "'Fruktur', cursive",
  'Serif': "serif", 'Sans': "sans-serif", 'Mono': "monospace" }
const fontSize = ref(32)
const fontColor = ref('#2c3636')
// Watermark
const watermarkText = ref('')
const watermarkFont = ref("'VT323', monospace")
const watermarkColor = ref('#99b886')
const watermarkImage = ref(null)
const watermarkImgObj = ref(null)
const watermarkPos = ref({x:0.8, y:0.95}); // normalized % on canvas
const moveMark = ref(false)

// update from main view
watch(() => props.canvas, () => { setupImage(); })

onMounted(() => { setupImage(); })

function setupImage() {
  if (!props.canvas) return
  canvasWidth.value = props.canvas.width
  canvasHeight.value = props.canvas.height
  // Draw the base to editor canvas
  let ctx = editCanvas.value.getContext('2d')
  ctx.clearRect(0,0,canvasWidth.value,canvasHeight.value)
  ctx.drawImage(props.canvas,0,0)
  baseImage.value = ctx.getImageData(0,0,canvasWidth.value,canvasHeight.value)
  applyAll()
}
// Apply all overlays/effects
function applyAll() {
  if (!editCanvas.value || !baseImage.value) return
  let ctx = editCanvas.value.getContext('2d')
  ctx.putImageData(baseImage.value,0,0)
  // Apply simple filter
  let filter = selectedFilter.value
  if (filter !== 'none') doFilter(ctx, filter)
  // Custom Text
  if (customText.value) {
    ctx.save()
    ctx.font = fontSize.value+'px '+selectedFont.value
    ctx.fillStyle = fontColor.value
    ctx.textAlign = 'center'
    ctx.textBaseline = 'top'
    ctx.shadowColor = "#fff8"
    ctx.shadowBlur = 8
    ctx.fillText(customText.value, canvasWidth.value/2,fontSize.value)
    ctx.restore()
  }
  // Watermark
  if (watermarkText.value) {
    ctx.save()
    ctx.font = Math.round(fontSize.value*0.8)+'px '+watermarkFont.value
    ctx.fillStyle = watermarkColor.value
    ctx.globalAlpha = 0.86
    ctx.textAlign='right'
    ctx.shadowColor="#0004"
    ctx.shadowBlur = 3
    ctx.fillText(watermarkText.value, 
      canvasWidth.value*watermarkPos.value.x, 
      canvasHeight.value*watermarkPos.value.y)
    ctx.restore()
  }
  // Watermark image
  if (watermarkImgObj.value) {
    let w = canvasWidth.value*0.15
    let h = watermarkImgObj.value.height* w / watermarkImgObj.value.width
    ctx.globalAlpha = 0.9
    ctx.drawImage(watermarkImgObj.value, canvasWidth.value*watermarkPos.value.x - w, canvasHeight.value*watermarkPos.value.y-h, w, h)
    ctx.globalAlpha = 1
  }
}
function doFilter(ctx, which) {
  let img = ctx.getImageData(0,0,canvasWidth.value,canvasHeight.value)
  let d = img.data
  switch (which) {
    case 'grayscale':
      for(let i=0;i<d.length;i+=4){
        let avg = (d[i]+d[i+1]+d[i+2])/3
        d[i]=d[i+1]=d[i+2]=avg
      } break;
    case 'sepia':
      for(let i=0;i<d.length;i+=4){
        let r=d[i],g=d[i+1],b=d[i+2]
        d[i]=Math.min(255, r*0.393+g*0.769+b*0.189)
        d[i+1]=Math.min(255, r*0.349+g*0.686+b*0.168)
        d[i+2]=Math.min(255, r*0.272+g*0.534+b*0.131)
      } break;
    case 'invert':
      for(let i=0;i<d.length;i+=4) d[i]=255-d[i],d[i+1]=255-d[i+1],d[i+2]=255-d[i+2];
      break
    case 'brightness':
      for(let i=0;i<d.length;i+=4) d[i]*=1.2, d[i+1]*=1.2, d[i+2]*=1.2;
      break
    case 'contrast':
      let f = (259 * (128+64)) / (255 * (259-64));
      for(let i=0;i<d.length;i+=4) {
        d[i]=f*(d[i]-128)+128; d[i+1]=f*(d[i+1]-128)+128; d[i+2]=f*(d[i+2]-128)+128;
      } break
  }
  ctx.putImageData(img,0,0)
}
// Handle drag to move watermark mark
function onCanvasClick(e) {
  if (!moveMark.value) return
  // place watermark at click pos
  let rect = editCanvas.value.getBoundingClientRect()
  let x = (e.clientX-rect.left)/rect.width
  let y = (e.clientY-rect.top)/rect.height
  watermarkPos.value = {x, y}
  moveMark.value = false
  applyAll()
}
function onWatermarkImage(ev) {
  let file = ev.target.files[0]
  if (!file) return
  let img = new window.Image()
  img.onload = () => { watermarkImgObj.value = img; applyAll() }
  img.src = URL.createObjectURL(file)
}
function onExport() {
  editCanvas.value.toBlob(blob => emit('save', blob), 'image/png', 0.98)
}
</script>
<style>
/* Overlay to dim background */
.retro-panel {
  background: #100c18e8;
  border: 2.5px solid #89ff9b;
  box-shadow: 0 4px 19px #1f1e1a99;
  margin: auto;
}
</style>
<!-- Listeners for canvas drag -->
<script>
export default {
  mounted() {
    this.$refs.editCanvas.addEventListener('click', this.onCanvasClick)
  },
  beforeUnmount() {
    this.$refs.editCanvas.removeEventListener('click', this.onCanvasClick)
  }
}
</script>