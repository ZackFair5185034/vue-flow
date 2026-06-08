<script lang="ts" setup>
import { shallowRef, toRef, watch } from 'vue'
import UserSelection from '../../components/UserSelection/UserSelection.vue'
import NodesSelection from '../../components/NodesSelection/NodesSelection.vue'
import type { EdgeChange, NodeChange } from '../../types'
import { SelectionMode } from '../../types'
import { useKeyPress, useVueFlow } from '../../composables'
import { areSetsEqual, getEventPosition, getNodesInside, getSelectionChanges } from '../../utils'
import { getMousePosition } from './utils'

const { isSelecting, selectionKeyPressed } = defineProps<{ isSelecting: boolean; selectionKeyPressed: boolean }>()

const {
  vueFlowRef,
  nodes,
  viewport,
  emits,
  userSelectionActive,
  removeSelectedElements,
  userSelectionRect,
  elementsSelectable,
  nodesSelectionActive,
  getSelectedEdges,
  getSelectedNodes,
  removeNodes,
  removeEdges,
  selectionMode,
  deleteKeyCode,
  multiSelectionKeyCode,
  multiSelectionActive,
  edgeLookup,
  nodeLookup,
  connectionLookup,
  defaultEdgeOptions,
  connectionStartHandle,
  panOnDrag,
} = useVueFlow()

const container = shallowRef<HTMLDivElement | null>(null)

const selectedNodeIds = shallowRef<Set<string>>(new Set())

const selectedEdgeIds = shallowRef<Set<string>>(new Set())

const containerBounds = shallowRef<DOMRect | null>(null)

const hasActiveSelection = toRef(() => elementsSelectable.value && (isSelecting || userSelectionActive.value))

const connectionInProgress = toRef(() => connectionStartHandle.value !== null)

// Used to prevent click events when the user lets go of the selectionKey during a selection
let selectionInProgress = false
let selectionStarted = false

const deleteKeyPressed = useKeyPress(deleteKeyCode, { actInsideInputWithModifier: false })

const multiSelectKeyPressed = useKeyPress(multiSelectionKeyCode)

watch(deleteKeyPressed, (isKeyPressed) => {
  if (!isKeyPressed) {
    return
  }

  removeNodes(getSelectedNodes.value)

  removeEdges(getSelectedEdges.value)

  nodesSelectionActive.value = false
})

watch(multiSelectKeyPressed, (isKeyPressed) => {
  multiSelectionActive.value = isKeyPressed
})

function wrapHandler(handler: Function, containerRef: HTMLDivElement | null) {
  return (event: MouseEvent) => {
    if (event.target !== containerRef) {
      return
    }

    handler?.(event)
  }
}

function onClick(event: MouseEvent) {
  if (selectionInProgress || connectionInProgress.value) {
    selectionInProgress = false
    return
  }

  emits.paneClick(event)

  removeSelectedElements()

  nodesSelectionActive.value = false
}

function onContextMenu(event: MouseEvent) {
  if (Array.isArray(panOnDrag.value) && panOnDrag.value?.includes(2)) {
    event.preventDefault()
    return
  }

  emits.paneContextMenu(event)
}

function onWheel(event: WheelEvent) {
  emits.paneScroll(event)
}

function onPointerDown(event: PointerEvent) {
  containerBounds.value = vueFlowRef.value?.getBoundingClientRect() ?? null

  if (
    !elementsSelectable.value ||
    !isSelecting ||
    event.button !== 0 ||
    event.target !== container.value ||
    !containerBounds.value
  ) {
    return
  }

  ;(event.target as Element)?.setPointerCapture?.(event.pointerId)

  const { x, y } = getMousePosition(event, containerBounds.value)

  selectionStarted = true
  selectionInProgress = false

  removeSelectedElements()

  userSelectionRect.value = {
    width: 0,
    height: 0,
    startX: x,
    startY: y,
    x,
    y,
  }

  emits.selectionStart(event)
}

function onPointerMove(event: PointerEvent) {
  if (!containerBounds.value || !userSelectionRect.value) {
    return
  }

  selectionInProgress = true

  const { x: mouseX, y: mouseY } = getEventPosition(event, containerBounds.value)
  const { startX = 0, startY = 0 } = userSelectionRect.value

  const nextUserSelectRect = {
    startX,
    startY,
    x: mouseX < startX ? mouseX : startX,
    y: mouseY < startY ? mouseY : startY,
    width: Math.abs(mouseX - startX),
    height: Math.abs(mouseY - startY),
  }

  // === CogniFlow 性能优化 (Phase B-2): 框选拖拽过程跳过 O(N) 包围盒检测 ===
  // 原始实现：每帧 60Hz 调 getNodesInside() 跑全节点命中检测 + getSelectionChanges()
  //           + emit nodes-change，前端无论是否拦截都要承担 VueFlow 内部开销。
  // 新实现：pointermove 期间只更新选区矩形 (供 UserSelection 视觉反馈)，
  //         命中检测 / emit 全部推迟到 onPointerUp 一次性执行。
  // 收益：50 节点画布 60Hz × O(N) 包围盒检测 → 0 次（仅维护选区视觉矩形）。
  // 视觉代价：框选过程中节点高亮不跟随，松手瞬间全亮（符合"只最终计算"的产品诉求）。
  // 兼容性：emit.selectionEnd 时机不变，前端 handleSelectionEnd 逻辑无需调整。
  userSelectionRect.value = nextUserSelectRect
  userSelectionActive.value = true
  nodesSelectionActive.value = false
}

/**
 * computeSelectionFromRect - 一次性计算选区命中节点/边并 emit 变化
 * 从原 onPointerMove 提取出来，供 onPointerUp 复用。P0-B2 优化。
 */
function computeSelectionFromRect(rect: typeof userSelectionRect.value) {
  if (!rect) return

  const prevSelectedNodeIds = selectedNodeIds.value
  const prevSelectedEdgeIds = selectedEdgeIds.value

  selectedNodeIds.value = new Set(
    getNodesInside(nodes.value, rect, viewport.value, selectionMode.value === SelectionMode.Partial, true).map(
      (node) => node.id,
    ),
  )

  selectedEdgeIds.value = new Set()
  const edgesSelectable = defaultEdgeOptions.value?.selectable ?? true

  // We look for all edges connected to the selected nodes
  for (const nodeId of selectedNodeIds.value) {
    const connections = connectionLookup.value.get(nodeId)
    if (!connections) {
      continue
    }
    for (const { edgeId } of connections.values()) {
      const edge = edgeLookup.value.get(edgeId)
      if (edge && (edge.selectable ?? edgesSelectable)) {
        selectedEdgeIds.value.add(edgeId)
      }
    }
  }

  if (!areSetsEqual(prevSelectedNodeIds, selectedNodeIds.value)) {
    const changes = getSelectionChanges(nodeLookup.value, selectedNodeIds.value, true) as NodeChange[]
    emits.nodesChange(changes)
  }

  if (!areSetsEqual(prevSelectedEdgeIds, selectedEdgeIds.value)) {
    const changes = getSelectionChanges(edgeLookup.value, selectedEdgeIds.value) as EdgeChange[]
    emits.edgesChange(changes)
  }
}

function onPointerUp(event: PointerEvent) {
  if (event.button !== 0 || !selectionStarted) {
    return
  }

  ;(event.target as Element)?.releasePointerCapture(event.pointerId)

  // We only want to trigger click functions when in selection mode if
  // the user did not move the mouse.
  if (!userSelectionActive.value && userSelectionRect.value && event.target === container.value) {
    onClick(event)
  }

  // === CogniFlow 性能优化 (Phase B-2): 框选结束一次性算命中 + emit ===
  // pointermove 期间只维护视觉矩形，命中检测全部推迟到 pointerup 一次性执行。
  // 必须在清 userSelectionRect / emits.selectionEnd 之前完成计算。
  computeSelectionFromRect(userSelectionRect.value)

  userSelectionActive.value = false
  userSelectionRect.value = null
  nodesSelectionActive.value = selectedNodeIds.value.size > 0

  emits.selectionEnd(event)

  // If the user kept holding the selectionKey during the selection,
  // we need to reset the selectionInProgress, so the next click event is not prevented
  if (selectionKeyPressed) {
    selectionInProgress = false
  }

  selectionStarted = false
}
</script>

<script lang="ts">
export default {
  name: 'Pane',
  compatConfig: { MODE: 3 },
}
</script>

<template>
  <div
    ref="container"
    class="vue-flow__pane vue-flow__container"
    :class="{ selection: isSelecting }"
    @click="(event) => (hasActiveSelection ? undefined : wrapHandler(onClick, container)(event))"
    @contextmenu="wrapHandler(onContextMenu, container)($event)"
    @wheel.passive="wrapHandler(onWheel, container)($event)"
    @pointerenter="(event) => (hasActiveSelection ? undefined : emits.paneMouseEnter(event))"
    @pointerdown="(event) => (hasActiveSelection ? onPointerDown(event) : emits.paneMouseMove(event))"
    @pointermove="(event) => (hasActiveSelection ? onPointerMove(event) : emits.paneMouseMove(event))"
    @pointerup="(event) => (hasActiveSelection ? onPointerUp(event) : undefined)"
    @pointerleave="emits.paneMouseLeave($event)"
  >
    <slot />
    <UserSelection v-if="userSelectionActive && userSelectionRect" :user-selection-rect="userSelectionRect" />
    <NodesSelection v-if="nodesSelectionActive && getSelectedNodes.length" />
  </div>
</template>
