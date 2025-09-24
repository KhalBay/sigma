<script setup lang="ts">
import {onMounted, onUnmounted} from 'vue'
import chroma from "chroma-js";
import Graph from "graphology";
import ForceSupervisor from "graphology-layout-force/worker";
import Sigma from "sigma";
import {v4 as uuid} from "uuid";
import closeIcon from '../assets/close.svg'

const initializeSigma = () => {
  const container = document.getElementById("sigma-container") as HTMLElement

  if (!container) {
    console.error("Sigma container not found")
    return
  }

  // Create a sample graph
  const graph = new Graph()
  graph.addNode("n1", {x: 0, y: 0, size: 10, color: chroma.random().hex(), status: 200})
  graph.addNode("n2", {x: -5, y: 5, size: 10, color: chroma.random().hex(), status: 200})
  graph.addNode("n3", {x: 5, y: 5, size: 10, color: chroma.random().hex(), status: 400})
  graph.addNode("n4", {x: 0, y: 10, size: 10, color: chroma.random().hex(), status: 200})
  graph.addEdge("n1", "n2")
  graph.addEdge("n2", "n4")
  graph.addEdge("n4", "n3")
  graph.addEdge("n3", "n1")

  const layout = new ForceSupervisor(graph, {
    isNodeFixed: (_, attr) => attr.highlighted
  })
  layout.start()

  const calculateEdgeStatus = (graph: Graph, source: string, target: string): number => {
    const sourceStatus = graph.getNodeAttribute(source, "status") || 200
    const targetStatus = graph.getNodeAttribute(target, "status") || 200
    return sourceStatus === 400 || targetStatus === 400 ? 400 : 200
  }

  const updateAllEdgeStatuses = (graph: Graph) => {
    graph.forEachEdge((edge, _attributes, source, target) => {
      const status = calculateEdgeStatus(graph, source, target)
      graph.setEdgeAttribute(edge, "status", status)
    })
  }

  updateAllEdgeStatuses(graph)

  // Create the sigma
  const renderer = new Sigma(graph, container, {
    minCameraRatio: 0.5,
    maxCameraRatio: 2,
    allowInvalidContainer: true,
    nodeReducer: (_node, data) => {
      const isWork = data.status === 200
      return {
        ...data,
        color: isWork ? 'green' : 'red', // Покраска нод для визуала
      }
    },
    edgeReducer: (_edge, data) => {
      const status = data.status || 200
      const isBroken = status === 400

      return {
        ...data,
        color: isBroken ? "red" : "green", // Тоже самое для ребер
        size: isBroken ? 3 : 1,
      }
    }
  })

  // Создаем отдельный canvas для иконок поверх основного
  const iconCanvas = document.createElement('canvas')
  const iconCtx = iconCanvas.getContext('2d')!
  iconCanvas.style.position = 'absolute'
  iconCanvas.style.top = '0'
  iconCanvas.style.left = '0'
  iconCanvas.style.pointerEvents = 'none' // Чтобы клики проходили сквозь
  container.style.position = 'relative'
  container.appendChild(iconCanvas)

  const syncCanvasSize = () => {
    const mainCanvas = container.querySelector('canvas') as HTMLCanvasElement
    if (mainCanvas) {
      iconCanvas.width = mainCanvas.width
      iconCanvas.height = mainCanvas.height
      iconCanvas.style.width = mainCanvas.style.width
      iconCanvas.style.height = mainCanvas.style.height
    }
  }

  const icon = new Image()
  icon.src = closeIcon

  const drawIconsOnBrokenEdges = () => {
    syncCanvasSize()

    iconCtx.clearRect(0, 0, iconCanvas.width, iconCanvas.height)

    graph.forEachEdge((_edge, attributes, source, target) => {
      if (attributes.status === 200) return
      const iconSize = 46

      const sourceNode = graph.getNodeAttributes(source)
      const targetNode = graph.getNodeAttributes(target)

      const sourceCoords = renderer.graphToViewport({x: sourceNode.x, y: sourceNode.y})
      const targetCoords = renderer.graphToViewport({x: targetNode.x, y: targetNode.y})

      const midX = (sourceCoords.x + targetCoords.x) / 2
      const midY = (sourceCoords.y + targetCoords.y) / 2

      const iconDx = midX - iconSize / 2
      const iconDy = midY - iconSize / 2

      if (icon.complete) {
        iconCtx.drawImage(icon, iconDx, iconDy, iconSize, iconSize)
      } else {
        iconCtx.fillText('X', midX, midY)
      }
    })
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
      status: Math.random() > 0.5 ? 200 : 400 // Рандомный статус для новых нод
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

    updateAllEdgeStatuses(graph)
  })

  renderer.on('afterRender', drawIconsOnBrokenEdges)

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
      Хелоу! Немного объясняю логику работы. Я беру за основу, что в ноде у нас будет лежать какой-то статус(200 или
      400). Исходя из него мы рассчитываем, что если нода сломана, то оба ребра из неё, тоже сломаны (никидываем статусы
      на ребра через updateAllEdgeStatuses) даже если ведут к рабочей ноде, т.е. одна из двух нод сломанно - ребро
      считается сломанным(без учета направлений ребер). И в итоге мы рисуем канвас поверх всего нашего графа. И внутри
      на каждое сломанное ребро весим картинку, в самую середину ребра.
      <br><br>
      Ещё есть в голове такой вариант без кастомного канваса. Можно взять сломанное ребро и на нем отрисовать обычную
      ноду. А уже внутрь её через "@sigma/node-image" засунуть картинку. И нода должна быть типа - position absolut.
      Чтобы она была не физическая и просто была привязана к середине ребра, без возможности взаимодействия с ней (
      создание \ драг). Но здесь надо уже очень хорошо знать тонкости библиотеки.
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