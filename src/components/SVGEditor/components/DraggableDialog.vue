<!-- DraggableDialog.vue -->
<template>
  <v-card
    :style="dialogStyle"
    class="draggable-dialog"
    :elevation="isDragging ? 8 : 2"
    @mousedown="handleMouseDown"
    ref="dialogRef"
  >
    <!-- Dialog Header -->
    <v-card-title
      class="drag-handle d-flex align-center py-0 cursor-move"
      @dblclick="isCollapsed = !isCollapsed"
    >
      <span class="text-truncate text-subtitle-1">{{ title }}</span>
      <v-spacer></v-spacer>
      <v-btn
        variant="text"
        size="small"
        @click="isCollapsed = !isCollapsed"
        :icon="isCollapsed ? 'mdi-chevron-down' : 'mdi-chevron-up'"
      ></v-btn>
    </v-card-title>

    <!-- Dialog Content -->
    <v-expand-transition>
      <div v-show="!isCollapsed">
        <v-card-text
          :style="{ maxHeight: `${maxHeight}px`, overflowY: 'auto' }"
          class="pa-0"
        >
          <slot></slot>
        </v-card-text>
      </div>
    </v-expand-transition>
  </v-card>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  title: {
    type: String,
    default: 'Dialog',
  },
  initialPosition: {
    type: Object,
    default: () => ({ x: 20, y: 20 }),
  },
  width: {
    type: Number,
    default: 200,
  },
  maxWidth: {
    type: Number,
    default: 400,
  },
  maxHeight: {
    type: Number,
    default: 600,
  },
  zIndex: {
    type: Number,
    default: 1,
  },
  alignment: {
    type: String,
    default: 'left',
    validator: (value) => ['left', 'right'].includes(value),
  },
})

const dialogRef = ref(null)
const position = ref(null)
const isDragging = ref(false)
const isCollapsed = ref(false)
const dragOffset = ref({ x: 0, y: 0 })

// Initialize position based on alignment
onMounted(() => {
  const dialogElement = dialogRef.value.$el
  const dialogWidth = dialogElement.offsetWidth
  const windowWidth = window.innerWidth

  if (props.alignment === 'right') {
    position.value = {
      x: windowWidth - dialogWidth - props.initialPosition.x,
      y: props.initialPosition.y,
    }
  } else {
    position.value = { ...props.initialPosition }
  }
})

const dialogStyle = computed(() => ({
  position: 'absolute',
  left: position.value ? `${position.value.x}px` : '0px',
  top: position.value ? `${position.value.y}px` : '0px',
  width: `${props.width}px`,
  maxWidth: `${props.maxWidth}px`,
  zIndex: props.zIndex,
  transition: isDragging.value ? 'none' : 'transform 0.2s, elevation 0.2s',
  transform: isDragging.value ? 'scale(1.02)' : 'scale(1)',
}))

const handleMouseDown = (e) => {
  if (e.target.closest('.drag-handle')) {
    isDragging.value = true
    const rect = dialogRef.value.$el.getBoundingClientRect()
    dragOffset.value = {
      x: e.clientX - rect.left,
      y: e.clientY - rect.top,
    }
  }
}

const handleMouseMove = (e) => {
  if (isDragging.value) {
    const newX = e.clientX - dragOffset.value.x
    const newY = e.clientY - dragOffset.value.y

    // Ensure dialog stays within viewport
    const maxX = window.innerWidth - dialogRef.value.$el.offsetWidth
    const maxY = window.innerHeight - dialogRef.value.$el.offsetHeight

    position.value = {
      x: Math.max(0, Math.min(newX, maxX)),
      y: Math.max(0, Math.min(newY, maxY)),
    }
  }
}

const handleMouseUp = () => {
  isDragging.value = false
}

// Handle window resize
const handleResize = () => {
  if (!isDragging.value && props.alignment === 'right' && position.value) {
    const dialogElement = dialogRef.value.$el
    const dialogWidth = dialogElement.offsetWidth
    const windowWidth = window.innerWidth
    position.value = {
      x: Math.min(position.value.x, windowWidth - dialogWidth),
      y: position.value.y,
    }
  }
}

onMounted(() => {
  window.addEventListener('mousemove', handleMouseMove)
  window.addEventListener('mouseup', handleMouseUp)
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('mousemove', handleMouseMove)
  window.removeEventListener('mouseup', handleMouseUp)
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped>
.draggable-dialog {
  touch-action: none;
  user-select: none;
}

.cursor-move {
  cursor: move;
}

.drag-handle {
  background-color: #8bc7ff;
}
</style>
