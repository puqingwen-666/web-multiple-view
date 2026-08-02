<script setup>
import { computed, ref, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  tree: { type: Object, required: true },
  activeId: { type: Number, default: null },
  open: { type: Boolean, default: false }
})

const emit = defineEmits(['select', 'add-root', 'add-child', 'close', 'toggle'])

const STORAGE_KEY = 'cw_fab_pos'

const pos = ref({ x: null, y: null })
const dragging = ref(false)
let dragStart = null
let pointerStart = null
let movedDuringPress = false

const flatNodes = computed(() => flatten(props.tree))

function flatten(node, depth = 0, acc = []) {
  if (!node) return acc
  acc.push({ id: node.id, title: node.title || node.url, url: node.url, depth, parentId: node.parentId ?? null })
  for (const child of node.children || []) {
    flatten(child, depth + 1, acc)
  }
  return acc
}

function clampPos(x, y) {
  const size = 56
  const margin = 6
  const maxX = window.innerWidth - size - margin
  const maxY = window.innerHeight - size - margin
  return {
    x: Math.max(margin, Math.min(maxX, x)),
    y: Math.max(margin, Math.min(maxY, y))
  }
}

function onPointerDown(e) {
  if (e.button !== 0 && e.pointerType === 'mouse') return
  const target = e.currentTarget
  try { target.setPointerCapture(e.pointerId) } catch (_) {}
  const rect = target.getBoundingClientRect()
  const startX = rect.left
  const startY = rect.top
  pointerStart = { x: e.clientX, y: e.clientY }
  dragStart = { x: startX, y: startY }
  movedDuringPress = false
  dragging.value = false

  function onMove(ev) {
    const dx = ev.clientX - pointerStart.x
    const dy = ev.clientY - pointerStart.y
    if (!dragging.value) {
      if (Math.abs(dx) < 6 && Math.abs(dy) < 6) return
      dragging.value = true
      movedDuringPress = true
    }
    pos.value = clampPos(dragStart.x + dx, dragStart.y + dy)
  }

  function onUp(ev) {
    try { target.releasePointerCapture(ev.pointerId) } catch (_) {}
    document.removeEventListener('pointermove', onMove)
    document.removeEventListener('pointerup', onUp)
    document.removeEventListener('pointercancel', onUp)
    if (dragging.value) {
      dragging.value = false
      localStorage.setItem(STORAGE_KEY, JSON.stringify(pos.value))
    } else if (!movedDuringPress) {
      emit('toggle')
    }
  }

  document.addEventListener('pointermove', onMove)
  document.addEventListener('pointerup', onUp)
  document.addEventListener('pointercancel', onUp)
  e.preventDefault()
}

const fabStyle = computed(() => {
  if (pos.value.x === null) return {}
  return {
    right: 'auto',
    bottom: 'auto',
    left: pos.value.x + 'px',
    top: pos.value.y + 'px'
  }
})

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      const p = JSON.parse(saved)
      if (typeof p.x === 'number' && typeof p.y === 'number') {
        pos.value = clampPos(p.x, p.y)
      }
    } catch (_) {}
  }
})

onBeforeUnmount(() => {})
</script>

<template>
  <div class="fab-root">
    <button
      class="fab-btn"
      :style="fabStyle"
      @pointerdown="onPointerDown"
      title="窗口列表"
    >+</button>

    <div v-if="props.open" class="fab-mask" @click="emit('toggle')">
      <div class="fab-panel" @click.stop>
        <header class="fab-head">
          <span>窗口树</span>
          <button class="head-add" @click="emit('add-root')">新建主窗口</button>
        </header>

        <ul class="tree-list">
          <li
            v-for="node in flatNodes"
            :key="node.id"
            :class="{ active: node.id === props.activeId }"
            :style="{ paddingLeft: 16 + node.depth * 18 + 'px' }"
          >
            <button class="row" @click="emit('select', node.id)">
              <span class="dot" :class="node.depth === 0 ? 'root' : 'child'"></span>
              <span class="title">{{ node.title }}</span>
            </button>
            <button class="add-child" @click.stop="emit('add-child', node.id)" title="新建子窗口">＋</button>
            <button class="close" @click.stop="emit('close', node.id)" title="关闭">×</button>
          </li>
          <li v-if="flatNodes.length === 0" class="empty">
            还没有窗口<br />点上方「新建主窗口」开始
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped>
.fab-root {
  position: fixed;
  z-index: 9999;
}

.fab-btn {
  position: fixed;
  right: 18px;
  bottom: 18px;
  width: 56px;
  height: 56px;
  border: 0;
  border-radius: 50%;
  background: #3b82f6;
  color: #fff;
  font-size: 30px;
  line-height: 1;
  cursor: pointer;
  box-shadow: 0 6px 18px rgba(59, 130, 246, 0.45);
  z-index: 9999;
  touch-action: none;
  user-select: none;
  -webkit-user-select: none;
}

.fab-btn:active {
  background: #2f6fe0;
}

.fab-mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  z-index: 9998;
}

.fab-panel {
  position: fixed;
  right: 12px;
  bottom: 84px;
  left: 12px;
  max-height: 70vh;
  display: flex;
  flex-direction: column;
  background: #fff;
  border-radius: 14px;
  box-shadow: 0 12px 36px rgba(0, 0, 0, 0.25);
  overflow: hidden;
  z-index: 9999;
}

.fab-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 16px;
  border-bottom: 1px solid #f0f1f3;
  font-size: 15px;
  font-weight: 600;
}

.head-add {
  border: 0;
  background: #3b82f6;
  color: #fff;
  font-size: 13px;
  padding: 6px 12px;
  border-radius: 6px;
  cursor: pointer;
}

.head-add:active {
  background: #2f6fe0;
}

.tree-list {
  list-style: none;
  margin: 0;
  padding: 6px 0;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}

.tree-list li {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 10px 12px;
  border-bottom: 1px solid #f5f6f7;
}

.tree-list li.active {
  background: #eef4ff;
}

.tree-list li.active .title {
  color: #2f6fe0;
  font-weight: 600;
}

.row {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 8px;
  border: 0;
  background: transparent;
  padding: 0;
  text-align: left;
  cursor: pointer;
  min-width: 0;
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}

.dot.root {
  background: #3b82f6;
}

.dot.child {
  background: #c5c8ce;
}

.title {
  flex: 1;
  font-size: 14px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.add-child,
.close {
  border: 0;
  background: transparent;
  color: #8a9099;
  font-size: 18px;
  line-height: 1;
  padding: 4px 8px;
  border-radius: 6px;
  cursor: pointer;
}

.add-child:active {
  background: #e0eaff;
  color: #2f6fe0;
}

.close:active {
  background: #ffe0e0;
  color: #e5484d;
}

.empty {
  display: block;
  text-align: center;
  color: #8a9099;
  font-size: 13px;
  padding: 24px 16px;
  line-height: 1.7;
}
</style>
