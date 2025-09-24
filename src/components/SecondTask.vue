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
      // const reverseKey = `${target}|${source}`
      edgesMap.set(key, { edge, attributes, source, target })
    })

    // Второй проход: определяем взаимные пары
    graph.forEachEdge((_edge, _attributes, source, target) => {
      const key = `${source}|${target}`
      const reverseKey = `${target}|${source}`

      // Если уже обработали эту пару, пропускаем
      if (pairToAggregate.has(key) || pairToAggregate.has(reverseKey)) {
        return
      }

      // Проверяем, существует ли обратное ребро
      if (edgesMap.has(reverseKey)) {
        // Это взаимная пара - назначаем обоим одинаковый aggregate
        pairToAggregate.set(key, currentAggregateId)
        pairToAggregate.set(reverseKey, currentAggregateId)
        currentAggregateId++
      }
    })

    // Третий проход: устанавливаем атрибуты
    graph.forEachEdge((_edge, attributes, source, target) => {
      const key = `${source}|${target}`

      if (pairToAggregate.has(key)) {
        attributes.agrigate = pairToAggregate.get(key)
      }
    })

    // Вывод результатов для проверки
    console.log('Mutual edges with aggregate IDs:')
    graph.forEachEdge((_edge, attributes, source, target) => {
      if (attributes.agrigate) {
        console.log(`Edge ${source} → ${target}: aggregate = ${attributes.agrigate}`)
      }
    })
    return pairToAggregate;
  }

  findAllAggregate(graph)


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
</style>