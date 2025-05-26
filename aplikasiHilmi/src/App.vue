<template>
  <div class="app">
    <h1>Graph Builder</h1>

    <section class="controls">
      <input v-model="nodeA" placeholder="Location A" />
      <input v-model="nodeB" placeholder="Location B" />
      <input v-model.number="distance" placeholder="Distance" type="number" />
      <button @click="addEdge">Add Connection</button>
    </section>

    <section class="controls">
      <label>Select Start Node:</label>
      <select v-model="selectedStartNode">
        <option disabled value="">-- Select --</option>
        <option v-for="node in nodes" :key="node" :value="node">{{ node }}</option>
      </select>
      <button @click="runDijkstra">Run Dijkstra</button>
      <button @click="runFloydWarshall">Run Floyd-Warshall</button>
    </section>

    <div class="graph-container">
      <div ref="cyContainer" class="cytoscape-container"></div>
    </div>

    <section class="results">
      <h2>Shortest Paths</h2>
      <ul>
        <li v-for="(result, i) in shortestPaths" :key="i">{{ result }}</li>
      </ul>
    </section>
  </div>
</template>

<script>
import { ref, onMounted, nextTick } from 'vue';
import cytoscape from 'cytoscape';

export default {
  name: 'GraphApp',
  setup() {
    const nodeA = ref('');
    const nodeB = ref('');
    const distance = ref(0);
    const selectedStartNode = ref('');
    const shortestPaths = ref([]);
    const nodes = ref([]);
    const cyContainer = ref(null);
    let cy;

    // Force canvas position fix - hacky but forceful fix for offset issue
    const forceFixCanvasPosition = () => {
      if (!cyContainer.value) return;
      const canvases = cyContainer.value.querySelectorAll('canvas');
      canvases.forEach(canvas => {
        canvas.style.left = '0px' ;
        canvas.style.top = '0px' ;
        canvas.style.position = 'absolute';
      });
    };

    const initCytoscape = () => {
      cy = cytoscape({
        container: cyContainer.value,
        elements: [],
        style: [
          { selector: 'node', style: { 'label': 'data(id)', 'background-color': '#0074D9', 'color': 'black', 'text-valign': 'center', 'text-halign': 'center', 'font-size': '10px' } },
          { selector: 'edge', style: { 'label': 'data(weight)', 'line-color': '#aaa', 'curve-style': 'bezier', 'text-rotation': 'autorotate', 'font-size': 12, 'color': '#555', 'font-size': '10px' } },
          { selector: '.highlighted', style: { 'background-color': '#FF4136', 'line-color': '#FF4136', 'transition-property': 'background-color, line-color', 'transition-duration': '0.5s' } }
        ],
        layout: { name: 'cose', animate: true }
      });
    };

    const resetViewport = () => {
      cy.fit();
      cy.zoom(1);
      cy.pan({ x: 0, y: 0 });
    };

    const adjustAndFix = () => {
      resetViewport();
      forceFixCanvasPosition();
    };

    const addEdge = () => {
      if (!nodeA.value || !nodeB.value || distance.value <= 0) return;

      if (!cy.getElementById(nodeA.value).length) {
        cy.add({ data: { id: nodeA.value } });
        if (!nodes.value.includes(nodeA.value)) nodes.value.push(nodeA.value);
      }
      if (!cy.getElementById(nodeB.value).length) {
        cy.add({ data: { id: nodeB.value } });
        if (!nodes.value.includes(nodeB.value)) nodes.value.push(nodeB.value);
      }

      if (!cy.getElementById(`${nodeA.value}-${nodeB.value}`).length) {
        cy.add({
          data: {
            id: `${nodeA.value}-${nodeB.value}`,
            source: nodeA.value,
            target: nodeB.value,
            weight: distance.value
          }
        });
      }

      const layout = cy.layout({ name: 'cose', animate: true });
      layout.run();

      layout.on('layoutstop', () => {
        adjustAndFix();
      });

      nodeA.value = nodeB.value = '';
      distance.value = 0;
    };

    const animatePath = path => {
      path.forEach(ele => ele.addClass('highlighted'));
      setTimeout(() => path.forEach(ele => ele.removeClass('highlighted')), 1500);
    };

    const runDijkstra = () => {
      if (!selectedStartNode.value) return;
      const dijkstra = cy.elements().dijkstra(`#${selectedStartNode.value}`, e => e.data('weight'));
      shortestPaths.value = [];

      cy.nodes().forEach(target => {
        const dist = dijkstra.distanceTo(target);
        shortestPaths.value.push(`From ${selectedStartNode.value} to ${target.id()}: ${dist === Infinity ? 'No path' : dist}`);
        animatePath(dijkstra.pathTo(target));
      });
    };

    const runFloydWarshall = () => {
      const fw = cy.elements().floydWarshall(e => e.data('weight'));
      shortestPaths.value = [];
      cy.nodes().forEach(src => {
        cy.nodes().forEach(dst => {
          if (src.id() !== dst.id()) {
            const dist = fw.distance(src, dst);
            shortestPaths.value.push(`From ${src.id()} to ${dst.id()}: ${dist === Infinity ? 'No path' : dist}`);
            animatePath(fw.path(src, dst));
          }
        });
      });
    };

    onMounted(() => {
      initCytoscape();
      window.addEventListener('resize', () => {
        cy.resize();
        adjustAndFix();
      });

      // Delay applying fix on mount because elements etc may load async
      nextTick(() => {
        cy.resize();
        adjustAndFix();
      });
    });

    return {
      nodeA,
      nodeB,
      distance,
      selectedStartNode,
      shortestPaths,
      nodes,
      cyContainer,
      addEdge,
      runDijkstra,
      runFloydWarshall
    };
  }
};
</script>

<style scoped>
.app {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  padding: 1rem;
  max-width: 900px;
  margin: auto;
  color: #111;
  background: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 0 12px rgba(0,0,0,0.1);
}
.controls {
  margin: 1rem 0;
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  align-items: center;
}
.controls input,
.controls select {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
  min-width: 120px;
}
.controls button {
  background-color: #0074D9;
  color: white;
  border: none;
  padding: 0.55rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s ease;
  font-weight: 600;
}
.controls button:hover {
  background-color: #005fa3;
}
.graph-container {
  border: 2px solid #0074D9;
  height: 600px;
  width: 100%;
  margin-bottom: 1rem;
  display: flex;
  justify-content: center;
  align-items: center;
  background: white;
  border-radius: 8px;
  box-shadow: inset 0 0 10px #0074D9;
  position: relative;
  overflow: hidden;
}
.cytoscape-container {
  height: 100%;
  width: 100%;
  position: absolute;
  top: 0;
  left: 0;
}
.results {
  background: #e8f0fe;
  padding: 1rem;
  border-radius: 8px;
  font-size: 1rem;
  line-height: 1.5;
  color: #003366;
  max-height: 240px;
  overflow-y: auto;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
}
</style>
