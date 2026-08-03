<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import MainWindow from './components/MainWindow.vue'
import RightPanel from './components/RightPanel.vue'
import EmbeddedWindow from './components/EmbeddedWindow.vue'
import FabTree from './components/FabTree.vue'

const MOBILE_BREAKPOINT = 768
const MAX_NODES = 15

const isMobile = ref(false)

function updateBreakpoint() {
  isMobile.value = typeof window !== 'undefined' && window.innerWidth < MOBILE_BREAKPOINT
}

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

const nodes = reactive([])
const childrenOf = reactive({})
let h5NextId = 1

const activeId = ref(null)
const fabOpen = ref(false)

const rootNodes = computed(() => nodes.filter((n) => n.parentId === null))
const activeNode = computed(() => nodes.find((n) => n.id === activeId.value))

function buildTree(node) {
  const children = (childrenOf[node.id] || [])
    .map((cid) => nodes.find((n) => n.id === cid))
    .filter(Boolean)
  return { ...node, children: children.map(buildTree) }
}

const tree = computed(() => rootNodes.value.map(buildTree))

const inputUrl = ref('')
const inputTitle = ref('')
const pendingParentId = ref(null)
const showAddDialog = ref(false)

function normalizeUrl(raw) {
  let url = (raw || '').trim()
  if (!url) return ''
  if (!/^https?:\/\//i.test(url)) url = 'https://' + url
  return url
}

function deriveTitle(url) {
  try {
    const u = new URL(url)
    return u.hostname.replace(/^www\./, '').slice(0, 30)
  } catch (_) {
    return url.slice(0, 30)
  }
}

function selectNode(id) {
  if (!nodes.find((n) => n.id === id)) return
  activeId.value = id
  fabOpen.value = false
}

function openAddDialog(parentId) {
  pendingParentId.value = parentId
  inputUrl.value = ''
  inputTitle.value = ''
  showAddDialog.value = true
}

function closeAddDialog() {
  showAddDialog.value = false
  inputUrl.value = ''
  inputTitle.value = ''
  pendingParentId.value = null
}

function confirmAdd() {
  const url = normalizeUrl(inputUrl.value)
  if (!url) return
  const parentId = pendingParentId.value
  const title = (inputTitle.value.trim() || deriveTitle(url))
  if (parentId === null) {
    const existingRoot = nodes.find((n) => n.parentId === null)
    if (existingRoot) {
      existingRoot.url = url
      existingRoot.title = title
      activeId.value = existingRoot.id
    } else {
      if (nodes.length >= MAX_NODES) {
        alert(`最多只能打开 ${MAX_NODES} 个窗口`)
        return
      }
      const id = h5NextId++
      nodes.push({ id, parentId: null, url, title })
      activeId.value = id
    }
  } else {
    if (nodes.length >= MAX_NODES) {
      alert(`最多只能打开 ${MAX_NODES} 个窗口`)
      return
    }
    const id = h5NextId++
    nodes.push({ id, parentId, url, title })
    if (!childrenOf[parentId]) childrenOf[parentId] = []
    childrenOf[parentId].push(id)
    activeId.value = id
  }
  fabOpen.value = false
  closeAddDialog()
}

function removeNode(id) {
  const toDelete = []
  function collect(cid) {
    toDelete.push(cid)
    for (const childId of childrenOf[cid] || []) collect(childId)
  }
  collect(id)
  for (const delId of toDelete) {
    const i = nodes.findIndex((n) => n.id === delId)
    if (i !== -1) nodes.splice(i, 1)
    delete childrenOf[delId]
    for (const k in childrenOf) {
      const arr = childrenOf[k]
      const j = arr.indexOf(delId)
      if (j !== -1) arr.splice(j, 1)
    }
  }
  activeId.value = nodes.length ? nodes[0].id : null
}

onMounted(() => {
  updateBreakpoint()
  window.addEventListener('resize', updateBreakpoint)

  if (localStorage.getItem('cw_leftUrl')) {
    leftUrl.value = localStorage.getItem('cw_leftUrl')
    showPlaceholder.value = false
    leftKey.value++
  }

  const saved = localStorage.getItem('cw_nodes')
  if (saved) {
    try {
      const data = JSON.parse(saved)
      data.nodes?.forEach((n) => {
        nodes.push(n)
        if (n.id >= h5NextId) h5NextId = n.id + 1
      })
      for (const k in data.childrenOf || {}) {
        childrenOf[k] = data.childrenOf[k]
      }
      activeId.value = data.activeId ?? (nodes[0]?.id ?? null)
    } catch (_) {}
  }
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', updateBreakpoint)
})

watch(leftUrl, (v) => {
  if (v) localStorage.setItem('cw_leftUrl', v)
})

watch(
  [nodes, childrenOf, activeId],
  () => {
    localStorage.setItem(
      'cw_nodes',
      JSON.stringify({
        nodes: nodes.map((n) => ({ ...n })),
        childrenOf: { ...childrenOf },
        activeId: activeId.value
      })
    )
  },
  { deep: true }
)
</script>

<template>
  <div class="app-shell">
    <template v-if="!isMobile">
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
    </template>

    <template v-else>
      <div v-if="!activeNode && nodes.length === 0" class="welcome">
        <div class="welcome-card">
          <div class="welcome-title">嵌入看板 · H5</div>
          <div class="welcome-desc">输入网址打开主网页，右下角悬浮按钮可新建窗口</div>
          <input
            class="welcome-input"
            v-model="inputUrl"
            placeholder="输入网址，如 https://example.com"
            @keydown.enter="confirmAdd"
          />
          <button class="welcome-btn" @click="confirmAdd" :disabled="!inputUrl.trim()">打开</button>
        </div>
      </div>

      <div v-else class="frames-stack">
        <iframe
          v-for="node in nodes"
          :key="node.id"
          :src="node.url"
          class="main-frame"
          :class="{ hidden: node.id !== activeId }"
          frameborder="0"
          allow="fullscreen"
          sandbox="allow-same-origin allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-downloads"
          referrerpolicy="no-referrer"
        ></iframe>
      </div>

      <FabTree
        :tree="tree[0] || {}"
        :active-id="activeId"
        :open="fabOpen"
        @toggle="fabOpen = !fabOpen"
        @select="selectNode"
        @add-root="openAddDialog(null)"
        @add-child="(pid) => openAddDialog(pid)"
        @close="removeNode"
      />
    </template>

    <div v-if="showAddDialog" class="dialog-mask" @click="closeAddDialog">
      <div class="dialog" @click.stop>
        <div class="dialog-title">
          {{ pendingParentId === null ? '新建主窗口' : '新建子窗口' }}
        </div>
        <input
          class="dialog-input"
          v-model="inputTitle"
          placeholder="标题（可选，留空用网址）"
        />
        <input
          class="dialog-input"
          v-model="inputUrl"
          placeholder="https://..."
          autofocus
          @keydown.enter="confirmAdd"
        />
        <div class="dialog-actions">
          <button class="cancel" @click="closeAddDialog">取消</button>
          <button class="ok" @click="confirmAdd" :disabled="!inputUrl.trim()">打开</button>
        </div>
      </div>
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

.welcome {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #f5f7fa 0%, #e9eef5 100%);
  padding: 20px;
}

.welcome-card {
  width: 100%;
  max-width: 420px;
  background: #fff;
  border-radius: 16px;
  padding: 28px 22px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.08);
  text-align: center;
}

.welcome-title {
  font-size: 20px;
  font-weight: 700;
  color: #1f2329;
  margin-bottom: 8px;
}

.welcome-desc {
  font-size: 13px;
  color: #8a9099;
  margin-bottom: 20px;
  line-height: 1.6;
}

.welcome-input {
  width: 100%;
  height: 44px;
  padding: 0 14px;
  border: 1px solid #d8dce2;
  border-radius: 10px;
  font-size: 15px;
  outline: none;
  box-sizing: border-box;
  margin-bottom: 12px;
}

.welcome-input:focus {
  border-color: #3b82f6;
}

.welcome-btn {
  width: 100%;
  height: 44px;
  border: 0;
  border-radius: 10px;
  background: #3b82f6;
  color: #fff;
  font-size: 15px;
  cursor: pointer;
}

.welcome-btn:active {
  background: #2f6fe0;
}

.main-frame {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  border: 0;
  display: block;
}

.main-frame.hidden {
  visibility: hidden;
  pointer-events: none;
}

.frames-stack {
  position: absolute;
  inset: 0;
}

.dialog-mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.4);
  z-index: 10000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.dialog {
  width: 100%;
  max-width: 380px;
  background: #fff;
  border-radius: 14px;
  padding: 18px 16px;
  box-shadow: 0 12px 36px rgba(0, 0, 0, 0.2);
}

.dialog-title {
  font-size: 16px;
  font-weight: 600;
  color: #1f2329;
  margin-bottom: 14px;
}

.dialog-input {
  width: 100%;
  height: 44px;
  padding: 0 12px;
  border: 1px solid #d8dce2;
  border-radius: 8px;
  font-size: 15px;
  outline: none;
  box-sizing: border-box;
  margin-bottom: 14px;
}

.dialog-input:focus {
  border-color: #3b82f6;
}

.dialog-actions {
  display: flex;
  gap: 10px;
}

.dialog-actions button {
  flex: 1;
  height: 42px;
  border: 0;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
}

.cancel {
  background: #f0f1f3;
  color: #4b5159;
}

.ok {
  background: #3b82f6;
  color: #fff;
}

.ok:disabled {
  background: #c5c8ce;
}

.ok:not(:disabled):active {
  background: #2f6fe0;
}
</style>
