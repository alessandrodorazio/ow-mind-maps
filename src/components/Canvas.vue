<template>
  <div class="mindmap-layout" @click="deselectRelation">
    <aside class="sidebar">
      <h2>Concepts</h2>
      <ul>
        <li
          v-for="concept in concepts"
          :key="concept.id"
          @click="teleportToConcept(concept.id)"
          style="cursor: pointer"
        >
          <strong>{{ concept.title }}</strong>
          <br />
          <span class="coords">({{ concept.x.toFixed(0) }}, {{ concept.y.toFixed(0) }})</span>
        </li>
      </ul>
    </aside>
    <div
      ref="viewportRef"
      class="viewport"
      :style="viewportStyle"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
      @mouseleave="onMouseUp"
      @wheel.prevent="onWheel"
      @mousemove.capture="updateMouseCoords"
      @click.stop
    >
      <!-- SVG for relations -->
      <svg
        class="relations-svg"
        :width="'100%'"
        :height="'100%'"
        style="position: absolute; left: 0; top: 0; pointer-events: none; z-index: 2"
      >
        <line
          v-for="relation in relations"
          :key="relation.id"
          :x1="getRelationEndpoints(relation.from, relation.to).start.x"
          :y1="getRelationEndpoints(relation.from, relation.to).start.y"
          :x2="getRelationEndpoints(relation.from, relation.to).end.x"
          :y2="getRelationEndpoints(relation.from, relation.to).end.y"
          :stroke="selectedRelationId === relation.id ? '#e53935' : '#1976d2'"
          :stroke-width="selectedRelationId === relation.id ? 4 : 2.5"
          marker-end="url(#arrowhead)"
          style="cursor: pointer; pointer-events: stroke"
          @click.stop="selectRelation(relation.id)"
        />
        <defs>
          <marker
            id="arrowhead"
            markerWidth="8"
            markerHeight="8"
            refX="8"
            refY="4"
            orient="auto"
            markerUnits="strokeWidth"
          >
            <polygon points="0 0, 8 4, 0 8" :fill="selectedRelationId ? '#e53935' : '#1976d2'" />
          </marker>
        </defs>
      </svg>
      <div class="world" @dblclick="onWorldDblClick" @click="handleWorldClick">
        <div class="world-content" :style="worldStyle">
          <MindMapConcept
            v-for="concept in concepts"
            :key="concept.id"
            :id="concept.id"
            :x="concept.x"
            :y="concept.y"
            :title="concept.title"
            :selected="selectedConceptId === concept.id"
            :size="concept.size || 'medium'"
            @update:title="updateConceptTitle(concept.id, $event)"
            @drag="dragConcept(concept.id, $event)"
            @dragStart="onConceptDragStart"
            @dragEnd="onConceptDragEnd"
            @click="handleConceptClick(concept.id)"
            @contextmenu.prevent="showConceptContextMenu($event, concept.id)"
            :ref="(el) => setConceptRef(concept.id, el)"
          />
        </div>
        <div
          v-if="contextMenu.visible"
          class="context-menu"
          :style="{ left: contextMenu.x + 'px', top: contextMenu.y + 'px' }"
        >
          <ul>
            <li v-if="contextMenu.conceptId !== null" @click="deleteConcept(contextMenu.conceptId)">
              Delete
            </li>
          </ul>
        </div>
      </div>
      <div class="coords-overlay">
        ({{ mouseWorld.x.toFixed(0) }}, {{ mouseWorld.y.toFixed(0) }})
      </div>
    </div>
    <aside class="rightbar">
      <template v-if="selectedConcept">
        <label for="title">Title:</label>
        <input
          id="title"
          v-model="selectedConcept.title"
          class="title-input"
          @input="updateConceptTitle(selectedConcept.id, selectedConcept.title)"
          maxlength="64"
          autocomplete="off"
        />
        <label for="size" style="margin-top: 12px">Size:</label>
        <select
          id="size"
          v-model="selectedConcept.size"
          class="size-select"
          @change="
            updateConceptSize(
              selectedConcept.id,
              selectedConcept.size as 'small' | 'medium' | 'large',
            )
          "
        >
          <option value="small">Small</option>
          <option value="medium">Medium</option>
          <option value="large">Large</option>
        </select>
        <label for="bullets" style="margin-top: 18px">Bullet Points (Markdown):</label>
        <textarea
          id="bullets"
          v-model="selectedConcept.bullets"
          class="bullets-input"
          placeholder="- Bullet 1\n- Bullet 2"
          @input="updateConceptBullets(selectedConcept.id, selectedConcept.bullets)"
        ></textarea>
        <div class="relations-list">
          <h3>Relations</h3>
          <ul>
            <li v-for="rel in getConceptRelations(selectedConcept.id)" :key="rel.id">
              <span v-if="rel.from === selectedConcept.id">→ {{ getConceptTitle(rel.to) }}</span>
              <span v-else>← {{ getConceptTitle(rel.from) }}</span>
            </li>
          </ul>
        </div>
      </template>
      <template v-else>
        <div class="placeholder">Select a concept to edit bullet points.</div>
      </template>
    </aside>
    <div class="bottombar">
      <button :class="{ active: relationMode }" @click="startRelationMode">Relation</button>
      <button @click="exportJSON">Export JSON</button>
      <label class="import-label">
        Import JSON
        <input type="file" accept="application/json" @change="importJSON" style="display: none" />
      </label>
      <span v-if="relationMode" class="relation-hint">
        <template v-if="relationStartConceptId === null">
          Click a concept to start a relation
        </template>
        <template v-else>
          Now click a concept to link from <b>{{ getConceptTitle(relationStartConceptId) }}</b>
        </template>
      </span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, defineOptions, watch, nextTick, onBeforeUnmount } from 'vue'
import MindMapConcept from './Concept.vue'

defineOptions({ name: 'MindMapCanvas' })

const LOCAL_STORAGE_KEY = 'mindmap-concepts'
const LOCAL_STORAGE_OFFSET_KEY = 'mindmap-offset'
const LOCAL_STORAGE_SCALE_KEY = 'mindmap-scale'
const LOCAL_STORAGE_RELATIONS_KEY = 'mindmap-relations'

const scale = ref(1)
const offset = ref({ x: 0, y: 0 })
const isPanning = ref(false)
const isDraggingConcept = ref(false)
const lastMouse = ref({ x: 0, y: 0 })
const viewportRef = ref<HTMLElement | null>(null)

const mouseWorld = ref({ x: 0, y: 0 })

// Concepts state
interface Concept {
  id: number
  x: number
  y: number
  title: string
  bullets: string
  size?: 'small' | 'medium' | 'large'
}
const concepts = ref<Concept[]>([])
let nextId = 1

const selectedConceptId = ref<number | null>(null)
const selectedConcept = computed(
  () => concepts.value.find((c) => c.id === selectedConceptId.value) || null,
)

const relationMode = ref(false)
const relationStartConceptId = ref<number | null>(null)

interface Relation {
  id: number
  from: number
  to: number
}
const relations = ref<Relation[]>([])
let nextRelationId = 1

const conceptRefs = ref<Record<number, HTMLElement | null>>({})
const conceptRects = ref<Record<number, DOMRect>>({})

const selectedRelationId = ref<number | null>(null)

const contextMenu = ref({ visible: false, x: 0, y: 0, conceptId: null as number | null })

function setConceptRef(id: number, el: Element | { $el: Element } | null) {
  // If el is a Vue component instance, get its $el
  if (el && '$el' in el) {
    conceptRefs.value[id] = (el as { $el: Element }).$el as HTMLElement
  } else {
    conceptRefs.value[id] = el as HTMLElement | null
  }
}

function updateConceptRects() {
  const rects: Record<number, DOMRect> = {}
  for (const id in conceptRefs.value) {
    const el = conceptRefs.value[Number(id)]
    if (el) {
      rects[Number(id)] = el.getBoundingClientRect()
    }
  }
  conceptRects.value = rects
}

watch(
  [concepts, offset, scale],
  () => {
    nextTick(updateConceptRects)
  },
  { deep: true },
)

onMounted(() => {
  // Load from localStorage
  const saved = localStorage.getItem(LOCAL_STORAGE_KEY)
  if (saved) {
    try {
      const arr = JSON.parse(saved)
      if (Array.isArray(arr)) {
        concepts.value = arr
          .filter(
            (c: Partial<Concept>): c is Concept =>
              typeof c.id === 'number' &&
              typeof c.x === 'number' &&
              typeof c.y === 'number' &&
              typeof c.title === 'string',
          )
          .map((c) => ({ ...c, bullets: c.bullets ?? '', size: c.size ?? 'medium' }))
        // Set nextId to max existing id + 1
        nextId = arr.reduce((max, c) => Math.max(max, c.id), 0) + 1
      }
    } catch {}
  }
  // Load offset and scale
  const savedOffset = localStorage.getItem(LOCAL_STORAGE_OFFSET_KEY)
  if (savedOffset) {
    try {
      const o = JSON.parse(savedOffset)
      if (typeof o.x === 'number' && typeof o.y === 'number') {
        offset.value = o
      }
    } catch {}
  } else if (viewportRef.value) {
    // Default center
    const vw = viewportRef.value.clientWidth
    const vh = viewportRef.value.clientHeight
    offset.value.x = vw / 2
    offset.value.y = vh / 2
  }
  const savedScale = localStorage.getItem(LOCAL_STORAGE_SCALE_KEY)
  if (savedScale) {
    const s = Number(savedScale)
    if (!isNaN(s) && s > 0) scale.value = s
  }
  // Load relations
  const savedRelations = localStorage.getItem(LOCAL_STORAGE_RELATIONS_KEY)
  if (savedRelations) {
    try {
      const arr = JSON.parse(savedRelations)
      if (Array.isArray(arr)) {
        relations.value = arr.filter(
          (r: Partial<Relation>): r is Relation =>
            typeof r.id === 'number' && typeof r.from === 'number' && typeof r.to === 'number',
        )
        nextRelationId = relations.value.reduce((max, r) => Math.max(max, r.id), 0) + 1
      }
    } catch {}
  }
  nextTick(updateConceptRects)
  window.addEventListener('keydown', onKeydown)
})

onBeforeUnmount(() => {
  localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(concepts.value))
  localStorage.setItem(LOCAL_STORAGE_OFFSET_KEY, JSON.stringify(offset.value))
  localStorage.setItem(LOCAL_STORAGE_SCALE_KEY, String(scale.value))
  localStorage.setItem(LOCAL_STORAGE_RELATIONS_KEY, JSON.stringify(relations.value))
  window.removeEventListener('keydown', onKeydown)
})

// Save to localStorage whenever concepts change
watch(
  concepts,
  (val) => {
    localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(val))
  },
  { deep: true },
)
// Save offset and scale
watch(
  offset,
  (val) => {
    localStorage.setItem(LOCAL_STORAGE_OFFSET_KEY, JSON.stringify(val))
  },
  { deep: true },
)
watch(scale, (val) => {
  localStorage.setItem(LOCAL_STORAGE_SCALE_KEY, String(val))
})
watch(
  relations,
  (val) => {
    localStorage.setItem(LOCAL_STORAGE_RELATIONS_KEY, JSON.stringify(val))
  },
  { deep: true },
)

const gridSize = 32
const viewportStyle = computed(() => {
  // The grid should move with the world offset and scale
  const bgSize = `${gridSize * scale.value}px ${gridSize * scale.value}px`
  const bgPosX = `${offset.value.x % (gridSize * scale.value)}px`
  const bgPosY = `${offset.value.y % (gridSize * scale.value)}px`
  return {
    backgroundColor: '#e9e9f7',
    backgroundImage:
      'linear-gradient(to right, #d3d3e7 1px, transparent 1px), linear-gradient(to bottom, #d3d3e7 1px, transparent 1px)',
    backgroundSize: bgSize,
    backgroundPosition: `${bgPosX} ${bgPosY}`,
    userSelect: 'none' as const,
    cursor: isPanning.value ? 'grabbing' : 'grab',
    position: 'relative' as const,
    width: '100%',
    height: '100vh',
    overflow: 'hidden',
    flex: '1 1 0',
  }
})

const worldStyle = computed(() => ({
  transform: `translate(${offset.value.x}px, ${offset.value.y}px) scale(${scale.value})`,
  width: '100vw',
  height: '100vh',
  boxSizing: 'border-box' as const,
  position: 'absolute' as const,
  left: 0,
  top: 0,
}))

function onMouseDown(e: MouseEvent) {
  if (isDraggingConcept.value) return
  isPanning.value = true
  lastMouse.value = { x: e.clientX, y: e.clientY }
}

function onMouseMove(e: MouseEvent) {
  if (isDraggingConcept.value) return
  if (!isPanning.value) return
  const dx = e.clientX - lastMouse.value.x
  const dy = e.clientY - lastMouse.value.y
  offset.value.x += dx
  offset.value.y += dy
  lastMouse.value = { x: e.clientX, y: e.clientY }
}

function onMouseUp() {
  if (isDraggingConcept.value) return
  isPanning.value = false
}

function onWheel(e: WheelEvent) {
  const zoomIntensity = 0.03
  const rect = viewportRef.value?.getBoundingClientRect()
  if (!rect) return
  const mouseX = e.clientX - rect.left
  const mouseY = e.clientY - rect.top

  // Convert mouse position to world coordinates before zoom
  const worldX = (mouseX - offset.value.x) / scale.value
  const worldY = (mouseY - offset.value.y) / scale.value

  let newScale = scale.value
  if (e.deltaY < 0) {
    newScale = Math.min(scale.value * (1 + zoomIntensity), 3)
  } else {
    newScale = Math.max(scale.value * (1 - zoomIntensity), 0.2)
  }

  // After zoom, adjust offset so the world point under the cursor stays under the cursor
  offset.value.x = mouseX - worldX * newScale
  offset.value.y = mouseY - worldY * newScale
  scale.value = newScale
}

function updateMouseCoords(e: MouseEvent) {
  const rect = viewportRef.value?.getBoundingClientRect()
  if (!rect) return
  const mouseX = e.clientX - rect.left
  const mouseY = e.clientY - rect.top
  mouseWorld.value.x = (mouseX - offset.value.x) / scale.value
  mouseWorld.value.y = (mouseY - offset.value.y) / scale.value
}

function onWorldDblClick(e: MouseEvent) {
  // Get mouse position relative to the viewport
  const rect = viewportRef.value?.getBoundingClientRect()
  if (!rect) return
  const mouseX = e.clientX - rect.left
  const mouseY = e.clientY - rect.top
  // Convert to world coordinates
  const worldX = (mouseX - offset.value.x) / scale.value
  const worldY = (mouseY - offset.value.y) / scale.value
  concepts.value.push({
    id: nextId++,
    x: worldX,
    y: worldY,
    title: 'Concept',
    bullets: '',
    size: 'medium',
  })
}

function updateConceptTitle(id: number, newTitle: string) {
  const c = concepts.value.find((c) => c.id === id)
  if (c) c.title = newTitle
}

function updateConceptBullets(id: number, newBullets: string) {
  const c = concepts.value.find((c) => c.id === id)
  if (c) c.bullets = newBullets
}

function updateConceptSize(id: number, newSize: 'small' | 'medium' | 'large') {
  const c = concepts.value.find((c) => c.id === id)
  if (c) c.size = newSize
}

function dragConcept(id: number, pos: { x: number; y: number }) {
  const c = concepts.value.find((c) => c.id === id)
  if (c) {
    c.x = pos.x
    c.y = pos.y
  }
}

function onConceptDragStart() {
  isDraggingConcept.value = true
}

function onConceptDragEnd() {
  isDraggingConcept.value = false
}

function startRelationMode() {
  relationMode.value = true
  if (selectedConceptId.value !== null) {
    relationStartConceptId.value = selectedConceptId.value
  } else {
    relationStartConceptId.value = null
  }
}

function handleConceptClick(id: number) {
  if (relationMode.value) {
    if (relationStartConceptId.value === null) {
      relationStartConceptId.value = id
    } else if (relationStartConceptId.value !== id) {
      // Create relation
      relations.value.push({
        id: nextRelationId++,
        from: relationStartConceptId.value,
        to: id,
      })
      relationMode.value = false
      relationStartConceptId.value = null
    }
  } else {
    selectConcept(id)
  }
}

function getConceptTitle(id: number | null) {
  const c = concepts.value.find((c) => c.id === id)
  return c ? c.title : ''
}

function getRectLineIntersection(
  cx: number,
  cy: number,
  w: number,
  h: number,
  tx: number,
  ty: number,
) {
  // Rectangle center (cx, cy), width w, height h
  // Target point (tx, ty)
  // Returns intersection point on rectangle border in direction of (tx, ty)
  const dx = tx - cx
  const dy = ty - cy
  const hw = w / 2
  const hh = h / 2
  let tMin = Infinity
  let ix = cx,
    iy = cy
  // Sides: left, right, top, bottom
  const sides = [
    { x: cx - hw, y1: cy - hh, y2: cy + hh }, // left
    { x: cx + hw, y1: cy - hh, y2: cy + hh }, // right
    { y: cy - hh, x1: cx - hw, x2: cx + hw }, // top
    { y: cy + hh, x1: cx - hw, x2: cx + hw }, // bottom
  ]
  // Left/right
  if (dx !== 0) {
    for (const s of [sides[0], sides[1]]) {
      if (typeof s.x === 'number' && typeof s.y1 === 'number' && typeof s.y2 === 'number') {
        const t = (s.x - cx) / dx
        if (t > 0) {
          const y = cy + t * dy
          if (y >= s.y1 && y <= s.y2 && t < tMin) {
            tMin = t
            ix = s.x
            iy = y
          }
        }
      }
    }
  }
  // Top/bottom
  if (dy !== 0) {
    for (const s of [sides[2], sides[3]]) {
      if (typeof s.y === 'number' && typeof s.x1 === 'number' && typeof s.x2 === 'number') {
        const t = (s.y - cy) / dy
        if (t > 0) {
          const x = cx + t * dx
          if (x >= s.x1 && x <= s.x2 && t < tMin) {
            tMin = t
            ix = x
            iy = s.y
          }
        }
      }
    }
  }
  return { x: ix, y: iy }
}

function getRelationEndpoints(fromId: number, toId: number) {
  const fromRect = conceptRects.value[fromId]
  const toRect = conceptRects.value[toId]
  if (!fromRect || !toRect) return { start: { x: 0, y: 0 }, end: { x: 0, y: 0 } }
  // Get center of each rect
  const fromCenter = {
    x: fromRect.left + fromRect.width / 2 - (viewportRef.value?.getBoundingClientRect().left ?? 0),
    y: fromRect.top + fromRect.height / 2 - (viewportRef.value?.getBoundingClientRect().top ?? 0),
  }
  const toCenter = {
    x: toRect.left + toRect.width / 2 - (viewportRef.value?.getBoundingClientRect().left ?? 0),
    y: toRect.top + toRect.height / 2 - (viewportRef.value?.getBoundingClientRect().top ?? 0),
  }
  const start = getRectLineIntersection(
    fromCenter.x,
    fromCenter.y,
    fromRect.width,
    fromRect.height,
    toCenter.x,
    toCenter.y,
  )
  const end = getRectLineIntersection(
    toCenter.x,
    toCenter.y,
    toRect.width,
    toRect.height,
    fromCenter.x,
    fromCenter.y,
  )
  return { start, end }
}

function getConceptRelations(id: number) {
  return relations.value.filter((r) => r.from === id || r.to === id)
}

function selectConcept(id: number) {
  selectedConceptId.value = id
}

function selectRelation(id: number) {
  selectedRelationId.value = id
}

function deselectRelation() {
  selectedRelationId.value = null
}

function showConceptContextMenu(e: MouseEvent, id: number) {
  contextMenu.value.visible = true
  contextMenu.value.x = e.clientX
  contextMenu.value.y = e.clientY
  contextMenu.value.conceptId = id
  document.addEventListener('click', hideContextMenu, { once: true })
}

function hideContextMenu() {
  contextMenu.value.visible = false
  contextMenu.value.conceptId = null
}

function deleteConcept(id: number) {
  // Remove the concept
  const idx = concepts.value.findIndex((c) => c.id === id)
  if (idx !== -1) concepts.value.splice(idx, 1)
  // Remove all relations involving this concept
  relations.value = relations.value.filter((r) => r.from !== id && r.to !== id)
  // Deselect if needed
  if (selectedConceptId.value === id) selectedConceptId.value = null
  hideContextMenu()
}

function onKeydown(e: KeyboardEvent) {
  // Only allow relation deletion by key if a relation is selected
  if ((e.key === 'Delete' || e.key === 'Backspace') && selectedRelationId.value !== null) {
    const idx = relations.value.findIndex((r) => r.id === selectedRelationId.value)
    if (idx !== -1) {
      relations.value.splice(idx, 1)
      selectedRelationId.value = null
    }
  }
}

function exportJSON() {
  const data = {
    concepts: concepts.value,
    relations: relations.value,
    offset: offset.value,
    scale: scale.value,
  }
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'mindmap.json'
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

function importJSON(e: Event) {
  const input = e.target as HTMLInputElement
  if (!input.files || !input.files[0]) return
  const file = input.files[0]
  const reader = new FileReader()
  reader.onload = (event) => {
    try {
      const data = JSON.parse(event.target?.result as string)
      if (!data.concepts || !data.relations || !data.offset || typeof data.scale !== 'number')
        throw new Error('Invalid format')
      concepts.value = data.concepts.map((c: Partial<Concept>) => ({
        ...c,
        size: c.size ?? 'medium',
      }))
      relations.value = data.relations
      offset.value = data.offset
      scale.value = data.scale
      nextId = concepts.value.reduce((max, c) => Math.max(max, c.id), 0) + 1
      nextRelationId = relations.value.reduce((max, r) => Math.max(max, r.id), 0) + 1
      alert('Mind map imported successfully!')
    } catch (err) {
      alert('Failed to import JSON: ' + (err as Error).message)
    }
    input.value = '' // reset file input
  }
  reader.readAsText(file)
}

function teleportToConcept(id: number) {
  const concept = concepts.value.find((c) => c.id === id)
  if (!concept || !viewportRef.value) return
  // Get viewport center
  const rect = viewportRef.value.getBoundingClientRect()
  const viewportCenterX = rect.width / 2
  const viewportCenterY = rect.height / 2
  // Use measured concept size if available, else default
  const conceptRect = conceptRects.value[id]
  let conceptCenterX, conceptCenterY
  if (conceptRect) {
    conceptCenterX = conceptRect.left + conceptRect.width / 2 - rect.left
    conceptCenterY = conceptRect.top + conceptRect.height / 2 - rect.top
  } else {
    // Fallback: use world coordinates
    conceptCenterX = concept.x * scale.value + offset.value.x + 150
    conceptCenterY = concept.y * scale.value + offset.value.y + 24
  }
  // Adjust offset so concept center is at viewport center
  offset.value.x += viewportCenterX - conceptCenterX
  offset.value.y += viewportCenterY - conceptCenterY
  selectConcept(id)
}

function handleWorldClick(e: MouseEvent) {
  // Only deselect if not clicking on a concept or context menu
  if (e.target === e.currentTarget) {
    deselectConcept()
    deselectRelation()
  }
}

function deselectConcept() {
  selectedConceptId.value = null
}
</script>

<style scoped>
.mindmap-layout {
  display: flex;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
.sidebar {
  width: 240px;
  background: #f4f4fa;
  border-right: 1px solid #d3d3e7;
  padding: 18px 12px 12px 18px;
  font-size: 15px;
  color: #222;
  z-index: 20;
  box-shadow: 2px 0 8px rgba(0, 0, 0, 0.03);
  overflow-y: auto;
}
.sidebar h2 {
  font-size: 18px;
  margin: 0 0 10px 0;
  font-weight: 600;
}
.sidebar ul {
  list-style: none;
  padding: 0;
  margin: 0;
}
.sidebar li {
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid #ececf7;
}
.sidebar .coords {
  color: #666;
  font-size: 13px;
}
.viewport {
  /* grid background now handled by :style binding */
}
.world {
  width: 100%;
  height: 100%;
  position: absolute;
  left: 0;
  top: 0;
  overflow: hidden;
}
.world-content {
  position: absolute;
  left: 0;
  top: 0;
  width: 100vw;
  height: 100vh;
  box-sizing: border-box;
  /* Remove grid background from here */
}
.rightbar {
  width: 340px;
  background: #f8f8ff;
  border-left: 1px solid #d3d3e7;
  padding: 24px 18px 18px 18px;
  font-size: 15px;
  color: #222;
  z-index: 20;
  box-shadow: -2px 0 8px rgba(0, 0, 0, 0.03);
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}
.rightbar h2 {
  font-size: 20px;
  margin: 0 0 12px 0;
  font-weight: 600;
}
.bullets-input {
  width: 100%;
  min-height: 180px;
  max-height: 320px;
  font-size: 15px;
  font-family: monospace;
  border: 1px solid #bbb;
  border-radius: 6px;
  padding: 10px;
  margin-top: 8px;
  background: #fff;
  resize: vertical;
}
.title-input {
  width: 100%;
  font-size: 17px;
  font-weight: 600;
  border: 1px solid #bbb;
  border-radius: 6px;
  padding: 8px 10px;
  margin-bottom: 10px;
  background: #fff;
  color: #222;
  margin-top: 0;
}
.placeholder {
  color: #888;
  font-size: 16px;
  margin-top: 40px;
  text-align: center;
}
.coords-overlay {
  position: absolute;
  top: 16px;
  right: 24px;
  background: rgba(30, 30, 40, 0.85);
  color: #fff;
  font-size: 14px;
  padding: 4px 10px;
  border-radius: 6px;
  pointer-events: none;
  z-index: 10;
  font-family: monospace;
}
.bottombar {
  position: fixed;
  left: 240px;
  right: 340px;
  bottom: 0;
  height: 54px;
  background: #f4f4fa;
  border-top: 1px solid #d3d3e7;
  display: flex;
  align-items: center;
  padding: 0 24px;
  z-index: 30;
  gap: 18px;
}
.bottombar button {
  background: #1976d2;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 8px 22px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}
.bottombar button.active {
  background: #1565c0;
}
.import-label {
  background: #fff;
  color: #1976d2;
  border: 1.5px solid #1976d2;
  border-radius: 6px;
  padding: 7px 20px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  margin-left: 0;
  margin-right: 0;
  transition:
    background 0.15s,
    color 0.15s;
  display: inline-block;
}
.import-label:hover {
  background: #e3f2fd;
}
.relation-hint {
  color: #1976d2;
  font-size: 15px;
  margin-left: 12px;
}
.relations-svg {
  pointer-events: auto;
  position: absolute;
  left: 0;
  top: 0;
  width: 100vw;
  height: 100vh;
  z-index: 2;
}
.relations-list {
  margin-top: 18px;
}
.relations-list h3 {
  font-size: 16px;
  margin: 0 0 6px 0;
  font-weight: 600;
}
.relations-list ul {
  list-style: none;
  padding: 0;
  margin: 0;
}
.relations-list li {
  font-size: 15px;
  color: #1976d2;
  margin-bottom: 4px;
}
.context-menu {
  position: fixed;
  z-index: 1000;
  background: #fff;
  border: 1px solid #bbb;
  border-radius: 6px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.13);
  min-width: 120px;
  font-size: 15px;
  padding: 4px 0;
}
.context-menu ul {
  list-style: none;
  margin: 0;
  padding: 0;
}
.context-menu li {
  padding: 8px 18px;
  cursor: pointer;
  color: #e53935;
  font-weight: 600;
  border-radius: 4px;
  transition: background 0.13s;
}
.context-menu li:hover {
  background: #fbe9e7;
}
.size-select {
  width: 100%;
  font-size: 15px;
  border: 1px solid #bbb;
  border-radius: 6px;
  padding: 7px 10px;
  margin-bottom: 10px;
  background: #fff;
  color: #222;
  margin-top: 0;
  margin-bottom: 12px;
}
</style>
