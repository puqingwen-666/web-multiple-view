<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  presets: { type: Array, default: () => [] },
  windows: { type: Array, default: () => [] }
})

const emit = defineEmits(['add', 'focus', 'close', 'start-drag', 'snapback'])

const showForm = ref(false)
const customUrl = ref('')
const customTitle = ref('')

const snappedWindows = computed(() => props.windows.filter((w) => w.mode === 'snapped'))

function addPreset(p) {
  emit('add', { title: p.title, url: p.url })
}

function addCustom() {
  let url = customUrl.value.trim()
  if (!url) return
  if (!/^https?:\/\//i.test(url)) url = 'https://' + url
  emit('add', { title: customTitle.value.trim() || url, url })
  customUrl.value = ''
  customTitle.value = ''
  showForm.value = false
}

function onItemContext(e, w) {
  e.preventDefault()
  if (w.mode === 'floating') emit('snapback', w.id)
}

function slotStyle(w, idx, total) {
  if (total === 0) return {}
  const per = 100 / total
  return {
    top: `calc(${idx * per}% + 4px)`,
    height: `calc(${per}% - 8px)`,
    zIndex: w.z
  }
}

function startDrag(e, w) {
  if (e.button !== 0) return
  emit('start-drag', w.id, e)
}
</script>

<template>
  <aside class="right-panel">
    <header class="head">
      <span class="title">右侧窗口</span>
      <button class="add" @click="showForm = !showForm" title="添加嵌入窗口">+</button>
    </header>

    <div v-if="showForm" class="form">
      <input
        v-model="customTitle"
        placeholder="标题（可选）"
        @keydown.enter="addCustom"
      />
      <input
        v-model="customUrl"
        placeholder="网址 https://…"
        @keydown.enter="addCustom"
      />
      <div class="form-actions">
        <button class="primary" @click="addCustom">添加</button>
        <button @click="showForm = false">取消</button>
      </div>
    </div>

    <div class="right-panel-inner">
      <div v-if="windows.length === 0" class="empty">
        点上方 <strong>+</strong> 添加第一个窗口<br />
        第一个窗口将占满整个右侧
      </div>

      <template v-else>
        <div
          v-for="(w, idx) in snappedWindows"
          :key="w.id"
          class="slot snapped"
          :style="slotStyle(w, idx, snappedWindows.length)"
          @pointerdown="emit('focus', w.id)"
          @contextmenu="onItemContext($event, w)"
        >
          <header
            class="bar draggable"
            @pointerdown="startDrag($event, w)"
          >
            <span class="w-title" :title="w.title">{{ w.title }}</span>
            <a
              class="open-ext"
              :href="w.url"
              target="_blank"
              rel="noopener"
              @click.stop
              @pointerdown.stop
              title="新标签打开"
            >↗</a>
            <button class="close" @click.stop="emit('close', w.id)" @pointerdown.stop title="关闭">×</button>
          </header>

          <div class="frame-wrap">
            <iframe
              :src="w.url"
              frameborder="0"
              allow="fullscreen"
              sandbox="allow-same-origin allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-downloads"
              referrerpolicy="no-referrer"
            ></iframe>
          </div>
        </div>
      </template>
    </div>
  </aside>
</template>

<style scoped>
.right-panel {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #ffffff;
  border-left: 1px solid #e5e6eb;
  overflow: hidden;
  position: relative;
}

.head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  border-bottom: 1px solid #e5e6eb;
  background: #fafbfc;
  flex-shrink: 0;
}

.title {
  font-weight: 600;
  font-size: 13px;
}

.add {
  width: 28px;
  height: 28px;
  border: 0;
  border-radius: 6px;
  background: #3b82f6;
  color: #fff;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
}

.add:hover {
  background: #2f6fe0;
}

.form {
  padding: 10px 14px;
  border-bottom: 1px solid #e5e6eb;
  display: flex;
  flex-direction: column;
  gap: 6px;
  background: #fafbfc;
  flex-shrink: 0;
}

.form input {
  height: 30px;
  padding: 0 10px;
  border: 1px solid #d8dce2;
  border-radius: 6px;
  font-size: 13px;
  font-family: inherit;
  outline: none;
}

.form input:focus {
  border-color: #3b82f6;
}

.form-actions {
  display: flex;
  gap: 6px;
}

.form-actions button {
  flex: 1;
  height: 30px;
  border: 1px solid #d8dce2;
  background: #fff;
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
}

.form-actions button.primary {
  background: #3b82f6;
  color: #fff;
  border-color: #3b82f6;
}

.right-panel-inner {
  flex: 1;
  position: relative;
  overflow: visible;
  min-height: 0;
}

.empty {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  color: #8a9099;
  font-size: 12px;
  line-height: 1.7;
  padding: 16px;
}

.slot {
  position: absolute;
  left: 6px;
  right: 6px;
  display: flex;
  flex-direction: column;
  border: 1px solid #d8dce2;
  border-radius: 6px;
  overflow: hidden;
  background: #fff;
  box-sizing: border-box;
}

.bar {
  height: 30px;
  display: flex;
  align-items: center;
  padding: 0 8px 0 10px;
  background: #f5f6f7;
  border-bottom: 1px solid #e5e6eb;
  user-select: none;
  gap: 4px;
  flex-shrink: 0;
  touch-action: none;
}

.bar.draggable {
  cursor: grab;
}

.w-title {
  flex: 1;
  font-size: 12px;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.open-ext,
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
</style>
