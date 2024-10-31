<template>
  <div class="zoom-pan-wrapper">
    <div
      class="zoomResizeContainer"
      ref="zoomResizeContainer"
      :class="{ grabbing }"
    >
      <div class="content-wrapper" ref="content">
        <!-- Slot for any content -->
        <slot></slot>
      </div>
    </div>
    <div class="controls" v-if="showControls">
      Hold SPACE + Scroll to zoom, Click and drag to pan
    </div>
  </div>
</template>

<script>
export default {
  name: 'ZoomAndResize',

  props: {
    initialScale: {
      type: Number,
      default: 1,
    },
    minScale: {
      type: Number,
      default: 0.5,
    },
    maxScale: {
      type: Number,
      default: 5,
    },
    showControls: {
      type: Boolean,
      default: true,
    },
  },

  data() {
    return {
      isSpacePressed: false,
      isDragging: false,
      startX: 0,
      startY: 0,
      translateX: 0,
      translateY: 0,
      scale: this.initialScale,
    }
  },
  computed: {
    grabbing() {
      return this.isSpacePressed
    },
  },

  mounted() {
    // Space bar detection
    window.addEventListener('keydown', this.handleKeyDown)
    window.addEventListener('keyup', this.handleKeyUp)

    // Add wheel event to zoomResizeContainer
    this.$refs.zoomResizeContainer.addEventListener('wheel', this.handleWheel, {
      passive: false,
    })

    // Add mouse events to zoomResizeContainer
    this.$refs.zoomResizeContainer.addEventListener(
      'mousedown',
      this.handleMouseDown
    )
    this.$refs.zoomResizeContainer.addEventListener(
      'dblclick',
      this.handleDoubleClick
    )

    // Cleanup listeners on component destroy
  },
  onBeforeUnmount(callback) {
    window.removeEventListener('keydown', this.handleKeyDown)
    window.removeEventListener('keyup', this.handleKeyUp)
    window.removeEventListener('mousemove', this.handleMouseMove)
    window.removeEventListener('mouseup', this.handleMouseUp)
    this.$refs.zoomResizeContainer.removeEventListener(
      'wheel',
      this.handleWheel
    )
    this.$refs.zoomResizeContainer.removeEventListener(
      'mousedown',
      this.handleMouseDown
    )
    this.$refs.zoomResizeContainer.removeEventListener(
      'dblclick',
      this.handleDoubleClick
    )
  },

  methods: {
    handleKeyDown(e) {
      if (e.code === 'Space') {
        e.preventDefault()
        this.isSpacePressed = true
      }
    },

    handleKeyUp(e) {
      if (e.code === 'Space') {
        this.isSpacePressed = false
      }
    },

    handleWheel(e) {
      if (this.isSpacePressed) {
        e.preventDefault()
        const delta = e.deltaY > 0 ? 0.9 : 1.1

        // Get the mouse position relative to the zoomResizeContainer
        const rect = this.$refs.zoomResizeContainer.getBoundingClientRect()
        const mouseX = e.clientX - rect.left
        const mouseY = e.clientY - rect.top

        // Calculate new scale
        const newScale = this.scale * delta

        // Limit scale
        if (newScale >= this.minScale && newScale <= this.maxScale) {
          // Calculate mouse position relative to content
          const beforeX = mouseX - this.translateX
          const beforeY = mouseY - this.translateY

          this.scale = newScale

          // Adjust position to zoom towards mouse cursor
          const afterX = beforeX * delta
          const afterY = beforeY * delta
          this.translateX += beforeX - afterX
          this.translateY += beforeY - afterY

          this.updateTransform()
        }
      }
    },

    handleMouseDown(e) {
      if (this.isSpacePressed) {
        this.isDragging = true
        this.startX = e.clientX - this.translateX
        this.startY = e.clientY - this.translateY

        window.addEventListener('mousemove', this.handleMouseMove)
        window.addEventListener('mouseup', this.handleMouseUp)
      }
    },

    handleMouseMove(e) {
      if (this.isDragging) {
        this.translateX = e.clientX - this.startX
        this.translateY = e.clientY - this.startY
        this.updateTransform()
      }
    },

    handleMouseUp() {
      this.isDragging = false
      window.removeEventListener('mousemove', this.handleMouseMove)
      window.removeEventListener('mouseup', this.handleMouseUp)
    },

    handleDoubleClick() {
      this.scale = this.initialScale
      this.translateX = 0
      this.translateY = 0
      this.updateTransform()
      this.$emit('reset')
    },

    updateTransform() {
      if (this.$refs.content) {
        this.$refs.content.style.transform = `translate(${this.translateX}px, ${this.translateY}px) scale(${this.scale})`

        this.$emit('transform-update', {
          scale: this.scale,
          translateX: this.translateX,
          translateY: this.translateY,
        })
      }
    },
  },
}
</script>

<style scoped>
.zoom-pan-wrapper {
  width: 100%;
  height: 100%;
}

.zoomResizeContainer {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: grab;
  background: #bababa;
  overflow: hidden;
  position: relative;
}

.zoomResizeContainer.grabbing {
  cursor: grabbing;
}

.zoomResizeContainer {
  cursor: default;
}

.content-wrapper {
  transform-origin: center;
  transition: transform 0.1s ease-out;
}

.controls {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px 20px;
  border-radius: 5px;
  font-family: Arial, sans-serif;
  z-index: 1000;
  opacity: 1;

  -moz-animation: HideAfter5 0s ease-in 5s forwards;
  /* Firefox */
  -webkit-animation: HideAfter5 0s ease-in 5s forwards;
  /* Safari and Chrome */
  -o-animation: HideAfter5 0s ease-in 5s forwards;
  /* Opera */
  animation: HideAfter5 0s ease-in 5s forwards;
  -webkit-animation-fill-mode: forwards;
  animation-fill-mode: forwards;
}
@keyframes HideAfter5 {
  to {
    width: 0;
    height: 0;
    opacity: 0;
    overflow: hidden;
    display: none;
  }
}

@-webkit-keyframes HideAfter5 {
  to {
    width: 0;
    height: 0;
    opacity: 0;
    visibility: hidden;
    display: none;
  }
}
</style>
