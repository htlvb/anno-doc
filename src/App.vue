<template>
  <DrawingCanvas @add="drawings.push($event)" />
  <div v-if="drawings.length" class="drawings-section">
    <div class="drawings-list">
      <div v-for="(src, i) in drawings" :key="i" class="drawing-item">
        <img
          :src="src"
          class="drawing-thumb"
          :class="{ selected: selectedIndex === i }"
          @click="selectedIndex = selectedIndex === i ? null : i"
        />
        <button class="delete-btn" @click="deleteDrawing(i)">✕</button>
      </div>
    </div>
    <OverlaySettings v-if="selectedIndex !== null" @change="overlayOptions = $event" />
    <button
      class="add-to-doc-btn"
      :disabled="selectedIndex === null || !pdfViewer?.hasDocument"
      @click="addToDocument"
    >
      Add to document
    </button>
  </div>
  <PdfViewer ref="pdfViewer" />
</template>

<script setup lang="ts">
import { ref, watch } from 'vue'
import DrawingCanvas from './components/DrawingCanvas.vue'
import PdfViewer from './components/PdfViewer.vue'
import OverlaySettings from './components/OverlaySettings.vue'
import type { OverlayOptions } from './components/PdfViewer.vue'

const STORAGE_KEY = 'annodoc-drawings'

function loadDrawings(): string[] {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    return raw ? (JSON.parse(raw) as string[]) : []
  } catch {
    return []
  }
}

const drawings = ref<string[]>(loadDrawings())

watch(drawings, (val) => localStorage.setItem(STORAGE_KEY, JSON.stringify(val)), { deep: true })
const selectedIndex = ref<number | null>(null)
const pdfViewer = ref<InstanceType<typeof PdfViewer> | null>(null)
const overlayOptions = ref<OverlayOptions>({ drawingColor: null })

async function addToDocument() {
  if (selectedIndex.value === null) return
  await pdfViewer.value?.addOverlay(drawings.value[selectedIndex.value], overlayOptions.value)
  selectedIndex.value = null
}

function deleteDrawing(i: number) {
  drawings.value.splice(i, 1)
  if (selectedIndex.value === i) selectedIndex.value = null
  else if (selectedIndex.value !== null && selectedIndex.value > i) selectedIndex.value--
}
</script>

<style scoped>
.drawings-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  padding: 1rem;
}

.drawings-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  justify-content: center;
}

.drawing-item {
  position: relative;
  display: inline-flex;
}

.drawing-item:hover .delete-btn {
  display: flex;
}

.delete-btn {
  display: none;
  position: absolute;
  top: -6px;
  right: -6px;
  width: 18px;
  height: 18px;
  align-items: center;
  justify-content: center;
  padding: 0;
  font-size: 10px;
  line-height: 1;
  border-radius: 50%;
  border: 1px solid #aaa;
  background: #fff;
  cursor: pointer;
}

.drawing-thumb {
  border: 2px solid #ccc;
  border-radius: 4px;
  max-height: 120px;
  cursor: pointer;
  transition: border-color 0.15s;
}

.drawing-thumb:hover {
  border-color: #888;
}

.drawing-thumb.selected {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px #93c5fd;
}

.add-to-doc-btn {
  padding: 0.3rem 1rem;
  cursor: pointer;
}

.add-to-doc-btn:disabled {
  opacity: 0.4;
  cursor: default;
}
</style>
