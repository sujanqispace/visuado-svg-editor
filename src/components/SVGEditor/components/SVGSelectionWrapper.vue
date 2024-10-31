<template>
  <div
    ref="container"
    class="selection-container"
    @mousedown="handleMouseDown"
    @mousemove="handleMouseMove"
    @mouseup="handleMouseUp"
    @mouseleave="handleMouseUp"
  >
    <slot></slot>

    <div
      v-if="isDrawing"
      class="selection-box"
      :style="{
        left: `${selectionBox.x}px`,
        top: `${selectionBox.y}px`,
        width: `${selectionBox.width}px`,
        height: `${selectionBox.height}px`,
      }"
    ></div>

    <!-- Render segment markers (optional for debugging) -->
    <svg v-if="debug" class="segment-markers">
      <circle
        v-for="point in segmentPoints"
        :key="`${point.x}-${point.y}`"
        :cx="point.x"
        :cy="point.y"
        r="2"
        fill="blue"
      />
    </svg>
  </div>
</template>

<script>
export default {
  name: 'SVGSegmentSelectionWrapper',
  props: {
    selectableSelector: {
      type: String,
      default: '.data-selectable',
    },
    segmentLength: {
      type: Number,
      default: 20, // Length of each segment in SVG units
    },
    debug: {
      type: Boolean,
      default: true,
    },
  },

  data() {
    return {
      isShiftPressed: false,
      isDrawing: false,
      selectionBox: {
        x: 0,
        y: 0,
        width: 0,
        height: 0,
      },
      startPoint: {
        x: 0,
        y: 0,
      },
      selectedSegments: new Map(), // Map<pathId, Set<segmentIndex>>
      pathSegments: new Map(), // Map<pathId, Array<{start, end, length}>>
      segmentPoints: [], // For debugging visualization
    }
  },

  mounted() {
    // Shift key detection
    window.addEventListener('keydown', this.handleKeyDown)
    window.addEventListener('keyup', this.handleKeyUp)
    this.initializePathSegments()
  },
  onBeforeUnmount(callback) {
    window.removeEventListener('keydown', this.handleKeyDown)
    window.removeEventListener('keyup', this.handleKeyUp)
  },
  methods: {
    handleKeyDown(e) {
      if (e.shiftKey) {
        e.preventDefault()
        this.isShiftPressed = true
      }
    },

    handleKeyUp(e) {
      this.isShiftPressed = false
    },
    initializePathSegments() {
      const svg = this.$refs.container.querySelector('svg')
      if (!svg) return

      const selectableElements = this.$refs.container.querySelectorAll(
        this.selectableSelector
      )
      console.log(selectableElements)
      selectableElements.forEach((path) => {
        const segments = this.dividePathIntoSegments(path)
        this.pathSegments.set(path.id, segments)
      })
    },

    dividePathIntoSegments(path) {
      const totalLength = path.getTotalLength()
      const numSegments = Math.ceil(totalLength / this.segmentLength)
      const segments = []
      this.segmentPoints = [] // Reset segment points

      for (let i = 0; i < numSegments; i++) {
        const startDistance = i * this.segmentLength
        const endDistance = Math.min((i + 1) * this.segmentLength, totalLength)

        const startPoint = path.getPointAtLength(startDistance)
        const endPoint = path.getPointAtLength(endDistance)

        segments.push({
          index: i,
          start: startPoint,
          end: endPoint,
          length: endDistance - startDistance,
          selected: false,
        })

        if (this.debug) {
          this.segmentPoints.push(startPoint)
          if (i === numSegments - 1) {
            this.segmentPoints.push(endPoint)
          }
        }
      }

      return segments
    },

    getMousePosition(event) {
      const rect = this.$refs.container.getBoundingClientRect()
      return {
        x: event.clientX - rect.left,
        y: event.clientY - rect.top,
      }
    },

    getSVGPoint(x, y) {
      const svg = this.$refs.container.querySelector('svg')
      const point = svg.createSVGPoint()
      point.x = x
      point.y = y

      const CTM = svg.getScreenCTM()
      if (CTM) {
        return point.matrixTransform(CTM.inverse())
      }
      return point
    },

    handleMouseDown(event) {
      if (!this.isShiftPressed) return
      if (event.button !== 0) return

      const point = this.getMousePosition(event)
      this.isDrawing = true
      this.startPoint = point
      this.selectionBox = {
        x: point.x,
        y: point.y,
        width: 0,
        height: 0,
      }

      if (!event.shiftKey) {
        this.selectedSegments.clear()
        this.updateSelectedSegments()
      }
    },

    handleMouseMove(event) {
      if (!this.isShiftPressed) return
      if (!this.isDrawing) return

      const point = this.getMousePosition(event)
      const newBox = {
        x: Math.min(point.x, this.startPoint.x),
        y: Math.min(point.y, this.startPoint.y),
        width: Math.abs(point.x - this.startPoint.x),
        height: Math.abs(point.y - this.startPoint.y),
      }
      this.selectionBox = newBox

      const svg = this.$refs.container.querySelector('svg')
      if (!svg) return

      const topLeft = this.getSVGPoint(newBox.x, newBox.y)
      const bottomRight = this.getSVGPoint(
        newBox.x + newBox.width,
        newBox.y + newBox.height
      )

      const selectionBoxSVG = {
        x: topLeft.x,
        y: topLeft.y,
        width: bottomRight.x - topLeft.x,
        height: bottomRight.y - topLeft.y,
      }

      this.updateSelectionForBox(selectionBoxSVG)
    },

    updateSelectionForBox(box) {
      const svg = this.$refs.container.querySelector('svg')
      if (!svg) return

      const selectableElements = svg.querySelectorAll(this.selectableSelector)

      selectableElements.forEach((path) => {
        if (!path.id) return

        const segments = this.pathSegments.get(path.id)
        if (!segments) return

        let pathSegments = this.selectedSegments.get(path.id) || new Set()

        segments.forEach((segment, index) => {
          if (this.isSegmentInBox(segment, box)) {
            pathSegments.add(index)
          }
        })

        if (pathSegments.size > 0) {
          this.selectedSegments.set(path.id, pathSegments)
        }
      })

      this.updateSelectedSegments()
      this.emitSelectionChange()
    },

    isSegmentInBox(segment, box) {
      // Check if either endpoint is in the box
      if (
        this.isPointInBox(segment.start, box) ||
        this.isPointInBox(segment.end, box)
      ) {
        return true
      }

      // Check if segment intersects with box edges
      return this.isLineIntersectingBox(segment.start, segment.end, box)
    },

    isLineIntersectingBox(start, end, box) {
      // Check intersection with box edges using line segment intersection
      const boxLines = [
        {
          start: { x: box.x, y: box.y },
          end: { x: box.x + box.width, y: box.y },
        },
        {
          start: { x: box.x + box.width, y: box.y },
          end: { x: box.x + box.width, y: box.y + box.height },
        },
        {
          start: { x: box.x + box.width, y: box.y + box.height },
          end: { x: box.x, y: box.y + box.height },
        },
        {
          start: { x: box.x, y: box.y + box.height },
          end: { x: box.x, y: box.y },
        },
      ]

      return boxLines.some((line) =>
        this.doLinesIntersect(start, end, line.start, line.end)
      )
    },

    doLinesIntersect(p1, p2, p3, p4) {
      const denominator =
        (p4.y - p3.y) * (p2.x - p1.x) - (p4.x - p3.x) * (p2.y - p1.y)
      if (denominator === 0) return false

      const ua =
        ((p4.x - p3.x) * (p1.y - p3.y) - (p4.y - p3.y) * (p1.x - p3.x)) /
        denominator
      const ub =
        ((p2.x - p1.x) * (p1.y - p3.y) - (p2.y - p1.y) * (p1.x - p3.x)) /
        denominator

      return ua >= 0 && ua <= 1 && ub >= 0 && ub <= 1
    },

    isPointInBox(point, box) {
      return (
        point.x >= box.x &&
        point.x <= box.x + box.width &&
        point.y >= box.y &&
        point.y <= box.y + box.height
      )
    },

    handleMouseUp() {
      this.isDrawing = false
    },

    updateSelectedSegments() {
      const svg = this.$refs.container.querySelector('svg')
      if (!svg) return

      const selectableElements = svg.querySelectorAll(this.selectableSelector)
      selectableElements.forEach((path) => {
        if (!path.id) return

        const selectedSegmentsForPath = this.selectedSegments.get(path.id)
        const segments = this.pathSegments.get(path.id)

        if (!segments) return

        // Create a new path element for highlighting selected segments
        let highlightPath = svg.querySelector(`#highlight-${path.id}`)
        if (!highlightPath) {
          highlightPath = document.createElementNS(
            'http://www.w3.org/2000/svg',
            'path'
          )
          highlightPath.id = `highlight-${path.id}`
          highlightPath.classList.add('segment-highlight')
          path.parentNode.insertBefore(highlightPath, path.nextSibling)
        }

        if (selectedSegmentsForPath && selectedSegmentsForPath.size > 0) {
          // Build path data for selected segments
          const pathData = Array.from(selectedSegmentsForPath)
            .map((index) => {
              const segment = segments[index]
              return `M ${segment.start.x} ${segment.start.y} L ${segment.end.x} ${segment.end.y}`
            })
            .join(' ')

          highlightPath.setAttribute('d', pathData)
          highlightPath.style.display = 'block'
        } else {
          highlightPath.style.display = 'none'
        }
      })
    },

    emitSelectionChange() {
      const selection = {}
      this.selectedSegments.forEach((segments, pathId) => {
        selection[pathId] = Array.from(segments)
      })

      this.$emit('selection-change', this.selectionBox, selection)
    },
  },
}
</script>

<style>
.selection-container {
  position: absolute;
  width: 100%;
  height: 100%;
}

.selection-box {
  position: absolute;
  border: 1px solid #2196f3;
  background-color: rgba(33, 150, 243, 0.1);
  pointer-events: none;
}

.segment-markers {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.segment-highlight {
  stroke: #2196f3;
  stroke-width: 3;
  fill: none;
  pointer-events: none;
}

[data-selectable] {
  cursor: pointer;
  transition: stroke 0.2s ease;
}

[data-selectable]:hover {
  stroke-width: 2;
}
</style>
