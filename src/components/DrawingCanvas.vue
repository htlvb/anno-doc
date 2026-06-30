<template>
  <div class="drawing-canvas-wrapper">
    <canvas
      ref="canvas"
      @mousedown="startDraw"
      @mousemove="draw"
      @mouseup="stopDraw"
      @mouseleave="stopDraw"
      @touchstart.prevent="startDrawTouch"
      @touchmove.prevent="drawTouch"
      @touchend="stopDraw"
    />
    <div class="actions">
      <button @click="clear">Clear</button>
      <button @click="add">Add</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const emit = defineEmits<{ add: [dataUrl: string] }>()

const LOGICAL_W = 700
const LOGICAL_H = 350
const PADDING = 6

const canvas = ref<HTMLCanvasElement | null>(null)
let ctx: CanvasRenderingContext2D | null = null
let drawing = false

onMounted(() => {
  const el = canvas.value!
  const dpr = window.devicePixelRatio || 1
  el.width = LOGICAL_W * dpr
  el.height = LOGICAL_H * dpr
  el.style.width = `${LOGICAL_W}px`
  el.style.height = `${LOGICAL_H}px`
  ctx = el.getContext('2d')!
  ctx.scale(dpr, dpr)
  ctx.strokeStyle = '#000'
  ctx.lineWidth = 2
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'
})

function getPos(e: MouseEvent) {
  const rect = canvas.value!.getBoundingClientRect()
  return { x: e.clientX - rect.left, y: e.clientY - rect.top }
}

function getTouchPos(e: TouchEvent) {
  const rect = canvas.value!.getBoundingClientRect()
  const touch = e.touches[0]
  return { x: touch.clientX - rect.left, y: touch.clientY - rect.top }
}

function startDraw(e: MouseEvent) {
  drawing = true
  const { x, y } = getPos(e)
  ctx!.beginPath()
  ctx!.moveTo(x, y)
}

function draw(e: MouseEvent) {
  if (!drawing) return
  const { x, y } = getPos(e)
  ctx!.lineTo(x, y)
  ctx!.stroke()
}

function startDrawTouch(e: TouchEvent) {
  drawing = true
  const { x, y } = getTouchPos(e)
  ctx!.beginPath()
  ctx!.moveTo(x, y)
}

function drawTouch(e: TouchEvent) {
  if (!drawing) return
  const { x, y } = getTouchPos(e)
  ctx!.lineTo(x, y)
  ctx!.stroke()
}

function stopDraw() {
  drawing = false
}

function clear() {
  ctx!.clearRect(0, 0, LOGICAL_W, LOGICAL_H)
}

function getBounds() {
  const c = canvas.value!
  const { data } = ctx!.getImageData(0, 0, c.width, c.height)
  let minX = c.width, minY = c.height, maxX = 0, maxY = 0
  for (let y = 0; y < c.height; y++) {
    for (let x = 0; x < c.width; x++) {
      if (data[(y * c.width + x) * 4 + 3] > 0) {
        if (x < minX) minX = x
        if (y < minY) minY = y
        if (x > maxX) maxX = x
        if (y > maxY) maxY = y
      }
    }
  }
  if (minX > maxX || minY > maxY) return null
  return { x: minX, y: minY, w: maxX - minX + 1, h: maxY - minY + 1 }
}

function add() {
  const bounds = getBounds()
  if (!bounds) return
  const c = canvas.value!
  const dpr = window.devicePixelRatio || 1
  const pad = PADDING * dpr
  const x = Math.max(0, bounds.x - pad)
  const y = Math.max(0, bounds.y - pad)
  const w = Math.min(c.width - x, bounds.w + pad * 2)
  const h = Math.min(c.height - y, bounds.h + pad * 2)
  const tmp = document.createElement('canvas')
  tmp.width = w
  tmp.height = h
  tmp.getContext('2d')!.drawImage(c, x, y, w, h, 0, 0, w, h)
  emit('add', tmp.toDataURL())
  clear()
}
</script>

<style scoped>
.drawing-canvas-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.actions {
  display: flex;
  gap: 0.5rem;
}

.actions button {
  padding: 0.25rem 0.75rem;
  cursor: pointer;
}

canvas {
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: crosshair;
  touch-action: none;
  max-width: 100%;
}
</style>
