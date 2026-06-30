<template>
  <div class="overlay-settings">
    <label class="row">
      <input type="checkbox" v-model="colorEnabled" />
      Drawing color
      <input type="color" v-model="color" :disabled="!colorEnabled" class="color-input" />
    </label>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue'
import type { OverlayOptions } from './PdfViewer.vue'

const emit = defineEmits<{ change: [options: OverlayOptions] }>()

const STORAGE_KEY = 'annodoc-overlay-settings'

function load() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    return raw ? JSON.parse(raw) as { colorEnabled: boolean; color: string } : null
  } catch { return null }
}

const saved = load()
const colorEnabled = ref(saved?.colorEnabled ?? false)
const color = ref(saved?.color ?? '#ff0000')

watch([colorEnabled, color], ([enabled, c]) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify({ colorEnabled: enabled, color: c }))
  emit('change', { drawingColor: enabled ? c : null })
}, { immediate: true })
</script>

<style scoped>
.overlay-settings {
  display: flex;
  align-items: center;
  justify-content: center;
}

.row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
}

.color-input {
  width: 36px;
  height: 24px;
  padding: 0;
  border: 1px solid #ccc;
  border-radius: 3px;
  cursor: pointer;
}

.color-input:disabled {
  opacity: 0.4;
  cursor: default;
}
</style>
