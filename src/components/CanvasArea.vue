<template>
  <div class="canvas-area" ref="containerRef">
    <v-stage
      ref="stageRef"
      :config="stageConfig"
      @wheel="handleWheel"
      @dragend="handleDragEnd"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @click="handleClick"
    >
      <v-layer>
        <template v-if="gridConfig.visible">
          <v-line v-for="(line, i) in gridLines.vertical" :key="'v-' + i" :config="{ points: [line.x, line.y1, line.x, line.y2], stroke: gridConfig.stroke, strokeWidth: gridConfig.strokeWidth }" />
          <v-line v-for="(line, i) in gridLines.horizontal" :key="'h-' + i" :config="{ points: [line.x1, line.y, line.x2, line.y], stroke: gridConfig.stroke, strokeWidth: gridConfig.strokeWidth }" />
        </template>
      </v-layer>
      
      <v-layer ref="layerRef">
        <template v-for="(shape, index) in shapes" :key="shape.attrs.id || shape.id">
          <component
            :is="getShapeComponent(shape.type)"
            :config="getShapeConfig(shape)"
            @dragend="handleShapeDragEnd($event, index)"
            @transformend="handleTransformEnd($event, index)"
          />
        </template>
        
        <template v-if="previewShape">
          <component
            :is="getShapeComponent(previewShape.type)"
            :config="getShapeConfig(previewShape)"
          />
        </template>
      </v-layer>

      <v-layer ref="connectionLayerRef">
        <v-line 
          v-for="conn in connections" 
          :key="conn.id" 
          :config="getConnectionLineConfig(conn)" 
        />
        <v-line v-if="connectionPreview" :config="connectionPreview" />
      </v-layer>

      <v-layer ref="connectionPointsLayerRef">
        <template v-if="showConnectionPoints">
          <template v-for="shape in connectableShapes" :key="'cp-' + shape.id">
            <v-circle
              v-for="(point, pIndex) in shape.connectionPoints"
              :key="'cp-' + shape.id + '-' + pIndex"
              :config="{
                x: point.x,
                y: point.y,
                radius: 8,
                fill: point.hovered ? '#4a90d9' : '#FFFFFF',
                stroke: '#4a90d9',
                strokeWidth: 2,
                name: 'connection-point-' + shape.id + '-' + pIndex
              }"
              @mouseenter="handleConnectionPointHover(shape.id, pIndex, true)"
              @mouseleave="handleConnectionPointHover(shape.id, pIndex, false)"
              @mousedown="handleConnectionPointMouseDown(shape.id, pIndex, $event)"
            />
          </template>
        </template>
      </v-layer>
      
      <v-layer ref="transformerLayerRef">
        <v-transformer
          ref="transformerRef"
          :config="transformerConfig"
        />
      </v-layer>
    </v-stage>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, computed, watch, nextTick } from 'vue'
import { useToolStore, type ToolType } from '../store/toolStore'
import { useHistoryStore } from '../store/historyStore'
import { usePageStore } from '../store/pageStore'

interface Shape {
  id: string
  type: ToolType
  attrs: Record<string, any>
}

interface Point {
  x: number
  y: number
}

interface ConnectionPoint {
  x: number
  y: number
  hovered: boolean
}

interface ConnectableShape {
  id: string
  connectionPoints: ConnectionPoint[]
}

interface Connection {
  id: string
  fromShapeId: string
  fromPointIndex: number
  toShapeId: string
  toPointIndex: number
  stroke: string
  strokeWidth: number
}

const toolStore = useToolStore()
const historyStore = useHistoryStore()
const pageStore = usePageStore()
const containerRef = ref<HTMLElement | null>(null)
const stageRef = ref<any>(null)
const layerRef = ref<any>(null)
const transformerRef = ref<any>(null)
const transformerLayerRef = ref<any>(null)

const themeColors = {
  white: { canvasBg: '#FFFFFF', containerBg: '#E0E0E0', gridColor: '#E8E8E8' },
  black: { canvasBg: '#2D5A27', containerBg: '#1E3D1A', gridColor: '#3D6B35' },
  dark: { canvasBg: '#1E1E1E', containerBg: '#121212', gridColor: '#2D2D2D' }
}

const currentThemeColors = computed(() => themeColors[toolStore.state.theme])

const stageConfig = reactive({ width: 800, height: 600 })
const gridConfig = reactive({ visible: false, spacing: 20, stroke: '#E8E8E8', strokeWidth: 1 })

const gridLines = computed(() => {
  if (!gridConfig.visible) return { vertical: [], horizontal: [] }
  const vLines = []
  const hLines = []
  const spacing = gridConfig.spacing
  for (let x = 0; x <= stageConfig.width; x += spacing) {
    vLines.push({ x, y1: 0, y2: stageConfig.height })
  }
  for (let y = 0; y <= stageConfig.height; y += spacing) {
    hLines.push({ x1: 0, x2: stageConfig.width, y })
  }
  return { vertical: vLines, horizontal: hLines }
})

const transformerConfig = reactive({
  borderStroke: '#2196F3', borderStrokeWidth: 2,
  anchorFill: '#FFFFFF', anchorStroke: '#2196F3', anchorSize: 10, rotateAnchorOffset: 30,
  enabledAnchors: ['top-left', 'top-right', 'bottom-left', 'bottom-right', 'middle-left', 'middle-right', 'top-center', 'bottom-center'],
  boundBoxFunc: (oldBox: any, newBox: any) => {
    if (newBox.width < 5 || newBox.height < 5) return oldBox
    return newBox
  }
})

const shapes = ref<Shape[]>([])
const selectedShapeIndex = ref<number | null>(null)
const isDrawing = ref(false)
const startPoint = ref<Point | null>(null)
const currentPoint = ref<Point | null>(null)
const previewShape = ref<Shape | null>(null)
const penPoints = ref<Point[]>([])
const compassState = ref<{ center: Point | null; radius: number; step: number }>({ center: null, radius: 0, step: 1 })

const connections = ref<Connection[]>([])
const connectionPreview = ref<any>(null)
const showConnectionPoints = ref(false)
const connectableShapes = ref<ConnectableShape[]>([])
const connectingFrom = ref<{ shapeId: string; pointIndex: number } | null>(null)
const currentHoveredPoint = ref<{ shapeId: string; pointIndex: number } | null>(null)

const connectionLayerRef = ref<any>(null)
const connectionPointsLayerRef = ref<any>(null)

const generateId = () => `shape_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`

const getShapeComponent = (type: ToolType): string => {
  const map: Record<ToolType, string> = {
    select: 'v-rect', line: 'v-line', rect: 'v-rect', circle: 'v-circle',
    triangle: 'v-line', polygon: 'v-regular-polygon', arrow: 'v-arrow',
    pen: 'v-line', eraser: 'v-rect', ruler: 'v-group', protractor: 'v-group',
    compass: 'v-group', setsquare: 'v-group', coordinate: 'v-group',
    function: 'v-group', numberLine: 'v-group', geometryMark: 'v-group',
    forceArrow: 'v-group', pulley: 'v-group', spring: 'v-group',
    incline: 'v-group', lever: 'v-group', magnetic: 'v-group',
    beaker: 'v-group', flask: 'v-group', testTube: 'v-group',
    alcoholLamp: 'v-group', molecule: 'v-group', latex: 'v-group', chemFormula: 'v-group',
    connect: 'v-line'
  }
  return map[type] || 'v-rect'
}

const getShapeConfig = (shape: Shape): Record<string, any> => {
  const config = { ...shape.attrs, id: shape.attrs.id || shape.id }
  if (shape.attrs.isSelected) config.draggable = true
  return config
}

const getConnectableShapeTypes = (): ToolType[] => {
  return ['rect', 'circle', 'triangle', 'polygon', 'line', 'arrow']
}

const getConnectionPointsForShape = (shape: Shape): ConnectionPoint[] => {
  const points: ConnectionPoint[] = []
  const attrs = shape.attrs
  const stage = stageRef.value?.getNode()
  const shapeId = attrs.id || shape.id
  const node = stage?.findOne('#' + shapeId)
  
  switch (shape.type) {
    case 'rect':
      let rx = attrs.x || 0
      let ry = attrs.y || 0
      let rw = attrs.width || 0
      let rh = attrs.height || 0
      
      if (node) {
        rx = node.x()
        ry = node.y()
        rw = node.width() * (node.scaleX() || 1)
        rh = node.height() * (node.scaleY() || 1)
      }
      
      points.push(
        { x: rx + rw / 2, y: ry, hovered: false },
        { x: rx + rw, y: ry + rh / 2, hovered: false },
        { x: rx + rw / 2, y: ry + rh, hovered: false },
        { x: rx, y: ry + rh / 2, hovered: false }
      )
      break
    case 'circle':
      let cx = attrs.x || 0
      let cy = attrs.y || 0
      let cr = attrs.radius || 0
      
      if (node) {
        cx = node.x()
        cy = node.y()
        cr = node.radius() * (node.scaleX() || 1)
      }
      
      points.push(
        { x: cx, y: cy - cr, hovered: false },
        { x: cx + cr, y: cy, hovered: false },
        { x: cx, y: cy + cr, hovered: false },
        { x: cx - cr, y: cy, hovered: false }
      )
      break
    case 'triangle':
      if (attrs.points && attrs.points.length >= 6) {
        const p = attrs.points
        points.push(
          { x: (p[0] + p[2]) / 2, y: (p[1] + p[3]) / 2, hovered: false },
          { x: (p[2] + p[4]) / 2, y: (p[3] + p[5]) / 2, hovered: false },
          { x: (p[4] + p[0]) / 2, y: (p[5] + p[1]) / 2, hovered: false }
        )
      }
      break
    case 'polygon':
      let px = attrs.x || 0
      let py = attrs.y || 0
      let pr = attrs.radius || 0
      const sides = attrs.sides || 5
      
      if (node) {
        px = node.x()
        py = node.y()
        pr = node.radius() * (node.scaleX() || 1)
      }
      
      for (let i = 0; i < sides; i++) {
        const angle = (i * 2 * Math.PI / sides) - Math.PI / 2
        points.push({
          x: px + pr * Math.cos(angle),
          y: py + pr * Math.sin(angle),
          hovered: false
        })
      }
      break
    case 'line':
    case 'arrow':
      if (attrs.points && attrs.points.length >= 4) {
        points.push(
          { x: attrs.points[0], y: attrs.points[1], hovered: false },
          { x: attrs.points[2], y: attrs.points[3], hovered: false }
        )
      }
      break
  }
  
  return points
}

const updateConnectableShapes = () => {
  const connectableTypes = getConnectableShapeTypes()
  connectableShapes.value = shapes.value
    .filter(shape => connectableTypes.includes(shape.type))
    .map(shape => ({
      id: shape.attrs.id || shape.id,
      connectionPoints: getConnectionPointsForShape(shape)
    }))
  console.log('updateConnectableShapes:', connectableShapes.value.length, 'shapes')
  console.log('connectableShapes:', connectableShapes.value.map(s => ({ id: s.id, points: s.connectionPoints.length })))
}

const getConnectionLineConfig = (conn: Connection): any => {
  const fromShape = connectableShapes.value.find(s => s.id === conn.fromShapeId)
  const toShape = connectableShapes.value.find(s => s.id === conn.toShapeId)
  
  if (!fromShape || !toShape) {
    console.warn('getConnectionLineConfig: Missing shape', conn.id, 'from:', conn.fromShapeId, 'to:', conn.toShapeId)
    return { points: [], stroke: conn.stroke, strokeWidth: conn.strokeWidth }
  }
  
  const fromPoint = fromShape.connectionPoints[conn.fromPointIndex]
  const toPoint = toShape.connectionPoints[conn.toPointIndex]
  
  if (!fromPoint || !toPoint) {
    console.warn('getConnectionLineConfig: Missing point', 'fromIndex:', conn.fromPointIndex, 'toIndex:', conn.toPointIndex)
    return { points: [], stroke: conn.stroke, strokeWidth: conn.strokeWidth }
  }
  
  return {
    id: conn.id,
    points: [fromPoint.x, fromPoint.y, toPoint.x, toPoint.y],
    stroke: conn.stroke,
    strokeWidth: conn.strokeWidth,
    lineCap: 'round',
    lineJoin: 'round'
  }
}

const updateConnectionLines = () => {
  updateConnectableShapes()
  const layer = connectionLayerRef.value?.getNode()
  if (layer) {
    layer.batchDraw()
  }
}

let isOnConnectionPoint = false

const handleConnectionPointHover = (shapeId: string, pointIndex: number, isHovered: boolean) => {
  const shape = connectableShapes.value.find(s => s.id === shapeId)
  if (shape && shape.connectionPoints[pointIndex]) {
    shape.connectionPoints[pointIndex].hovered = isHovered
  }
  isOnConnectionPoint = isHovered
  if (isHovered) {
    currentHoveredPoint.value = { shapeId, pointIndex }
  } else {
    currentHoveredPoint.value = null
  }
}

const handleConnectionPointMouseDown = (shapeId: string, pointIndex: number, e: any) => {
  e.evt.stopPropagation()
  isOnConnectionPoint = true
  
  console.log('=== handleConnectionPointMouseDown ===')
  console.log('shapeId:', shapeId, 'pointIndex:', pointIndex)
  console.log('connectingFrom before:', JSON.stringify(connectingFrom.value))
  
  if (connectingFrom.value === null) {
    connectingFrom.value = { shapeId, pointIndex }
    const shape = connectableShapes.value.find(s => s.id === shapeId)
    if (shape) {
      const point = shape.connectionPoints[pointIndex]
      connectionPreview.value = {
        points: [point.x, point.y, point.x, point.y],
        stroke: toolStore.state.strokeColor,
        strokeWidth: toolStore.state.strokeWidth,
        dash: [5, 5]
      }
      console.log('Started connection from:', point)
    }
  } else {
    console.log('Already connecting from another point, canceling previous connection')
    // 如果已经在连接过程中，点击另一个连接点会取消之前的连接并开始新的连接
    connectingFrom.value = { shapeId, pointIndex }
    const shape = connectableShapes.value.find(s => s.id === shapeId)
    if (shape) {
      const point = shape.connectionPoints[pointIndex]
      connectionPreview.value = {
        points: [point.x, point.y, point.x, point.y],
        stroke: toolStore.state.strokeColor,
        strokeWidth: toolStore.state.strokeWidth,
        dash: [5, 5]
      }
      console.log('Started new connection from:', point)
    }
  }
  console.log('connectingFrom after:', JSON.stringify(connectingFrom.value))
}

const handleConnectionMouseMove = () => {
  if (connectingFrom.value && connectionPreview.value) {
    const pos = getRelativePointerPosition()
    if (pos) {
      const shape = connectableShapes.value.find(s => s.id === connectingFrom.value!.shapeId)
      if (shape) {
        const point = shape.connectionPoints[connectingFrom.value.pointIndex]
        connectionPreview.value.points = [point.x, point.y, pos.x, pos.y]
      }
    }
  }
}

const handleConnectionMouseUp = () => {
  if (connectingFrom.value) {
    // 如果当前悬停在连接点上，自动创建连接
    if (currentHoveredPoint.value && 
        (currentHoveredPoint.value.shapeId !== connectingFrom.value.shapeId || 
         currentHoveredPoint.value.pointIndex !== connectingFrom.value.pointIndex)) {
      const newConnection: Connection = {
        id: generateId(),
        fromShapeId: connectingFrom.value.shapeId,
        fromPointIndex: connectingFrom.value.pointIndex,
        toShapeId: currentHoveredPoint.value.shapeId,
        toPointIndex: currentHoveredPoint.value.pointIndex,
        stroke: toolStore.state.strokeColor,
        strokeWidth: toolStore.state.strokeWidth
      }
      connections.value.push(newConnection)
      
      nextTick(() => {
        updateConnectableShapes()
        const layer = connectionLayerRef.value?.getNode()
        if (layer) {
          layer.batchDraw()
        }
      })
      
      historyStore.saveState(shapes.value)
    }
    connectingFrom.value = null
    connectionPreview.value = null
    currentHoveredPoint.value = null
  }
}

const updateSize = () => {
  if (containerRef.value) {
    stageConfig.width = containerRef.value.clientWidth
    stageConfig.height = containerRef.value.clientHeight
  }
}

const getRelativePointerPosition = (): Point | null => {
  const stage = stageRef.value?.getNode()
  if (!stage) return null
  const transform = stage.getAbsoluteTransform().copy()
  transform.invert()
  const pos = stage.getPointerPosition()
  if (!pos) return null
  return transform.point(pos)
}

const handleWheel = (e: any) => {
  e.evt.preventDefault()
  const stage = stageRef.value?.getNode()
  if (!stage) return
  const oldScale = stage.scaleX()
  const pointer = stage.getPointerPosition()
  if (!pointer) return
  const scaleBy = 1.1
  const direction = e.evt.deltaY > 0 ? -1 : 1
  const newScale = direction > 0 ? oldScale * scaleBy : oldScale / scaleBy
  if (newScale < 0.1 || newScale > 10) return
  const mousePointTo = { x: (pointer.x - stage.x()) / oldScale, y: (pointer.y - stage.y()) / oldScale }
  const newPos = { x: pointer.x - mousePointTo.x * newScale, y: pointer.y - mousePointTo.y * newScale }
  stage.scale({ x: newScale, y: newScale })
  stage.position(newPos)
}

const handleDragEnd = () => {}

const handleMouseDown = (e: any) => {
  const stage = stageRef.value?.getNode()
  if (!stage) return
  const tool = toolStore.state.currentTool
  const pos = getRelativePointerPosition()
  if (!pos) return
  
  const transformer = transformerRef.value?.getNode()
  if (e.target === transformer || (e.target !== stage && tool === 'select')) {
  } else if (e.target === stage && tool !== 'select') {
    if (!isOnConnectionPoint) {
      deselectShape()
    }
  }
  
  if (tool === 'connect') {
    isOnConnectionPoint = false
    return
  }
  
  if (tool === 'select' || tool === 'eraser') return
  isDrawing.value = true
  startPoint.value = pos
  currentPoint.value = pos
  updatePreviewShape()
}

const handleMouseMove = (e: any) => {
  const tool = toolStore.state.currentTool
  const stage = stageRef.value?.getNode()
  if (!stage) return
  const pos = getRelativePointerPosition()
  if (!pos) return

  if (tool === 'connect') {
    handleConnectionMouseMove()
    return
  }

  if (tool === 'eraser') {
    const shape = e?.target
    if (shape && shape !== stage && shapes.value.length > 0) {
      const shapeIndex = shapes.value.findIndex(s => stage.findOne('#' + s.id) === shape)
      if (shapeIndex !== -1) { shapes.value.splice(shapeIndex, 1); deselectShape() }
    }
    return
  }

  if (!isDrawing.value || !startPoint.value) return
  currentPoint.value = pos
  updatePreviewShape()
}

const handleMouseUp = () => {
  const tool = toolStore.state.currentTool
  if (tool === 'connect') {
    // 无论是否在连接点上，都处理连接创建
    handleConnectionMouseUp()
    isOnConnectionPoint = false
    return
  }
  if (tool === 'eraser') return
  if (!isDrawing.value || !startPoint.value || !currentPoint.value) {
    isDrawing.value = false; return
  }
  finishDrawing()
}

const handleClick = (e: any) => {
  const tool = toolStore.state.currentTool
  const stage = stageRef.value?.getNode()
  if (!stage) return

  let clickedShape = e.target
  const transformer = transformerRef.value?.getNode()
  
  if (clickedShape === transformer && transformer?.nodes().length > 0) {
    clickedShape = transformer.nodes()[0]
  }
  
  if (clickedShape === stage || clickedShape === transformer) { 
    deselectShape()
    return
  }
  
  const clickedId = clickedShape?.attrs?.id || (typeof clickedShape?.id === 'function' ? clickedShape.id() : null)
  console.log('Click:', clickedShape?.className, 'id:', clickedId)
  console.log('Available shapes:', shapes.value.map(s => s.attrs.id || s.id))

  if (tool === 'eraser') {
    if (clickedShape && shapes.value.length > 0) {
      const shapeIndex = shapes.value.findIndex(s => (s.attrs.id || s.id) === clickedId)
      if (shapeIndex !== -1) { shapes.value.splice(shapeIndex, 1); deselectShape() }
    }
    return
  }

  if (tool === 'connect') {
    return
  }

  if (tool === 'select') {
    const shapeIndex = shapes.value.findIndex(s => (s.attrs.id || s.id) === clickedId)
    console.log('shapeIndex:', shapeIndex)
    if (shapeIndex !== -1) {
      selectShape(shapeIndex)
    }
  }
}

const updatePreviewShape = () => {
  if (!startPoint.value || !currentPoint.value) return
  const tool = toolStore.state.currentTool
  const { strokeColor, fillColor, strokeWidth, fillEnabled } = toolStore.state
  const start = startPoint.value
  const end = currentPoint.value
  const w = end.x - start.x, h = end.y - start.y
  let shape: Shape | null = null

  switch (tool) {
    case 'line':
      shape = { id: 'preview', type: 'line', attrs: { points: [start.x, start.y, end.x, end.y], stroke: strokeColor, strokeWidth, lineCap: 'round' } }
      break
    case 'rect':
      shape = { id: 'preview', type: 'rect', attrs: { x: w > 0 ? start.x : end.x, y: h > 0 ? start.y : end.y, width: Math.abs(w), height: Math.abs(h), stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent' } }
      break
    case 'circle':
      const r = Math.max(Math.abs(end.x - start.x), Math.abs(end.y - start.y)) / 2
      shape = { id: 'preview', type: 'circle', attrs: { x: (start.x + end.x) / 2, y: (start.y + end.y) / 2, radius: r, stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent' } }
      break
    case 'triangle':
      shape = { id: 'preview', type: 'triangle', attrs: { points: [start.x, start.y + Math.abs(h), end.x, end.y + Math.abs(h), start.x + Math.abs(w) / 2, start.y], stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent', closed: true } }
      break
    case 'arrow':
      shape = { id: 'preview', type: 'arrow', attrs: { points: [start.x, start.y, end.x, end.y], stroke: strokeColor, strokeWidth, pointerLength: 10, pointerWidth: 10, fill: strokeColor } }
      break
    case 'pen':
      penPoints.value.push(end)
      shape = { id: 'preview', type: 'pen', attrs: { points: penPoints.value.flatMap(p => [p.x, p.y]), stroke: strokeColor, strokeWidth, lineCap: 'round', tension: 0.5 } }
      break
    case 'polygon':
      const centerX = (start.x + end.x) / 2, centerY = (start.y + end.y) / 2
      const radius = Math.max(Math.abs(end.x - start.x), Math.abs(end.y - start.y)) / 2
      shape = { id: 'preview', type: 'polygon', attrs: { x: centerX, y: centerY, sides: toolStore.state.polygonSides, radius, stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent' } }
      break
  }
  previewShape.value = shape
}

const finishDrawing = () => {
  if (!startPoint.value || !currentPoint.value) { resetDrawingState(); return }
  const tool = toolStore.state.currentTool
  const { strokeColor, fillColor, strokeWidth, fillEnabled } = toolStore.state
  const start = startPoint.value
  const end = currentPoint.value
  const minDistance = 5
  if (Math.abs(end.x - start.x) < minDistance && Math.abs(end.y - start.y) < minDistance && tool !== 'pen') { resetDrawingState(); return }

  let shape: Shape | null = null
  const w = end.x - start.x, h = end.y - start.y
  const shapeId = generateId()

  switch (tool) {
    case 'line':
      shape = { id: shapeId, type: 'line', attrs: { id: shapeId, points: [start.x, start.y, end.x, end.y], stroke: strokeColor, strokeWidth, lineCap: 'round' } }
      break
    case 'rect':
      shape = { id: shapeId, type: 'rect', attrs: { id: shapeId, x: w > 0 ? start.x : end.x, y: h > 0 ? start.y : end.y, width: Math.abs(w), height: Math.abs(h), stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent' } }
      break
    case 'circle':
      const r = Math.max(Math.abs(w), Math.abs(h)) / 2
      shape = { id: shapeId, type: 'circle', attrs: { id: shapeId, x: (start.x + end.x) / 2, y: (start.y + end.y) / 2, radius: r, stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent' } }
      break
    case 'triangle':
      shape = { id: shapeId, type: 'triangle', attrs: { id: shapeId, points: [start.x, start.y + Math.abs(h), end.x, end.y + Math.abs(h), start.x + Math.abs(w) / 2, start.y], stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent', closed: true } }
      break
    case 'arrow':
      shape = { id: shapeId, type: 'arrow', attrs: { id: shapeId, points: [start.x, start.y, end.x, end.y], stroke: strokeColor, strokeWidth, pointerLength: 10, pointerWidth: 10, fill: strokeColor } }
      break
    case 'pen':
      if (penPoints.value.length > 1) shape = { id: shapeId, type: 'pen', attrs: { id: shapeId, points: penPoints.value.flatMap(p => [p.x, p.y]), stroke: strokeColor, strokeWidth, lineCap: 'round', tension: 0.5 } }
      break
    case 'polygon':
      const centerX = (start.x + end.x) / 2, centerY = (start.y + end.y) / 2, radius = Math.max(Math.abs(w), Math.abs(h)) / 2
      if (radius > 0) shape = { id: shapeId, type: 'polygon', attrs: { id: shapeId, x: centerX, y: centerY, sides: toolStore.state.polygonSides, radius, stroke: strokeColor, strokeWidth, fill: fillEnabled ? fillColor : 'transparent' } }
      break
  }

  if (shape) { shapes.value.push(shape); historyStore.saveState(shapes.value); pageStore.setCurrentPageShapes([...shapes.value]); updateConnectableShapes() }
  resetDrawingState()
}

const resetDrawingState = () => {
  isDrawing.value = false; startPoint.value = null; currentPoint.value = null; previewShape.value = null; penPoints.value = []
}

const selectShape = (index: number) => {
  console.log('=== selectShape called ===', index)
  const transformer = transformerRef.value?.getNode()
  const transformerLayer = transformerLayerRef.value?.getNode()
  const layer = layerRef.value?.getNode()
  
  console.log('transformer:', transformer, 'layer:', layer)
  
  // 先清空 transformer
  if (transformer) { 
    transformer.nodes([])
    console.log('Cleared transformer')
  }
  
  // 清除之前的选择状态
  if (selectedShapeIndex.value !== null && shapes.value[selectedShapeIndex.value]) {
    shapes.value[selectedShapeIndex.value].attrs.isSelected = false
    console.log('Cleared previous selection')
  }
  
  selectedShapeIndex.value = index
  shapes.value[index].attrs.isSelected = true
  console.log('Set isSelected = true for index:', index)
  window.dispatchEvent(new CustomEvent('whiteboard:shape-selected', { detail: shapes.value[index].attrs }))
  
  const stage = stageRef.value?.getNode()
  if (!stage || !transformer || !layer) {
    console.log('Missing stage, transformer or layer')
    return
  }
  
  // 使用 attrs.id 来查找
  const shapeId = shapes.value[index].attrs.id || shapes.value[index].id
  console.log('Looking for shape with id:', shapeId)
  
  const shapeNode = stage.findOne('#' + shapeId)
  console.log('Found shapeNode:', shapeNode, 'layer:', shapeNode?.getLayer())
  
  if (shapeNode) {
    // 直接设置节点的 draggable
    shapeNode.draggable(true)
    transformer.nodes([shapeNode])
    console.log('Added node to transformer, draggable:', shapeNode.draggable(), 'transformer nodes:', transformer.nodes())
    // 刷新所有 layer
    layer.batchDraw()
    transformerLayer?.batchDraw()
    console.log('Done batchDraw')
  }
}

const deselectShape = () => {
  console.log('=== deselectShape called ===')
  const stage = stageRef.value?.getNode()
  
  if (selectedShapeIndex.value !== null && shapes.value[selectedShapeIndex.value]) {
    const shapeId = shapes.value[selectedShapeIndex.value].attrs.id || shapes.value[selectedShapeIndex.value].id
    const shapeNode = stage?.findOne('#' + shapeId)
    if (shapeNode) {
      shapeNode.draggable(false)
      console.log('Set draggable = false for:', shapeId)
    }
    shapes.value[selectedShapeIndex.value].attrs.isSelected = false
    console.log('Set isSelected = false for index:', selectedShapeIndex.value)
  }
  selectedShapeIndex.value = null
  window.dispatchEvent(new CustomEvent('whiteboard:shape-deselected'))
  nextTick(() => {
    const transformer = transformerRef.value?.getNode()
    const transformerLayer = transformerLayerRef.value?.getNode()
    if (transformer) { 
      transformer.nodes([]); 
      console.log('Cleared transformer in deselect')
      transformer.getLayer()?.batchDraw() 
    }
  })
}

const handleShapeDragEnd = (e: any, index: number) => {
  const shape = shapes.value[index]
  const node = e.target
  
  if (shape && node) {
    if (shape.type === 'rect') {
      shape.attrs.x = node.x()
      shape.attrs.y = node.y()
    } else if (shape.type === 'circle' || shape.type === 'polygon') {
      shape.attrs.x = node.x()
      shape.attrs.y = node.y()
    }
    console.log('Shape dragged, new position:', shape.attrs.x, shape.attrs.y)
  }
  
  historyStore.saveState(shapes.value)
  updateConnectableShapes()
  
  nextTick(() => {
    const layer = connectionLayerRef.value?.getNode()
    if (layer) layer.batchDraw()
  })
}

const handleTransformEnd = () => {
  historyStore.saveState(shapes.value)
  updateConnectableShapes()
  
  nextTick(() => {
    const layer = connectionLayerRef.value?.getNode()
    if (layer) layer.batchDraw()
  })
}

const deleteSelectedShape = () => {
  if (selectedShapeIndex.value !== null) { shapes.value.splice(selectedShapeIndex.value, 1); deselectShape(); historyStore.saveState(shapes.value) }
}

const getSelectedShapes = (): Shape[] => {
  if (selectedShapeIndex.value === null) return []
  return [shapes.value[selectedShapeIndex.value]]
}

const copySelectedShapes = () => { const selected = getSelectedShapes(); if (selected.length > 0) historyStore.copyShapes(selected) }
const cutSelectedShapes = () => { const selected = getSelectedShapes(); if (selected.length > 0) { historyStore.cutShapes(selected); deleteSelectedShape() } }
const pasteShapes = () => {
  const pasted = historyStore.pasteShapes()
  if (pasted.length > 0) { shapes.value.push(...pasted); const newIndex = shapes.value.length - pasted.length; selectShape(newIndex); historyStore.saveState(shapes.value) }
}
const duplicateSelectedShapes = () => {
  const selected = getSelectedShapes()
  if (selected.length > 0) { const duplicated = historyStore.duplicateShapes(selected); shapes.value.push(...duplicated); const newIndex = shapes.value.length - duplicated.length; selectShape(newIndex); historyStore.saveState(shapes.value) }
}

const rotateClockwise = () => {
  if (selectedShapeIndex.value === null) return
  const shape = shapes.value[selectedShapeIndex.value]
  const stage = stageRef.value?.getNode()
  const node = stage?.findOne('#' + shape.id)
  if (node) { const r = node.rotation(); node.rotation(r + 15); shape.attrs.rotation = r + 15; historyStore.saveState(shapes.value) }
}
const rotateCounterClockwise = () => {
  if (selectedShapeIndex.value === null) return
  const shape = shapes.value[selectedShapeIndex.value]
  const stage = stageRef.value?.getNode()
  const node = stage?.findOne('#' + shape.id)
  if (node) { const r = node.rotation(); node.rotation(r - 15); shape.attrs.rotation = r - 15; historyStore.saveState(shapes.value) }
}

const flipHorizontal = () => {
  if (selectedShapeIndex.value === null) return
  const shape = shapes.value[selectedShapeIndex.value]
  const stage = stageRef.value?.getNode()
  const node = stage?.findOne('#' + shape.id)
  if (node) { const sx = node.scaleX(); node.scaleX(-sx); shape.attrs.scaleX = -sx; historyStore.saveState(shapes.value) }
}
const flipVertical = () => {
  if (selectedShapeIndex.value === null) return
  const shape = shapes.value[selectedShapeIndex.value]
  const stage = stageRef.value?.getNode()
  const node = stage?.findOne('#' + shape.id)
  if (node) { const sy = node.scaleY(); node.scaleY(-sy); shape.attrs.scaleY = -sy; historyStore.saveState(shapes.value) }
}

const moveLayerUp = () => {
  if (selectedShapeIndex.value === null || selectedShapeIndex.value >= shapes.value.length - 1) return
  const i = selectedShapeIndex.value
  const t = shapes.value[i]; shapes.value[i] = shapes.value[i + 1]; shapes.value[i + 1] = t; selectedShapeIndex.value = i + 1; historyStore.saveState(shapes.value)
}
const moveLayerDown = () => {
  if (selectedShapeIndex.value === null || selectedShapeIndex.value <= 0) return
  const i = selectedShapeIndex.value
  const t = shapes.value[i]; shapes.value[i] = shapes.value[i - 1]; shapes.value[i - 1] = t; selectedShapeIndex.value = i - 1; historyStore.saveState(shapes.value)
}
const moveToTop = () => {
  if (selectedShapeIndex.value === null || selectedShapeIndex.value >= shapes.value.length - 1) return
  const i = selectedShapeIndex.value
  const s = shapes.value.splice(i, 1)[0]; shapes.value.push(s); selectedShapeIndex.value = shapes.value.length - 1; historyStore.saveState(shapes.value)
}
const moveToBottom = () => {
  if (selectedShapeIndex.value === null || selectedShapeIndex.value <= 0) return
  const i = selectedShapeIndex.value
  const s = shapes.value.splice(i, 1)[0]; shapes.value.unshift(s); selectedShapeIndex.value = 0; historyStore.saveState(shapes.value)
}

const undo = () => {
  if (!historyStore.canUndo.value) return
  const newShapes = historyStore.undo(shapes.value)
  shapes.value = newShapes
  deselectShape()
}
const redo = () => {
  if (!historyStore.canRedo.value) return
  const newShapes = historyStore.redo(shapes.value)
  shapes.value = newShapes
  deselectShape()
}

const loadCurrentPageShapes = () => { shapes.value = [...pageStore.getCurrentPageShapes()] }
const saveCurrentPageShapes = () => { pageStore.setCurrentPageShapes([...shapes.value]) }

watch(() => pageStore.state.currentPage, () => { saveCurrentPageShapes(); loadCurrentPageShapes(); deselectShape() })

const handleKeyDown = (e: KeyboardEvent) => {
  if ((e.target as HTMLElement).tagName === 'INPUT' || (e.target as HTMLElement).tagName === 'TEXTAREA') return
  const key = e.key.toLowerCase()
  const ctrlKey = e.ctrlKey || e.metaKey

  if (ctrlKey) {
    switch (key) {
      case 'z': e.preventDefault(); undo(); break
      case 'y': e.preventDefault(); redo(); break
      case 'c': e.preventDefault(); copySelectedShapes(); break
      case 'x': e.preventDefault(); cutSelectedShapes(); break
      case 'v': e.preventDefault(); pasteShapes(); break
      case 'd': e.preventDefault(); duplicateSelectedShapes(); break
      case ']': e.preventDefault(); rotateClockwise(); break
      case '[': e.preventDefault(); rotateCounterClockwise(); break
    }
    return
  }

  switch (key) {
    case 'v': toolStore.setTool('select'); break
    case 'c': toolStore.setTool('connect'); break
    case 'l': toolStore.setTool('line'); break
    case 'r': toolStore.setTool('rect'); break
    case 'e': toolStore.setTool('circle'); break
    case 't': toolStore.setTool('triangle'); break
    case 'a': toolStore.setTool('arrow'); break
    case 'b': toolStore.setTool('pen'); break
    case 'x': toolStore.setTool('eraser'); break
    case '1': toolStore.setTool('ruler'); break
    case '2': toolStore.setTool('protractor'); break
    case '3': toolStore.setTool('compass'); break
    case '4': toolStore.setTool('setsquare'); break
    case 'delete': case 'backspace': deleteSelectedShape(); break
  }
}

const handleUndoEvent = () => undo()
const handleRedoEvent = () => redo()
const handleRotateCWEvent = () => rotateClockwise()
const handleRotateCCWEvent = () => rotateCounterClockwise()
const handleFlipHEvent = () => flipHorizontal()
const handleFlipVEvent = () => flipVertical()
const handleLayerUpEvent = () => moveLayerUp()
const handleLayerDownEvent = () => moveLayerDown()
const handleLayerTopEvent = () => moveToTop()
const handleLayerBottomEvent = () => moveToBottom()

const updateSelectedShapeProperty = (type: string, value: any) => {
  if (selectedShapeIndex.value === null) return
  const stage = stageRef.value?.getNode()
  if (!stage) return
  const shape = shapes.value[selectedShapeIndex.value]
  const node = stage.findOne('#' + shape.id)
  if (!node) return
  switch (type) {
    case 'fill': shape.attrs.fill = value; node.fill(value); break
    case 'fillOpacity': shape.attrs.fillOpacity = value; node.fillOpacity(value); break
    case 'stroke': shape.attrs.stroke = value; node.stroke(value); if (shape.type === 'arrow') { node.fill(value); shape.attrs.fill = value } break
    case 'strokeWidth': shape.attrs.strokeWidth = value; node.strokeWidth(value); break
  }
  node.getLayer()?.batchDraw()
  historyStore.saveState(shapes.value)
}

const handleUpdateShapeEvent = (e: CustomEvent) => { const { type, value } = e.detail; updateSelectedShapeProperty(type, value) }
const handleAlignEvent = (e: CustomEvent) => {}

const handleExportPNG = async (event: Event) => {
  const customEvent = event as CustomEvent
  const resolution = customEvent.detail?.resolution || 1
  const stage = stageRef.value?.getNode()
  if (!stage) return
  const dataURL = stage.toDataURL({ pixelRatio: resolution, mimeType: 'image/png' })
  const link = document.createElement('a')
  link.download = `白板页面${pageStore.currentPageNumber.value}.png`
  link.href = dataURL
  document.body.appendChild(link); link.click(); document.body.removeChild(link)
}

const handleProjectLoaded = () => { loadCurrentPageShapes(); deselectShape() }

const applyThemeColors = () => {
  const colors = currentThemeColors.value
  const canvasArea = containerRef.value
  const konvaContent = canvasArea?.querySelector('.konvajs-content') as HTMLElement
  if (canvasArea) canvasArea.style.backgroundColor = colors.containerBg
  if (konvaContent) konvaContent.style.backgroundColor = colors.canvasBg
  gridConfig.stroke = colors.gridColor
}

const updateGridConfig = () => { gridConfig.visible = toolStore.state.gridEnabled; gridConfig.spacing = toolStore.state.gridSpacing; gridConfig.stroke = currentThemeColors.value.gridColor }

watch(() => toolStore.state.currentTool, (newTool) => {
  if (newTool !== 'select') deselectShape()
  const stage = stageRef.value?.getNode()
  if (stage) stage.draggable(newTool === 'select')
  
  if (newTool === 'connect') {
    console.log('Switched to connect tool')
    showConnectionPoints.value = true
    nextTick(() => {
      updateConnectableShapes()
      console.log('connectableShapes updated:', connectableShapes.value.length)
    })
  } else {
    showConnectionPoints.value = false
    connectingFrom.value = null
    connectionPreview.value = null
  }
})

watch(shapes, () => {
  if (showConnectionPoints.value || connections.value.length > 0) {
    updateConnectableShapes()
    nextTick(() => {
      const layer = connectionLayerRef.value?.getNode()
      if (layer) layer.batchDraw()
    })
  }
}, { deep: true })

onMounted(() => {
  loadCurrentPageShapes()
  updateSize()
  applyThemeColors()
  updateGridConfig()
  window.addEventListener('resize', updateSize)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('whiteboard:undo', handleUndoEvent)
  window.addEventListener('whiteboard:redo', handleRedoEvent)
  window.addEventListener('whiteboard:rotate-cw', handleRotateCWEvent)
  window.addEventListener('whiteboard:rotate-ccw', handleRotateCCWEvent)
  window.addEventListener('whiteboard:flip-h', handleFlipHEvent)
  window.addEventListener('whiteboard:flip-v', handleFlipVEvent)
  window.addEventListener('whiteboard:layer-up', handleLayerUpEvent)
  window.addEventListener('whiteboard:layer-down', handleLayerDownEvent)
  window.addEventListener('whiteboard:layer-top', handleLayerTopEvent)
  window.addEventListener('whiteboard:layer-bottom', handleLayerBottomEvent)
  window.addEventListener('whiteboard:align', handleAlignEvent as EventListener)
  window.addEventListener('whiteboard:update-shape', handleUpdateShapeEvent as EventListener)
  window.addEventListener('whiteboard:export-png', handleExportPNG)
  window.addEventListener('whiteboard:project-loaded', handleProjectLoaded)
  window.addEventListener('whiteboard:theme-changed', applyThemeColors)
  window.addEventListener('whiteboard:grid-changed', updateGridConfig)
})

onUnmounted(() => {
  saveCurrentPageShapes()
  window.removeEventListener('resize', updateSize)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('whiteboard:undo', handleUndoEvent)
  window.removeEventListener('whiteboard:redo', handleRedoEvent)
  window.removeEventListener('whiteboard:rotate-cw', handleRotateCWEvent)
  window.removeEventListener('whiteboard:rotate-ccw', handleRotateCCWEvent)
  window.removeEventListener('whiteboard:flip-h', handleFlipHEvent)
  window.removeEventListener('whiteboard:flip-v', handleFlipVEvent)
  window.removeEventListener('whiteboard:layer-up', handleLayerUpEvent)
  window.removeEventListener('whiteboard:layer-down', handleLayerDownEvent)
  window.removeEventListener('whiteboard:layer-top', handleLayerTopEvent)
  window.removeEventListener('whiteboard:layer-bottom', handleLayerBottomEvent)
  window.removeEventListener('whiteboard:align', handleAlignEvent as EventListener)
  window.removeEventListener('whiteboard:update-shape', handleUpdateShapeEvent as EventListener)
  window.removeEventListener('whiteboard:export-png', handleExportPNG)
  window.removeEventListener('whiteboard:project-loaded', handleProjectLoaded)
  window.removeEventListener('whiteboard:theme-changed', applyThemeColors)
  window.removeEventListener('whiteboard:grid-changed', updateGridConfig)
})

defineExpose({
  undo, redo, copySelectedShapes, cutSelectedShapes, pasteShapes, duplicateSelectedShapes,
  rotateClockwise, rotateCounterClockwise, flipHorizontal, flipVertical,
  moveLayerUp, moveLayerDown, moveToTop, moveToBottom, deleteSelectedShape,
  getSelectedShapes, shapes
})
</script>

<style scoped>
.canvas-area {
  flex: 1;
  background-color: #E0E0E0;
  padding: 16px;
  overflow: hidden;
  transition: background-color 0.3s ease;
}
.canvas-area :deep(.konvajs-content) {
  background-color: #FFFFFF;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}
</style>