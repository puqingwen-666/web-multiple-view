<script setup>
import { ref, reactive, computed, onMounted, watch } from 'vue'
import MainWindow from './components/MainWindow.vue'
import RightPanel from './components/RightPanel.vue'
import EmbeddedWindow from './components/EmbeddedWindow.vue'

const PRESETS = [
  { title: 'Google', url: 'https://www.google.com' },
  { title: 'MDN Web Docs', url: 'https://developer.mozilla.org' },
  { title: 'GitHub', url: 'https://github.com' },
  { title: 'Stack Overflow', url: 'https://stackoverflow.com' },
  { title: 'Bing', url: 'https://www.bing.com' },
  { title: 'Wikipedia', url: 'https://www.wikipedia.org' }
]

const leftUrl = ref('')
const leftInput = ref('')
const leftKey = ref(0)
const leftRatio = ref(0.7)
const showPlaceholder = ref(true)

const windows = reactive([])
let nextId = 1
let nextZ = 100

const leftPercent = computed(() => (leftRatio.value * 100).toFixed(2) + '%')
const rightPercent = computed(() => ((1 - leftRatio.value) * 100).toFixed(2) + '%')

function loadLeft() {
  let url = leftInput.value.trim()
  if (!url) return
  if (!/^https?:\/\//i.test(url)) url = 'https://' + url
  leftUrl.value = url
  showPlaceholder.value = false
  leftKey.value++
}

function clearLeft() {
  leftUrl.value = ''
  leftInput.value = ''
  showPlaceholder.value = true
  leftKey.value++
}

function startDivider(e) {
  if (e.button !== 0) return
  e.preventDefault()
  const target = e.currentTarget
  try { target.setPointerCapture(e.pointerId) } catch (_) {}
  const appEl = document.querySelector('.app-shell')
  const rect = appEl.getBoundingClientRect()
  function onMove(ev) {
    const r = (ev.clientX - rect.left) / rect.width
    leftRatio.value = Math.min(0.95, Math.max(0.05, r))
  }
  function onUp(ev) {
    try { target.releasePointerCapture(ev.pointerId) } catch (_) {}
    document.removeEventListener('pointermove', onMove)
    document.removeEventListener('pointerup', onUp)
    document.removeEventListener('pointercancel', onUp)
  }
  document.addEventListener('pointermove', onMove)
  document.addEventListener('pointerup', onUp)
  document.addEventListener('pointercancel', onUp)
}

function addWindow({ title, url }) {
  const id = nextId++
  windows.push({
    id,
    title: title || url,
    url,
    mode: 'snapped',
    x: 0,
    y: 0,
    width: 320,
    height: 240,
    z: ++nextZ
  })
}

function removeWindow(id) {
  const idx = windows.findIndex((w) => w.id === id)
  if (idx !== -1) windows.splice(idx, 1)
}

function focusWindow(id) {
  const w = windows.find((w) => w.id === id)
  if (w) w.z = ++nextZ
}

function updateWindow(id, patch) {
  const w = windows.find((w) => w.id === id)
  if (w) Object.assign(w, patch)
}

const DETACH_THRESHOLD = 4

function startSnappedDrag(id, e) {
  if (e.button !== 0) return
  e.preventDefault()
  focusWindow(id)
  const target = e.currentTarget
  try { target.setPointerCapture(e.pointerId) } catch (_) {}
  const start = { x: e.clientX, y: e.clientY }
  const appEl = document.querySelector('.app-shell')
  const appRect = appEl.getBoundingClientRect()
  let detached = false
  let innerRect = null
  let offsetYInEl = 0

  function onMove(ev) {
    const dx = ev.clientX - start.x
    const dy = ev.clientY - start.y
    if (!detached) {
      if (Math.abs(dx) < DETACH_THRESHOLD && Math.abs(dy) < DETACH_THRESHOLD) return
      detached = true
      const w = windows.find((x) => x.id === id)
      if (!w) return
      innerRect = document.querySelector('.right-panel-inner').getBoundingClientRect()
      const snappedCount = windows.filter((x) => x.mode === 'snapped').length
      const idxInSnapped = windows
        .filter((x) => x.mode === 'snapped')
        .findIndex((x) => x.id === id)
      const perHeight = 1 / snappedCount
      const elTop = innerRect.top + idxInSnapped * perHeight * innerRect.height
      const elHeight = perHeight * innerRect.height
      offsetYInEl = Math.max(8, Math.min(elHeight - 24, start.y - elTop))
      const offsetX = 60
      const width = Math.max(280, Math.round(innerRect.width * 0.5))
      const height = Math.max(180, Math.round(innerRect.height * 0.5))
      const nx = ev.clientX - appRect.left - offsetX
      const ny = ev.clientY - appRect.top - offsetYInEl
      updateWindow(id, {
        mode: 'floating',
        x: Math.max(0, nx),
        y: Math.max(0, ny),
        width,
        height,
        z: ++nextZ
      })
      return
    }
    const w = windows.find((x) => x.id === id)
    if (!w) return
    const offsetX = 60
    updateWindow(id, {
      x: Math.max(0, ev.clientX - appRect.left - offsetX),
      y: Math.max(0, ev.clientY - appRect.top - offsetYInEl)
    })
  }

  function onUp(ev) {
    try { target.releasePointerCapture(ev.pointerId) } catch (_) {}
    document.removeEventListener('pointermove', onMove)
    document.removeEventListener('pointerup', onUp)
    document.removeEventListener('pointercancel', onUp)
  }

  document.addEventListener('pointermove', onMove)
  document.addEventListener('pointerup', onUp)
  document.addEventListener('pointercancel', onUp)
}

function snapbackWindow(id) {
  const w = windows.find((w) => w.id === id)
  if (!w || w.mode !== 'floating') return
  updateWindow(id, { mode: 'snapped', z: ++nextZ })
}

onMounted(() => {
  if (localStorage.getItem('cw_leftUrl')) {
    leftUrl.value = localStorage.getItem('cw_leftUrl')
    showPlaceholder.value = false
    leftKey.value++
  }
})

watch(leftUrl, (v) => {
  if (v) localStorage.setItem('cw_leftUrl', v)
})
</script>

<template>
  <div class="app-shell">
    <MainWindow
      class="main"
      :url="leftUrl"
      :input="leftInput"
      :placeholder-shown="showPlaceholder"
      :iframe-key="leftKey"
      @update:input="(v) => (leftInput = v)"
      @load="loadLeft"
      @clear="clearLeft"
    />

    <div class="divider" @pointerdown="startDivider">
      <div class="handle"></div>
    </div>

    <div class="right-zone" :style="{ flex: '0 0 ' + rightPercent }">
      <RightPanel
        :presets="PRESETS"
        :windows="windows"
        @add="addWindow"
        @focus="focusWindow"
        @close="removeWindow"
        @start-drag="startSnappedDrag"
        @snapback="snapbackWindow"
      />
    </div>

    <div class="floating-layer">
      <EmbeddedWindow
        v-for="w in windows.filter((x) => x.mode === 'floating')"
        :key="w.id"
        :model="w"
        :app-rect-provider="() => document.querySelector('.app-shell').getBoundingClientRect()"
        @focus="focusWindow(w.id)"
        @close="removeWindow(w.id)"
        @update="(patch) => updateWindow(w.id, patch)"
        @snapback="snapbackWindow(w.id)"
      />
    </div>
  </div>
</template>

<style scoped>
.app-shell {
  display: flex;
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
}

.main {
  flex: 0 0 v-bind('leftPercent');
  height: 100%;
  min-width: 0;
}

.divider {
  flex: 0 0 6px;
  height: 100%;
  background: #e5e6eb;
  cursor: col-resize;
  position: relative;
  z-index: 5;
  transition: background 0.15s;
  touch-action: none;
}

.divider:hover {
  background: #c5c8ce;
}

.divider .handle {
  position: absolute;
  top: 50%;
  left: 1px;
  width: 4px;
  height: 40px;
  margin-top: -20px;
  background: #8a9099;
  border-radius: 2px;
  opacity: 0.5;
}

.divider:hover .handle {
  opacity: 1;
}

.right-zone {
  height: 100%;
  min-width: 0;
  position: relative;
}

.floating-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 9999;
}

.floating-layer > * {
  pointer-events: auto;
}
</style>
