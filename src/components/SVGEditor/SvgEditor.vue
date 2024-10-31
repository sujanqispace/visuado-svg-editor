<template>
  <v-app class="position-relative">
    <v-system-bar window absolute color="primary">
      <div>SVG Editor</div>
      <v-spacer></v-spacer>
      <div>
        <v-switch
          density="compact"
          hide-details
          v-model="svgEditorSettings.showPathBoundingBox"
          label="Show Bounding Box"
        ></v-switch>
      </div>

      <div>
        <div class="system-bar-group">
          <div v-if="svgEditorSettings.drawMode">
            <v-icon icon="mdi-draw"></v-icon>
            Drawing Mode
          </div>
          <div v-else>
            <v-icon icon="mdi-cursor-default"></v-icon>
            Selection Mode
          </div>
        </div>
      </div>
      <div class="d-flex">
        <div class="system-bar-group d-flex">
          <div @click="zoomReset">{{ (zoomLevel * 100).toFixed(2) }}%</div>
          <v-icon @click="zoomIn" class="ms-2">mdi-magnify-plus</v-icon>
          <v-icon @click="zoomOut" class="ms-2">mdi-magnify-minus</v-icon>
        </div>

        <div class="system-bar-group">
          <v-icon
            @click="saveSVG"
            icon="mdi-content-save"
            class="ms-2"
          ></v-icon>
        </div>
      </div>
    </v-system-bar>

    <DraggableDialog
      title="Background"
      :initial-position="{ x: 10, y: 50 }"
      :z-index="2"
    >
      <BackgroundSelector @setBackground="(e) => setBackgroundUrl(e)" />
    </DraggableDialog>

    <DraggableDialog
      title="Paths"
      :initial-position="{ x: 10, y: 200 }"
      :z-index="3"
    >
      <v-list>
        <v-list-item
          density="compact"
          v-for="path in paths"
          :key="path.id"
          @click="selectPath(path)"
          :class="{
            'bg-primary': selectedPath && selectedPath.id === path.id,
          }"
        >
          {{ path.name }} {{ path.apartmentId }}
        </v-list-item>
      </v-list>
    </DraggableDialog>

    <DraggableDialog
      v-if="selectedPath"
      title="Edit Path Segments"
      :initial-position="{ x: 210, y: 33, right: true }"
      :width="300"
      :z-index="2"
      alignment="right"
    >
      <div class="dialog-content">
        <!-- Right Sidebar: Path Editor -->
        <div>
          <v-card>
            <v-card-text>
              <div class="d-flex mb-2">
                <v-btn
                  :color="svgEditorSettings.drawMode ? 'primary' : 'default'"
                  @click="setModeToDrawing"
                  icon="mdi-draw"
                  size="x-small"
                  class="mr-1"
                ></v-btn>
                <v-btn
                  :color="svgEditorSettings.drawMode ? 'default' : 'primary'"
                  @click="setModeToSelect"
                  icon="mdi-cursor-default"
                  size="x-small"
                ></v-btn>
                <v-spacer></v-spacer>
                <v-btn
                  color="error"
                  @click="clearSegments"
                  icon="mdi-delete"
                  size="x-small"
                ></v-btn>
              </div>
              <v-textarea
                v-model="editingPath"
                @input="updatePath"
                label="Path Data"
                rows="5"
              ></v-textarea>
              <!-- <v-btn color="primary" @click="savePath">Update</v-btn> -->
            </v-card-text>
          </v-card>
          <v-card v-if="pathSegments.length">
            <v-card-title>Path Segments</v-card-title>
            <v-card-text>
              <div>
                <div
                  v-for="(segment, index) in pathSegments"
                  :key="index"
                  class="overflow-scroll"
                >
                  <div
                    class="border d-flex justify-space-between align-center px-2 cursor-pointer"
                    :class="{
                      'active-path-segment': selectedPathSegmentIndex === index,
                    }"
                    @click="selectPathSegment(segment, index)"
                  >
                    <div class="d-flex justify-center">
                      {{ segment }}
                      <span v-if="selectedPathSegmentIndex === index">*</span>
                    </div>
                    <div class="d-flex justify-end">
                      <v-btn
                        icon="mdi-delete"
                        size="x-small"
                        @click="deleteSegment(index)"
                        color="error"
                        class="ml-auto"
                      ></v-btn>
                    </div>
                  </div>
                </div>
              </div>
            </v-card-text>
          </v-card>
        </div>
      </div>
    </DraggableDialog>

    <!-- Complex component example -->
    <SVGSelectionWrapper @selection-change="handleSelectionChange">
      <ZoomAndResize
        class="wrapper"
        ref="svgContainer"
        :initial-scale="1"
        :min-scale="0.5"
        :max-scale="5"
        :show-controls="true"
        @transform-update="handleTransformUpdate"
        @reset="handleReset"
      >
        <!-- Middle: SVG Preview -->
        <div>
          <svg
            :width="svgWidth"
            :height="svgHeight"
            :viewBox="viewBox"
            :style="{
              transform: `scale(${zoomLevel})`,
              background: backgroundUrl ? `url(${backgroundUrl})` : 'white ',
              backgroundPosition: 'center',
              backgroundRepeat: 'no-repeat',
              backgroundSize: 'contain',
            }"
            preserveAspectRatio="xMidYMid meet"
            id="svg-element"
            @click="drawAPoint"
            @mousemove="mouseMove($event, selectedPathSegmentIndex)"
            @mouseup="mouseUp($event, selectedPathSegmentIndex)"
            @mouseleave="mouseLeave($event, selectedPathSegmentIndex)"
            :class="{
              'svg-drawing-mode': svgEditorSettings.drawMode,
            }"
          >
            <g>
              <path
                v-for="path in paths"
                :key="path.id"
                :d="path.d"
                :stroke="path.stroke || 'black'"
                :fill="'none'"
                :stroke-width="1"
                :opacity="selectedPath && selectedPath.id !== path.id ? 0.3 : 1"
                :class="{
                  'selected-path': selectedPath && selectedPath.id === path.id,
                  'data-selectable':
                    selectedPath && selectedPath.id === path.id,
                }"
              />
            </g>
            <rect
              v-if="
                selectedPathBoundingBox && svgEditorSettings.showPathBoundingBox
              "
              :x="selectedPathBoundingBox.x"
              :y="selectedPathBoundingBox.y"
              :width="selectedPathBoundingBox.width"
              :height="selectedPathBoundingBox.height"
              class="active-bounding-box"
              fill="none"
              stroke="rgba(0, 0, 0, 0.7)"
              stroke-width="1"
              stroke-dasharray="13,5"
              vector-effect="non-scaling-stroke"
            />
            <circle
              v-if="selectedSegmentCoords && selectedPathSegmentIndex !== -1"
              :cx="selectedSegmentCoords[0]"
              :cy="selectedSegmentCoords[1]"
              class="active-vertex"
              r="15"
              fill="red"
              fill-opacity="0.5"
              stroke="rgba(0, 0, 0, 1)"
              stroke-width="3"
              @mousedown="mouseDown($event, selectedPathSegmentIndex)"
            ></circle>
          </svg>
        </div>
      </ZoomAndResize>
    </SVGSelectionWrapper>

    <div id="debug-info">
      <!-- Add this for debugging -->
      {{ mouseWithExtractor }} {{ parentEl }}
      <div v-if="selectedPathBoundingBox">
        Bounding Box: {{ selectedPathBoundingBox }}
      </div>
    </div>

    <DisplayPressedKeys />

    <v-dialog max-width="500">
      <!-- SVG Input and Parsing -->
      <template v-slot:activator="{ props: activatorProps }">
        <v-fab
          extended
          absolute
          color="primary"
          prepend-icon="mdi-plus"
          v-bind="activatorProps"
          text="New SVG"
        ></v-fab>
      </template>

      <template v-slot:default="{ isActive }">
        <v-card>
          <v-card-title>SVG Input</v-card-title>
          <v-card-text>
            <v-textarea
              v-model="svgInput"
              label="Paste your SVG here"
              rows="10"
            ></v-textarea>
            <v-btn color="primary" @click="parseSVG">Parse SVG</v-btn>
          </v-card-text>

          <v-card-actions>
            <v-spacer></v-spacer>

            <v-btn text="Close Dialog" @click="isActive.value = false"></v-btn>
          </v-card-actions>
        </v-card>
      </template>
    </v-dialog>
  </v-app>
</template>

<script setup>
import { ref, onMounted, watch, computed } from 'vue'
import BackgroundSelector from './components/BackgroundSelector.vue'
import ZoomAndResize from './components/ZoomAndResize.vue'
import DraggableDialog from './components/DraggableDialog.vue'
import SVGSelectionWrapper from './components/SVGSelectionWrapper.vue'
import DisplayPressedKeys from './components/DisplayPressedKeys.vue'

import { useMouse, useParentElement } from '@vueuse/core'
import { reactive } from 'vue'

const handleTransformUpdate = ({ scale, translateX, translateY }) => {}

const handleReset = () => {}

const parentEl = useParentElement()
const mouseDefault = reactive(useMouse())
const extractor = (event) => {
  if (typeof Touch !== 'undefined' && event instanceof Touch) return null
  else return [event.offsetX, event.offsetY]
}

const mouseWithExtractor = reactive(
  useMouse({ target: parentEl, type: extractor })
)

const svgEditorSettings = ref({
  showPathBoundingBox: true,
  drawMode: false,
})

const changeIdentifier = ref(0)
const backgroundUrl = ref(null)
const svgDocument = ref(null)
const zoomLevel = ref(0.5)
const paths = ref([])
const selectedPath = ref(null)
const selectedPathSegment = ref(null)
const selectedPathSegmentIndex = ref(-1)
const editingPath = ref('')
const svgInput = ref('')
const svgContainer = ref(null)
const svgWidth = ref(0)
const svgHeight = ref(0)
const viewBox = ref('0 0 2560 1440')
const viewBoxWidth = ref(2560)
const viewBoxHeight = ref(1440)
const pathSegments = ref([])

const selectedPathBoundingBox = ref(null)

// Default 2K resolution SVG example
const defaultSVG = `
      <svg xmlns="http://www.w3.org/2000/svg" width="2560" height="1440" viewBox="0 0 2560 1440">
        <path d="M100,100 L346,100 L460,340 L100,340 Z" fill="none" stroke="blue" stroke-width="5"/>
        <path d="M900,300 L1060,300 L1190,740 L700,740 Z" fill="none" stroke="red" stroke-width="5"/>
        <path d="M1580,500 L1660,500 L1660,940 L900,940 Z" fill="none" stroke="green" stroke-width="5"/>
      </svg>
    `

const setBackgroundUrl = (url) => {
  backgroundUrl.value = url
}

watch(selectedPath, (newValue) => {
  changeIdentifier.value = changeIdentifier.value + 1
  pathSegments.value = newValue.d.trim().split(/(?=[MmLlHhVvCcSsQqTtAaZz])/)
  selectedPathSegmentIndex.value = 0
})

const selectPath = (path) => {
  selectedPathSegmentIndex.value = -1
  selectedPath.value = path
  editingPath.value = path.d

  svgEditorSettings.value.drawMode = false
}

const updatePath = () => {
  if (selectedPath.value) {
    selectedPath.value.d = editingPath.value
  }
}

const selectPathSegment = (segment, index) => {
  selectedPathSegmentIndex.value = index
}

const selectedSegmentCoords = computed(() => {
  if (selectedPathSegmentIndex.value !== -1) {
    // regex find Integer or float, Integer or float
    const coords = selectedPathSegment.value.match(/\d+(\.\d+)?/g)
    return coords
  } else {
    return null
  }
})

const clearSegments = () => {
  if (selectedPath.value) {
    selectedPath.value.d = 'M100,100 Z'
    editingPath.value = 'M100,100 Z'
    pathSegments.value = ['M100,100', 'Z']
  }
}

const savePath = () => {
  if (selectedPath.value) {
    const index = paths.value.findIndex((p) => p.id === selectedPath.value.id)
    if (index !== -1) {
      paths.value[index] = { ...selectedPath.value, d: editingPath.value }
    }
  }
}

const isPointInBox = (point, box) => {
  return (
    point.x >= box.x &&
    point.x <= box.x + box.width &&
    point.y >= box.y &&
    point.y <= box.y + box.height
  )
}

const handleSelectionChange = (box, selection) => {
  console.log('Box', box)

  console.log('Hello', selectedPath.value)

  // console.log('Selected elements:', elements)
  // console.log('Selected IDs:', ids)
}

const deleteSegment = (index) => {
  pathSegments.value = pathSegments.value.filter((segment, i) => i !== index)
  editingPath.value = pathSegments.value.join('')
}

// watch(pathSegments, (newValue) => {
//   editingPath.value = newValue.join('')
// })

watch(editingPath, (newValue) => {
  if (selectedPath.value) {
    selectedPath.value.d = newValue
  }
})

const isDragging = ref(false)
const mouseDown = (event, index) => {
  isDragging.value = true
}
const mouseMove = (event, index) => {
  // update selectedPathSegment value with new coords

  if (isDragging.value === true) {
    const segment = `${calculatedMouseX.value},${calculatedMouseY.value}`
    updateSegment(segment, index)
    selectedPathSegment.value = segment
  }
}

const mouseUp = (event, index) => {
  if (isDragging.value === true) {
    isDragging.value = false
    const segment = `${calculatedMouseX.value},${calculatedMouseY.value}`
    updateSegment(segment, index)
    selectedPathSegment.value = segment
  }
}

const mouseLeave = (event, index) => {
  isDragging.value = false
}

const parseSVG = () => {
  const parser = new DOMParser()
  svgDocument.value = parser.parseFromString(
    svgInput.value || defaultSVG,
    'image/svg+xml'
  )
  const pathElements = svgDocument.value.querySelectorAll('path')
  const svgElement = svgDocument.value.querySelector('svg')

  if (svgElement) {
    if (svgElement.getAttribute('viewBox')) {
      viewBox.value = svgElement.getAttribute('viewBox') || viewBox.value
    } else {
      if (
        svgElement.getAttribute('height') &&
        svgElement.getAttribute('width')
      ) {
        const h = svgElement.getAttribute('height')
        const w = svgElement.getAttribute('width')
        viewBox.value = `0 0 ${w} ${h}`
      } else {
        viewBox.value = '0 0 2560 1440'
      }
    }

    const [, , vbWidth, vbHeight] = viewBox.value.split(' ').map(Number)
    svgWidth.value = vbWidth
    svgHeight.value = vbHeight

    viewBoxHeight.value = vbHeight
    viewBoxWidth.value = vbWidth
  }

  paths.value = Array.from(pathElements).map((pathEl, index) => {
    const parentG = pathEl.closest('g')
    return {
      id: index + 1,
      name: `Path ${index + 1}`,
      apartmentId: parentG
        ? parentG.getAttribute('data-apartment-id') ||
          parentG.getAttribute('data-building-id')
        : null,
      d: pathEl.getAttribute('d'),
      stroke: pathEl.getAttribute('stroke'),
      fill: pathEl.getAttribute('fill'),
      strokeWidth: pathEl.getAttribute('stroke-width'),
    }
  })

  updateSVGSize()
}

const updateSVGSize = () => {
  // svgWidth.value = 2560
  // svgHeight.value = 1440
  if (svgContainer.value) {
    const [, , vbWidth, vbHeight] = viewBox.value.split(' ').map(Number)

    svgHeight.value = vbHeight
    svgWidth.value = vbWidth
    viewBoxHeight.value = vbHeight
    viewBoxWidth.value = vbWidth
  }
}

const updateSelectedPathBBox = () => {
  if (selectedPath.value) {
    const svgNS = 'http://www.w3.org/2000/svg'
    const svg = document.createElementNS(svgNS, 'svg')
    svg.setAttribute('viewBox', viewBox.value)
    const path = document.createElementNS(svgNS, 'path')

    path.setAttribute('d', selectedPath.value.d)
    svg.appendChild(path)
    document.body.appendChild(svg)

    const bbox = path.getBBox()
    document.body.removeChild(svg)

    selectedPathBoundingBox.value = {
      x: bbox.x,
      y: bbox.y,
      width: bbox.width,
      height: bbox.height,
    }
  } else {
    selectedPathBoundingBox.value = null
  }
}

watch(selectedPath, updateSelectedPathBBox)
watch(() => selectedPath.value?.d, updateSelectedPathBBox)

onMounted(() => {
  parseSVG() // Load default SVG on mount
  updateSVGSize()
  window.addEventListener('resize', updateSVGSize)
})

watch(paths, updateSVGSize)

const zoomIn = () => {
  zoomLevel.value += 0.1
}
const zoomOut = () => {
  zoomLevel.value -= 0.1
}
const zoomReset = () => {
  zoomLevel.value = 1
}

const setModeToDrawing = () => {
  if (selectedPath.value) {
    svgEditorSettings.value.drawMode = true
  }
}

const setModeToSelect = () => {
  svgEditorSettings.value.drawMode = false
}

const calculatedMouseX = computed(() => {
  return ((mouseWithExtractor.x / svgWidth.value) * viewBoxWidth.value).toFixed(
    2
  )
})

const calculatedMouseY = computed(() => {
  return (
    (mouseWithExtractor.y / svgHeight.value) *
    viewBoxHeight.value
  ).toFixed(2)
})

const drawAPoint = () => {
  if (svgEditorSettings.value.drawMode) {
    let newSegment = `${calculatedMouseX.value},${calculatedMouseY.value}`

    if (editingPath.value.length === 0) {
      newSegment = `M ${newSegment}Z`
    } else {
      newSegment = `L ${calculatedMouseX.value},${calculatedMouseY.value}`
    }

    const index = selectedPathSegmentIndex.value

    if (index !== -1) {
      addSegment(newSegment, index)
    }
  }
}

const segmentDataToPath = (originalSegment, newSegment) => {
  if (pathSegments.value.length === 0 || originalSegment[0] === 'M') {
    return `M ${newSegment}`
  }

  return `L ${newSegment}`
}

const updateSegment = (segment, index) => {
  const pathToUpdate = pathSegments.value
  const originalSegment = pathToUpdate[index]
  pathToUpdate[index] = segmentDataToPath(originalSegment, segment)

  editingPath.value = pathToUpdate.join('')

  pathSegments.value = pathToUpdate.join('').split(/(?=[MmLlHhVvCcSsQqTtAaZz])/)
  updatePath()
}

const addSegment = (segment, index) => {
  const updatedPath = [
    ...pathSegments.value.slice(0, index + 1),
    segment,
    ...pathSegments.value.slice(index + 1),
  ]

  editingPath.value = updatedPath.join('')
  updatePath()

  pathSegments.value = updatedPath.join('').split(/(?=[MmLlHhVvCcSsQqTtAaZz])/)
  selectedPathSegmentIndex.value = index + 1
}

watch(selectedPathSegmentIndex, (newValue) => {
  selectedPathSegment.value = pathSegments.value[newValue]
})

const saveSVG = () => {
  const svgElement = svgDocument.value.querySelector('svg')

  let rootElementAttribs = ''

  for (const attr of svgElement.attributes) {
    rootElementAttribs += `${attr.name} = "${attr.value}" `
  }

  const svgString = `<svg ${rootElementAttribs}>`

  const svgPaths = paths.value
    .map(
      (path) =>
        `<path d="${path.d}" stroke="${path.stroke || 'black'}" fill="${
          path.fill || 'none'
        }" stroke-width="${path.strokeWidth || 2}" />`
    )
    .join('')

  const svgData = svgString + svgPaths + '</svg>'

  const blob = new Blob([svgData], { type: 'image/svg+xml' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  const filename = new Date().toISOString()
  link.download = filename + '.svg'
  link.click()
}
</script>

<style scoped>
.selected-path {
  stroke-width: 4;
  filter: drop-shadow(0 0 3px rgba(0, 0, 0, 0.5));
}

#svg-element {
  border: 1px solid #ededed;
  background-color: #fff;
  -webkit-box-shadow: -1px 3px 32px 2px rgba(0, 0, 0, 0.82);
  -moz-box-shadow: -1px 3px 32px 2px rgba(0, 0, 0, 0.82);
  box-shadow: -1px 3px 32px 2px rgba(0, 0, 0, 0.82);
}

#debug-info {
  position: fixed;
  padding: 0;
  bottom: 0;
  left: 0;
  font-size: 8px;
  background: #ccc;
  z-index: 100;
}

.svg-drawing-mode {
  cursor: crosshair;
}

.active-vertex {
  cursor: move;
}

.active-path-segment {
  background: rgba(0, 0, 0, 0.05);
}

.system-bar-group {
  padding-right: 8px;
  padding-left: 8px;
  border-right: 1px solid #777373;
}

#background-dialog {
  min-width: 220px;
  left: 4px;
  top: 40px;
}

#paths-dialog {
  min-width: 220px;
  left: 4px;
  top: 250px;
}

#editor-dialog {
  right: 3px;
  top: 50px;
}

.draggable-div {
  position: absolute;
  z-index: 1000;
  background: #ededed;
  border: 1px solid #c5c4c4;
  -webkit-box-shadow: 10px 10px 15px -6px rgba(0, 0, 0, 0.82);
  -moz-box-shadow: 10px 10px 15px -6px rgba(0, 0, 0, 0.82);
  box-shadow: 10px 10px 15px -6px rgba(0, 0, 0, 0.82);
}

.draggable-div .dialog-content {
  max-height: 70vh;
  overflow-y: scroll;
}

.dialog-title {
  display: block;
  background: #c5c4c4;
  margin: 0;
  border-radius: 5px 0 0 5px;
  padding: 10px;
  font-weight: 600;
}
</style>
