<script setup>
import { ref } from 'vue'

const props = defineProps({
  url: { type: String, default: '' },
  input: { type: String, default: '' },
  placeholderShown: { type: Boolean, default: true },
  iframeKey: { type: Number, default: 0 },
  history: { type: Array, default: () => [] }
})

const emit = defineEmits(['update:input', 'load', 'clear', 'pick-history'])

const focused = ref(false)

function onFocus() {
  focused.value = true
}
function onBlur() {
  setTimeout(() => { focused.value = false }, 200)
}
function pick(url) {
  emit('pick-history', url)
  focused.value = false
}
</script>

<template>
  <section class="main-window">
    <header class="header">
      <div class="url-bar">
        <div class="input-wrap">
          <input
            class="url-input"
            :value="input"
            @input="emit('update:input', $event.target.value)"
            @keydown.enter="emit('load')"
            @focus="onFocus"
            @blur="onBlur"
            placeholder="输入网址，如 https://example.com"
            spellcheck="false"
          />
          <ul v-if="focused && history.length" class="history-list">
            <li
              v-for="item in history"
              :key="item"
              @mousedown.prevent="pick(item)"
              :title="item"
            >{{ item }}</li>
          </ul>
        </div>
        <button class="btn primary" @click="emit('load')" :disabled="!input">加载</button>
        <button class="btn" @click="emit('clear')" :disabled="!url && !input">清空</button>
      </div>
    </header>

    <div class="body">
      <div v-if="placeholderShown" class="placeholder">
        <div class="ph-card">
          <div class="ph-title">主窗口</div>
          <div class="ph-desc">
            在上方输入网址并点「加载」，<br />
            页面将嵌入此处。
          </div>
        </div>
      </div>
      <iframe
        v-else
        :key="iframeKey"
        :src="url"
        class="frame"
        frameborder="0"
        allow="fullscreen"
        sandbox="allow-same-origin allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-downloads"
        referrerpolicy="no-referrer"
      ></iframe>
    </div>
  </section>
</template>

<style scoped>
.main-window {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #ffffff;
  border-right: 1px solid #e5e6eb;
  min-width: 0;
}

.header {
  padding: 10px 12px;
  border-bottom: 1px solid #e5e6eb;
  background: #fafbfc;
}

.url-bar {
  display: flex;
  gap: 6px;
  align-items: flex-start;
}

.input-wrap {
  flex: 1;
  position: relative;
  min-width: 0;
}

.url-input {
  width: 100%;
  height: 32px;
  padding: 0 10px;
  border: 1px solid #d8dce2;
  border-radius: 6px;
  font-size: 13px;
  font-family: inherit;
  outline: none;
  background: #fff;
  box-sizing: border-box;
}

.url-input:focus {
  border-color: #3b82f6;
}

.history-list {
  position: absolute;
  top: 34px;
  left: 0;
  right: 0;
  z-index: 100;
  list-style: none;
  margin: 0;
  padding: 4px 0;
  background: #fff;
  border: 1px solid #d8dce2;
  border-radius: 6px;
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.12);
  max-height: 280px;
  overflow-y: auto;
}

.history-list li {
  padding: 6px 12px;
  font-size: 12px;
  color: #4b5159;
  cursor: pointer;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.history-list li:hover {
  background: #eef4ff;
  color: #2f6fe0;
}

.btn {
  height: 32px;
  padding: 0 12px;
  border: 1px solid #d8dce2;
  background: #fff;
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
  color: #4b5159;
  flex-shrink: 0;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn.primary {
  background: #3b82f6;
  color: #fff;
  border-color: #3b82f6;
}

.btn.primary:disabled {
  background: #c5c8ce;
  border-color: #c5c8ce;
}

.body {
  flex: 1;
  position: relative;
  background: #fff;
  min-height: 0;
}

.frame {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  border: 0;
  display: block;
}

.placeholder {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f5f6f7;
}

.ph-card {
  text-align: center;
  padding: 32px 48px;
  border: 1px dashed #c5c8ce;
  border-radius: 8px;
  background: #fff;
}

.ph-title {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 8px;
  color: #1f2329;
}

.ph-desc {
  font-size: 13px;
  color: #8a9099;
  line-height: 1.6;
}
</style>
