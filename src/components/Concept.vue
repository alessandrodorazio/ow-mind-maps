<template>
  <div
    class="concept"
    :class="[sizeClass, { selected }]"
    :style="{ left: x + 'px', top: y + 'px' }"
    @mousedown="onDragStart"
    @dblclick.stop="startEditing"
  >
    <input
      v-if="editing"
      ref="inputRef"
      class="concept-input"
      v-model="editTitle"
      @blur="saveEdit"
      @keydown.enter="saveEdit"
      @keydown.esc="cancelEdit"
      maxlength="64"
    />
    <span v-else :style="{ fontWeight: fontWeightStyle }">{{ title }}</span>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, nextTick, defineOptions, computed } from 'vue'

defineOptions({ name: 'MindMapConcept' })

const props = defineProps<{
  id: number | string
  x: number
  y: number
  title: string
  selected?: boolean
  size?: 'small' | 'medium' | 'large' | 'xlarge'
  fontWeight?: 'regular' | 'bold'
}>()
const emits = defineEmits(['update:title', 'drag', 'dragStart', 'dragEnd'])

const editing = ref(false)
const editTitle = ref(props.title)
const inputRef = ref<HTMLInputElement | null>(null)

watch(
  () => props.title,
  (val) => {
    if (!editing.value) editTitle.value = val
  },
)

const sizeClass = computed(() => {
  if (props.size === 'small') return 'concept-small'
  if (props.size === 'large') return 'concept-large'
  if (props.size === 'xlarge') return 'concept-xlarge'
  return 'concept-medium'
})

const fontWeightStyle = computed(() => {
  return props.fontWeight === 'bold' ? 'bold' : 'normal'
})

function startEditing() {
  editing.value = true
  nextTick(() => inputRef.value?.focus())
}
function saveEdit() {
  if (editing.value) {
    emits('update:title', editTitle.value)
    editing.value = false
  }
}
function cancelEdit() {
  editing.value = false
  editTitle.value = props.title
}

// Drag logic
let dragStart = { x: 0, y: 0, mouseX: 0, mouseY: 0, dx: 0, dy: 0 }
function onDragStart(e: MouseEvent) {
  if (editing.value) return
  // Calculate offset from pointer to top-left of concept
  dragStart = {
    x: props.x,
    y: props.y,
    mouseX: e.clientX,
    mouseY: e.clientY,
    dx: e.clientX - props.x,
    dy: e.clientY - props.y,
  }
  emits('dragStart', { dx: dragStart.dx, dy: dragStart.dy })
  window.addEventListener('mousemove', onDragMove)
  window.addEventListener('mouseup', onDragEnd)
}
function onDragMove(e: MouseEvent) {
  // Use the initial offset to keep the pointer at the same place in the concept
  const newX = e.clientX - dragStart.dx
  const newY = e.clientY - dragStart.dy
  emits('drag', { x: newX, y: newY })
}
function onDragEnd() {
  emits('dragEnd')
  window.removeEventListener('mousemove', onDragMove)
  window.removeEventListener('mouseup', onDragEnd)
}
</script>

<style scoped>
.concept {
  position: absolute;
  background: #fff;
  color: #111;
  border: 1.5px solid #222;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
  padding: 12px 18px;
  font-size: 16px;
  cursor: grab;
  user-select: none;
  transition:
    box-shadow 0.15s,
    border-color 0.15s,
    font-size 0.15s,
    min-width 0.15s,
    min-height 0.15s,
    max-width 0.15s,
    padding 0.15s;
  z-index: 1;
  outline: none;
  display: flex;
  align-items: center;
  min-width: 80px;
  min-height: 32px;
  max-width: 300px;
}
.concept.selected {
  border: 2.5px solid #1976d2;
  box-shadow: 0 0 0 2px #90caf9;
}
.concept:active {
  cursor: grabbing;
}
.concept-input {
  width: 100%;
  font-size: inherit;
  border: none;
  outline: none;
  background: #fff;
  color: #111;
  padding: 0;
}
/* Size variants */
.concept-small {
  font-size: 13px;
  min-width: 60px;
  max-width: 180px;
  min-height: 24px;
  padding: 7px 12px;
}
.concept-medium {
  font-size: 16px;
  min-width: 80px;
  max-width: 300px;
  min-height: 32px;
  padding: 12px 18px;
}
.concept-large {
  font-size: 22px;
  min-width: 120px;
  max-width: 420px;
  min-height: 48px;
  padding: 18px 28px;
}
.concept-xlarge {
  font-size: 30px;
  min-width: 180px;
  max-width: 600px;
  min-height: 64px;
  padding: 28px 38px;
}
</style>
