<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  model: { type: Object, required: true },
  appRectProvider: { type: Function, default: () => () => ({ left: 0, top: 0 }) }
})

const emit = defineEmits(['focus', 'close', 'update', 'snapback'])

const dragging = ref(false)
const resizing = ref(false)

const style = computed(() => ({
  left: props.model.x + 'px',
  top: props.model.y + 'px',
  width: props.model.width + 'px',
  height: props.model.height + 'px',
  zIndex: props.model.z
}))

function startDrag(e) {
  if (e.button !== 0) return
  e.preventDefault()
  e.stopPropagation()
  emit('focus')
  const target = e.currentTarget
  try { target.setPointerCapture(e.pointerId) } catch (_) {}
  const startX = e.clientX
  const startY = e.clientY
  const origX = props.model.x
  const origY = props.model.y
  dragging.value = true

  function onMove(ev) {
    const nx = origX + (ev.clientX - startX)
    const ny = origY + (ev.clientY - startY)
    emit('update', { x: nx, y: ny })
  }
  function onUp(ev) {
    dragging.value = false
    try { target.releasePointerCapture(ev.pointerId) } catch (_) {}
    document.removeEventListener('pointermove', onMove)
    document.removeEventListener('pointerup', onUp)
    document.removeEventListener('pointercancel', onUp)
  }
  document.addEventListener('pointermove', onMove)
  document.addEventListener('pointerup', onUp)
  document.addEventListener('pointercancel', onUp)
}

function startResize(e) {
  if (e.button !== 0) return
  e.preventDefault()
  e.stopPropagation()
  emit('focus')
  const target = e.currentTarget
  try { target.setPointerCapture(e.pointerId) } catch (_) {}
  const startX = e.clientX
  const startY = e.clientY
  const origW = props.model.width
  const origH = props.model.height
  resizing.value = true

  function onMove(ev) {
    const nw = Math.max(240, origW + (ev.clientX - startX))
    const nh = Math.max(160, origH + (ev.clientY - startY))
    emit('update', { width: nw, height: nh })
  }
  function onUp(ev) {
    resizing.value = false
    try { target.releasePointerCapture(ev.pointerId) } catch (_) {}
    document.removeEventListener('pointermove', onMove)
    document.removeEventListener('pointerup', onUp)
    document.removeEventListener('pointercancel', onUp)
  }
  document.addEventListener('pointermove', onMove)
  document.addEventListener('pointerup', onUp)
  document.addEventListener('pointercancel', onUp)
}

function onContext(e) {
  e.preventDefault()
  emit('snapback')
}
</script>

<template>
  <div
    class="embedded"
    :style="style"
    @pointerdown="emit('focus')"
    @contextmenu="onContext"
  >
    <header class="bar" @pointerdown="startDrag" :style="{ touchAction: 'none' }">
      <span class="title" :title="model.title">{{ model.title }}</span>
      <a
        class="open-ext"
        :href="model.url"
        target="_blank"
        rel="noopener"
        @pointerdown.stop
        @click.stop
        title="在新标签打开"
      >↗</a>
      <button
        class="snapback"
        @click.stop="emit('snapback')"
        @pointerdown.stop
        title="回归右侧布局（右键 / ↩）"
      >↩</button>
      <button class="close" @click.stop="emit('close')" @pointerdown.stop title="关闭">×</button>
    </header>

    <div class="frame-wrap">
      <iframe
        :src="model.url"
        frameborder="0"
        allow="fullscreen"
        sandbox="allow-same-origin allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-downloads"
        referrerpolicy="no-referrer"
      ></iframe>
      <div v-if="dragging || resizing" class="mask"></div>
    </div>

    <div class="resize-handle" @pointerdown="startResize" :style="{ touchAction: 'none' }"></div>
  </div>
</template>

<style scoped>
.embedded {
  position: absolute;
  background: #fff;
  border: 1px solid #b8c0cc;
  border-radius: 8px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.16);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.bar {
  height: 32px;
  display: flex;
  align-items: center;
  padding: 0 8px 0 12px;
  background: #f5f6f7;
  border-bottom: 1px solid #e5e6eb;
  cursor: move;
  user-select: none;
  gap: 6px;
  flex-shrink: 0;
  touch-action: none;
}

.title {
  flex: 1;
  font-size: 13px;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.open-ext,
.snapback,
.close {
  border: 0;
  background: transparent;
  color: #8a9099;
  cursor: pointer;
  border-radius: 4px;
  padding: 2px 6px;
  font-size: 14px;
  line-height: 1;
  text-decoration: none;
}

.open-ext:hover {
  background: #e5e6eb;
  color: #1f2329;
}

.snapback:hover {
  background: #e0eaff;
  color: #2f6fe0;
}

.close:hover {
  background: #ffe0e0;
  color: #e5484d;
}

.frame-wrap {
  flex: 1;
  position: relative;
  background: #fff;
  min-height: 0;
}

.frame-wrap iframe {
  width: 100%;
  height: 100%;
  border: 0;
  display: block;
}

.mask {
  position: absolute;
  inset: 0;
  background: transparent;
  cursor: inherit;
}

.resize-handle {
  position: absolute;
  right: 0;
  bottom: 0;
  width: 16px;
  height: 16px;
  cursor: nwse-resize;
  background: linear-gradient(
    135deg,
    transparent 50%,
    #8a9099 50%,
    #8a9099 60%,
    transparent 60%,
    transparent 70%,
    #8a9099 70%,
    #8a9099 80%,
    transparent 80%
  );
  touch-action: none;
}
</style>
