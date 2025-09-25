<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'
import chroma from "chroma-js";
import EdgeCurveProgram from "@sigma/edge-curve";
import { EdgeArrowProgram } from "sigma/rendering";
import { MultiGraph } from "graphology";
import ForceSupervisor from "graphology-layout-force/worker";
import Sigma from "sigma";
import {v4 as uuid} from "uuid";


const initializeSigma = () => {
  const container = document.getElementById("sigma-container") as HTMLElement

  if (!container) {
    console.error("Sigma container not found")
    return
  }

  // Create a sample graph
  const graph = new MultiGraph()
  graph.addNode("b", { x: 1, y: -1, size: 20, label: "Bastian" })
  graph.addNode("c", { x: 3, y: -2, size: 10, label: "Charles" })
  graph.addNode("d", { x: 1, y: -3, size: 10, label: "Dorothea" })
  graph.addNode("e", { x: 3, y: -4, size: 20, label: "Ernestine" })

  graph.addEdge("b", "c", { forceLabel: true, label: "works with", type: "curved", curvature: 0.5 })
  graph.addEdge("b", "d", { forceLabel: true, label: "works with" })
  graph.addEdge("c", "b", { forceLabel: true, size: 3, label: "works with", type: "curved" })
  graph.addEdge("c", "e", { forceLabel: true, size: 3, label: "works with" })
  graph.addEdge("d", "c", { forceLabel: true, label: "works with", type: "curved", curvature: 0.1 })
  graph.addEdge("d", "e", { forceLabel: true, label: "works with", type: "curved", curvature: 1 })
  graph.addEdge("e", "d", { forceLabel: true, size: 2, label: "works with", type: "curved" })

  const layout = new ForceSupervisor(graph, {
    isNodeFixed: (_, attr) => attr.highlighted
  })
  layout.start()

  const renderer = new Sigma(graph, container, {
    allowInvalidContainer: true,
    defaultEdgeType: "straight",
    renderEdgeLabels: true,
    edgeProgramClasses: {
      straight: EdgeArrowProgram,
      curved: EdgeCurveProgram,
    },
  })

  const findAllAggregate = (graph: MultiGraph) => {
    // Словарь для хранения взаимных пар и их aggregate ID
    const pairToAggregate = new Map()
    let currentAggregateId = 1

    // Создаем множество всех ребер для проверки взаимности
    const edgesMap = new Map()

    // Первый проход: собираем все ребра
    graph.forEachEdge((edge, attributes, source, target) => {
      const key = `${source}|${target}`
      edgesMap.set(key, { edge, attributes, source, target })
    })

    // Второй проход: определяем взаимные пары
    graph.forEachEdge((_edge, _attributes, source, target) => {
      const key = `${source}|${target}`
      const reverseKey = `${target}|${source}`

      if (pairToAggregate.has(key) || pairToAggregate.has(reverseKey)) {
        return
      }

      if (edgesMap.has(reverseKey)) {
        pairToAggregate.set(key, currentAggregateId)
        pairToAggregate.set(reverseKey, currentAggregateId)
        currentAggregateId++
      }
    })

    // Третий проход: устанавливаем атрибуты
    graph.forEachEdge((_edge, attributes, source, target) => {
      const key = `${source}|${target}`

      if (pairToAggregate.has(key)) attributes.agrigate = pairToAggregate.get(key)
    })
    return pairToAggregate
  }

  findAllAggregate(graph)

  const circleCanvas = document.createElement('canvas')
  const circleCtx = circleCanvas.getContext('2d')!
  circleCanvas.style.position = 'absolute'
  circleCanvas.style.top = '0'
  circleCanvas.style.left = '0'
  circleCanvas.style.pointerEvents = 'none'
  container.style.position = 'relative'
  container.appendChild(circleCanvas)

  const syncCanvasSize = () => {
    const mainCanvas = container.querySelector('canvas') as HTMLCanvasElement
    if (mainCanvas) {
      circleCanvas.width = mainCanvas.width
      circleCanvas.height = mainCanvas.height
      circleCanvas.style.width = mainCanvas.style.width
      circleCanvas.style.height = mainCanvas.style.height
    }
  }

  const drawAggregateCircles = () => {
    syncCanvasSize()
    circleCtx.clearRect(0, 0, circleCanvas.width, circleCanvas.height)

    const aggregateGroups = new Map()

    graph.forEachEdge((_edge, attributes, source, target) => {
      if (attributes.agrigate) {
        const aggId = attributes.agrigate
        if (!aggregateGroups.has(aggId)) aggregateGroups.set(aggId, [])
        aggregateGroups.get(aggId).push({source, target, attributes})
      }
    })

    aggregateGroups.forEach((edges, _aggId) => {
      if (edges.length === 2) {
        const edge1 = edges[0]
        const edge2 = edges[1]

        const edge1Points = getCurvedEdgePoints(edge1, graph, renderer)
        const edge2Points = getCurvedEdgePoints(edge2, graph, renderer)

        const mid1 = edge1Points[Math.floor(edge1Points.length / 2)]!
        const mid2 = edge2Points[Math.floor(edge2Points.length / 2)]!

        const t = .5
        const x = mid1.x + (mid2.x - mid1.x) * t
        const y = mid1.y + (mid2.y - mid1.y) * t

        const angle = Math.atan2(mid2.y - mid1.y, mid2.x - mid1.x)

        const distance = Math.sqrt(Math.pow(mid2.x - mid1.x, 2) + Math.pow(mid2.y - mid1.y, 2))
        const radiusX = Math.max(distance / 2, 10) + 10
        const radiusY = Math.max(distance / 4, 15)

        circleCtx.save()
        circleCtx.translate(x, y)
        circleCtx.rotate(angle)

        circleCtx.beginPath()
        circleCtx.ellipse(0, 0, radiusX, radiusY, 0, 0, 2 * Math.PI)
        circleCtx.strokeStyle = 'green'
        circleCtx.lineWidth = 1
        circleCtx.stroke()

        circleCtx.restore()
      }
    })
  }

  const getCurvedEdgePoints = (edge: any, graph: MultiGraph, renderer: Sigma) => {
    const sourceNode = graph.getNodeAttributes(edge.source)
    const targetNode = graph.getNodeAttributes(edge.target)

    const sourceCoords = renderer.graphToViewport({x: sourceNode.x, y: sourceNode.y})
    const targetCoords = renderer.graphToViewport({x: targetNode.x, y: targetNode.y})

    const points = []
    const steps = 10 // Количество точек для аппроксимации кривой

    const curvature = edge.attributes?.curvature || 0.25

    for (let i = 0; i <= steps; i++) {
      const t = i / steps

      // Квадратичная кривая Безье с контрольной точкой
      const controlX = (sourceCoords.x + targetCoords.x) / 2 +
          (targetCoords.y - sourceCoords.y) * curvature
      const controlY = (sourceCoords.y + targetCoords.y) / 2 -
          (targetCoords.x - sourceCoords.x) * curvature

      // Вычисляем точку на кривой Безье
      const x = Math.pow(1 - t, 2) * sourceCoords.x +
          2 * (1 - t) * t * controlX +
          Math.pow(t, 2) * targetCoords.x

      const y = Math.pow(1 - t, 2) * sourceCoords.y +
          2 * (1 - t) * t * controlY +
          Math.pow(t, 2) * targetCoords.y

      points.push({ x, y })
    }

    return points
  }


  //
  // Drag'n'drop feature
  // ~~~~~~~~~~~~~~~~~~~
  //

  // State for drag'n'drop
  let draggedNode: string | null = null
  let isDragging = false

  // On mouse down on a node
  renderer.on("downNode", (e) => {
    isDragging = true
    draggedNode = e.node
    graph.setNodeAttribute(draggedNode, "highlighted", true)
    if (!renderer.getCustomBBox()) renderer.setCustomBBox(renderer.getBBox())
  })

  // Обновляем рисование при изменениях
  renderer.on('afterRender', drawAggregateCircles)

  // On mouse move
  renderer.on("moveBody", ({event}) => {
    if (!isDragging || !draggedNode) return

    // Get new position of node
    const pos = renderer.viewportToGraph(event)

    graph.setNodeAttribute(draggedNode, "x", pos.x)
    graph.setNodeAttribute(draggedNode, "y", pos.y)

    // Prevent sigma to move camera:
    event.preventSigmaDefault()
    event.original.preventDefault()
    event.original.stopPropagation()
  })

  // On mouse up
  const handleUp = () => {
    if (draggedNode) {
      graph.removeNodeAttribute(draggedNode, "highlighted")
    }
    isDragging = false
    draggedNode = null
  }
  renderer.on("upNode", handleUp)
  renderer.on("upStage", handleUp)

  //
  // Create node (and edge) by click
  // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  //

  renderer.on("clickStage", ({event}: { event: { x: number; y: number } }) => {
    const coordForGraph = renderer.viewportToGraph({x: event.x, y: event.y})

    const node = {
      ...coordForGraph,
      size: 8,
      color: chroma.random().hex(),
    }

    const closestNodes = graph
        .nodes()
        .map((nodeId) => {
          const attrs = graph.getNodeAttributes(nodeId)
          const distance = Math.pow(node.x - attrs.x, 2) + Math.pow(node.y - attrs.y, 2)
          return {nodeId, distance}
        })
        .sort((a, b) => a.distance - b.distance)
        .slice(0, 2)

    const id = uuid()
    graph.addNode(id, node)

    closestNodes.forEach((e) => graph.addEdge(id, e.nodeId))
  })

  return () => {
    layout.stop()
    renderer.kill()
  }
}

let cleanup: (() => void) | null = null

onMounted(() => {
  const newCleanup = initializeSigma()
  if (newCleanup) {
    cleanup = newCleanup
  }
})

onUnmounted(() => {
  if (cleanup) {
    cleanup()
  }
})
</script>

<template>
  <div class="flex">
    <div id="sigma-container"></div>
    <p>
      Тут конечно не обошлось без нейросети, так как рассчитывая кривые безье я бы сильно увеличил время работы и мог
      не успеть. Здесь кроме как нового слоя канвас у меня нет идей как это можно реализовать ещё.

      P.S. Ещё я не успел проверить на корректную работу при добавлении новых ребер в существующую агрегацию.
    </p>

  </div>
</template>

<style scoped>
#sigma-container {
  width: 95vw;
  display: flex;
  height: 90vh;
  position: relative;
  border: 2px dotted black;
  box-sizing: border-box;
  margin: 10px 0;
}

.flex {
  display: flex;
  flex-direction: column;
  align-items: center;
}

p {
  margin: 20px 0;
}
</style>