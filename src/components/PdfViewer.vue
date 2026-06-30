<template>
  <div class="pdf-viewer">
    <div v-if="!pdfDoc" class="upload-area" @dragover.prevent @drop.prevent="onDrop">
      <label class="upload-label">
        <input type="file" accept="application/pdf" @change="onFileChange" />
        <span>Drop a PDF here or click to upload</span>
      </label>
    </div>

    <template v-else>
      <div class="toolbar">
        <button @click="reset">Upload another</button>
        <span class="page-info">{{ currentPage }} / {{ totalPages }}</span>
        <button :disabled="currentPage <= 1" @click="changePage(-1)">&#8249;</button>
        <button :disabled="currentPage >= totalPages" @click="changePage(1)">&#8250;</button>
        <button @click="exportPdf">Export PDF</button>
      </div>
      <div class="canvas-container">
        <canvas
          ref="canvas"
          :style="{ cursor: canvasCursor }"
          @mousedown="onMouseDown"
          @mousemove="onMouseMove"
          @mouseup="onMouseUp"
          @mouseleave="onMouseUp"
          @dblclick="onDblClick"
          @touchstart.prevent="onTouchStart"
          @touchmove.prevent="onTouchMove"
          @touchend="onMouseUp"
        />
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, shallowRef, computed, watch, nextTick } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import type { PDFDocumentProxy } from 'pdfjs-dist'
import { PDFDocument } from 'pdf-lib'

pdfjsLib.GlobalWorkerOptions.workerSrc = new URL(
  'pdfjs-dist/build/pdf.worker.min.mjs',
  import.meta.url,
).toString()

export interface OverlayOptions {
  drawingColor: string | null
}

type Overlay = { img: HTMLImageElement; x: number; y: number; scale: number }
type HandleId = 'tl' | 'tr' | 'bl' | 'br'
type DragState =
  | { kind: 'move'; overlay: Overlay; offsetX: number; offsetY: number }
  | { kind: 'scale'; overlay: Overlay; handle: HandleId; anchorX: number; anchorY: number; startScale: number }

const HANDLE = 10

const canvas = ref<HTMLCanvasElement | null>(null)
const pdfDoc = shallowRef<PDFDocumentProxy | null>(null)
const currentPage = ref(1)
const totalPages = ref(0)
const canvasCursor = ref('default')

const overlays: Overlay[] = []
let pageCache: ImageData | null = null
let drag: DragState | null = null
let activeOverlay: Overlay | null = null
let pdfData: ArrayBuffer | null = null

// ── geometry ────────────────────────────────────────────────────────────────

function overlayW(o: Overlay) { return o.img.naturalWidth * o.scale }
function overlayH(o: Overlay) { return o.img.naturalHeight * o.scale }

function cornerPos(o: Overlay): Record<HandleId, { x: number; y: number }> {
  const w = overlayW(o), h = overlayH(o)
  return {
    tl: { x: o.x,     y: o.y },
    tr: { x: o.x + w, y: o.y },
    bl: { x: o.x,     y: o.y + h },
    br: { x: o.x + w, y: o.y + h },
  }
}

function hitHandle(o: Overlay, x: number, y: number): HandleId | null {
  for (const [id, pos] of Object.entries(cornerPos(o)) as [HandleId, { x: number; y: number }][]) {
    if (Math.abs(x - pos.x) <= HANDLE && Math.abs(y - pos.y) <= HANDLE) return id
  }
  return null
}

function hitOverlay(x: number, y: number): Overlay | null {
  for (let i = overlays.length - 1; i >= 0; i--) {
    const o = overlays[i]
    if (x >= o.x && x <= o.x + overlayW(o) && y >= o.y && y <= o.y + overlayH(o)) return o
  }
  return null
}

function toCanvasCoords(clientX: number, clientY: number) {
  const el = canvas.value!
  const rect = el.getBoundingClientRect()
  return {
    x: (clientX - rect.left) * (el.width / rect.width),
    y: (clientY - rect.top) * (el.height / rect.height),
  }
}

// ── rendering ────────────────────────────────────────────────────────────────

async function renderPage(pageNum: number) {
  if (!pdfDoc.value || !canvas.value) return
  const page = await pdfDoc.value.getPage(pageNum)
  const dpr = window.devicePixelRatio || 1
  const viewport = page.getViewport({ scale: 1.5 * dpr })
  const el = canvas.value
  el.width = viewport.width
  el.height = viewport.height
  el.style.width = `${viewport.width / dpr}px`
  el.style.height = `${viewport.height / dpr}px`
  await page.render({ canvas: el, viewport }).promise
  pageCache = el.getContext('2d')!.getImageData(0, 0, el.width, el.height)
  drawOverlays()
}

function drawOverlays() {
  if (!canvas.value) return
  const ctx = canvas.value.getContext('2d')!
  if (pageCache) ctx.putImageData(pageCache, 0, 0)

  for (const o of overlays)
    ctx.drawImage(o.img, o.x, o.y, overlayW(o), overlayH(o))
  if (activeOverlay) drawHandles(ctx, activeOverlay)
}

function drawHandles(ctx: CanvasRenderingContext2D, o: Overlay) {
  const w = overlayW(o), h = overlayH(o)
  ctx.save()
  ctx.strokeStyle = '#2563eb'
  ctx.lineWidth = 1.5
  ctx.setLineDash([4, 3])
  ctx.strokeRect(o.x, o.y, w, h)
  ctx.restore()

  ctx.fillStyle = '#fff'
  ctx.strokeStyle = '#2563eb'
  ctx.lineWidth = 1.5
  for (const pos of Object.values(cornerPos(o))) {
    ctx.beginPath()
    ctx.rect(pos.x - HANDLE / 2, pos.y - HANDLE / 2, HANDLE, HANDLE)
    ctx.fill()
    ctx.stroke()
  }

}

// ── interaction ──────────────────────────────────────────────────────────────

const HANDLE_CURSORS: Record<HandleId, string> = {
  tl: 'nwse-resize', tr: 'nesw-resize',
  bl: 'nesw-resize', br: 'nwse-resize',
}

function interact(x: number, y: number, down: boolean) {
  // check handles on active overlay first
  if (activeOverlay) {
    const h = hitHandle(activeOverlay, x, y)
    if (h) {
      if (down) {
        const corners = cornerPos(activeOverlay)
        const opposite: Record<HandleId, HandleId> = { tl: 'br', tr: 'bl', bl: 'tr', br: 'tl' }
        const anchor = corners[opposite[h]]
        drag = { kind: 'scale', overlay: activeOverlay, handle: h, anchorX: anchor.x, anchorY: anchor.y, startScale: activeOverlay.scale }
      }
      canvasCursor.value = HANDLE_CURSORS[h]
      return
    }
  }

  const hit = hitOverlay(x, y)
  if (hit) {
    canvasCursor.value = 'move'
    if (down) {
      drag = { kind: 'move', overlay: hit, offsetX: x - hit.x, offsetY: y - hit.y }
      if (activeOverlay !== hit) { activeOverlay = hit; drawOverlays() }
    }
  } else {
    canvasCursor.value = 'default'
    if (down && activeOverlay !== null) { activeOverlay = null; drawOverlays() }
  }
}

function applyDrag(x: number, y: number) {
  if (!drag) return
  if (drag.kind === 'move') {
    drag.overlay.x = x - drag.offsetX
    drag.overlay.y = y - drag.offsetY
  } else {
    const { overlay: o, handle, anchorX, anchorY } = drag
    let newScale: number
    if (handle === 'br' || handle === 'tr') {
      newScale = (x - anchorX) / o.img.naturalWidth
    } else {
      newScale = (anchorX - x) / o.img.naturalWidth
    }
    newScale = Math.max(0.05, newScale)
    o.scale = newScale
    // reposition so anchor corner stays fixed
    const corners = cornerPos(o)
    const opposite: Record<HandleId, HandleId> = { tl: 'br', tr: 'bl', bl: 'tr', br: 'tl' }
    const movedAnchor = corners[opposite[handle]]
    o.x += anchorX - movedAnchor.x
    o.y += anchorY - movedAnchor.y
  }
  drawOverlays()
}

function onMouseDown(e: MouseEvent) {
  const { x, y } = toCanvasCoords(e.clientX, e.clientY)
  interact(x, y, true)
}

function onMouseMove(e: MouseEvent) {
  const { x, y } = toCanvasCoords(e.clientX, e.clientY)
  if (drag) applyDrag(x, y)
  else interact(x, y, false)
}

function onMouseUp() { drag = null }

function onDblClick(e: MouseEvent) {
  const { x, y } = toCanvasCoords(e.clientX, e.clientY)
  const hit = hitOverlay(x, y)
  const i = overlays.lastIndexOf(hit!)
  if (i === -1) return
  if (activeOverlay === overlays[i]) activeOverlay = null
  overlays.splice(i, 1)
  drawOverlays()
}

function onTouchStart(e: TouchEvent) {
  const t = e.touches[0]
  const { x, y } = toCanvasCoords(t.clientX, t.clientY)
  interact(x, y, true)
}

function onTouchMove(e: TouchEvent) {
  if (!drag) return
  const t = e.touches[0]
  const { x, y } = toCanvasCoords(t.clientX, t.clientY)
  applyDrag(x, y)
}

// ── lifecycle ────────────────────────────────────────────────────────────────

watch(currentPage, (p) => { pageCache = null; renderPage(p) })
watch(pdfDoc, async () => {
  pageCache = null
  await nextTick()
  renderPage(1)
})

function onFileChange(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) readFile(file)
}

function onDrop(e: DragEvent) {
  const file = e.dataTransfer?.files[0]
  if (file?.type === 'application/pdf') readFile(file)
}

function readFile(file: File) {
  const reader = new FileReader()
  reader.onload = (e) => {
    pdfData = e.target!.result as ArrayBuffer
    loadPdf(pdfData.slice(0))
  }
  reader.readAsArrayBuffer(file)
}

async function loadPdf(data: ArrayBuffer) {
  pdfDoc.value = await pdfjsLib.getDocument({ data }).promise
  totalPages.value = pdfDoc.value.numPages
  currentPage.value = 1
}

function changePage(delta: number) { currentPage.value += delta }

function reset() {
  pdfDoc.value?.cleanup()
  pdfDoc.value = null
  currentPage.value = 1
  totalPages.value = 0
  pageCache = null
  pdfData = null
  overlays.length = 0
  activeOverlay = null
  drag = null
}

async function exportPdf() {
  if (!pdfData) return
  const doc = await PDFDocument.load(pdfData)
  const pages = doc.getPages()
  const renderScale = 1.5 * (window.devicePixelRatio || 1)

  for (const o of overlays) {
    const pngBytes = await fetch(o.img.src).then((r) => r.arrayBuffer())
    const pdfImage = await doc.embedPng(pngBytes)
    const imgW = overlayW(o) / renderScale
    const imgH = overlayH(o) / renderScale
    const imgX = o.x / renderScale
    for (const page of pages) {
      const imgY = page.getHeight() - o.y / renderScale - imgH
      page.drawImage(pdfImage, { x: imgX, y: imgY, width: imgW, height: imgH })
    }
  }

  const bytes = await doc.save()
  const url = URL.createObjectURL(new Blob([bytes.buffer as ArrayBuffer], { type: 'application/pdf' }))
  const a = document.createElement('a')
  a.href = url
  a.download = 'annotated.pdf'
  a.click()
  URL.revokeObjectURL(url)
}

async function applyDrawingColor(dataUrl: string, color: string | null): Promise<string> {
  if (!color) return dataUrl
  const img = new Image()
  await new Promise<void>((r) => { img.onload = () => r(); img.src = dataUrl })
  const c = document.createElement('canvas')
  c.width = img.naturalWidth; c.height = img.naturalHeight
  const ctx = c.getContext('2d')!
  ctx.drawImage(img, 0, 0)
  ctx.globalCompositeOperation = 'source-in'
  ctx.fillStyle = color
  ctx.fillRect(0, 0, c.width, c.height)
  return c.toDataURL('image/png')
}

async function addOverlay(dataUrl: string, options: OverlayOptions): Promise<void> {
  const coloredUrl = await applyDrawingColor(dataUrl, options.drawingColor)
  const img = new Image()
  img.onload = () => {
    if (!canvas.value) return
    const x = Math.round((canvas.value.width - img.naturalWidth) / 2)
    const y = Math.round((canvas.value.height - img.naturalHeight) / 2)
    overlays.push({ img, x, y, scale: 1 })
    drawOverlays()
  }
  img.src = coloredUrl
}

const hasDocument = computed(() => pdfDoc.value !== null)

defineExpose({ addOverlay, hasDocument })
</script>

<style scoped>
.pdf-viewer {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  padding: 2rem;
}

.upload-area {
  border: 2px dashed #aaa;
  border-radius: 8px;
  padding: 3rem 4rem;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.2s;
}

.upload-area:hover { border-color: #555; }

.upload-label {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  cursor: pointer;
  font-size: 1rem;
  color: #555;
}

.upload-label input { display: none; }

.toolbar {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.toolbar button {
  padding: 0.25rem 0.75rem;
  font-size: 1rem;
  cursor: pointer;
}

.toolbar button:disabled {
  opacity: 0.4;
  cursor: default;
}

.page-info {
  min-width: 5rem;
  text-align: center;
}

.canvas-container { box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15); }

canvas {
  display: block;
  max-width: 100%;
}
</style>
