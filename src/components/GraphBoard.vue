<template>
  <div class="grafo-container">
    <h2 class="titulo"> Pizarra de Grafos </h2>

    <!-- Barra de acciones -->
    <div class="barra-acciones">
      <button @click="mostrarFormNodo = true" class="btn azul">➕ Añadir nodo</button>
      <button @click="mostrarMatriz" class="btn violeta"> Mostrar matriz</button>
      <button @click="limpiarGrafo" class="btn rojo"> Eliminar todo</button>
    </div>
    
    <div class="creador" >
      <small>Hecho por Daniel Zamorano</small>
    </div>
<!-- eliminar -->
<div v-if="confirmarEliminar" class="mensaje-confirmacion">
  <p><strong>¿Seguro que quieres eliminar todo el grafo?</strong></p>
  <button class="btn rojo" @click="confirmarEliminarGrafo">Sí</button>
  <button class="btn ghost" @click="confirmarEliminar = false">No</button>
</div>
<transition name="fade-scale">
  <div v-if="ventanaGrupo" class="ventana-flotante">
    <h3> Análisis de Grupo</h3>
    <p v-html="resultadoGrupo"></p>
    <div class="acciones">
      <button class="btn rojo" @click="ventanaGrupo = false">❌ Cerrar</button>
    </div>
  </div>
</transition>

    <!-- Formulario inline -->
    <transition name="fade">
      <div v-if="mostrarFormNodo" class="panel-inline">
        <input v-model="nuevoNombreNodo" placeholder="Nombre del nodo" />
        <div class="acciones">
          <button class="btn verde" @click="agregarNodoConfirmar">Agregar</button>
          <button class="btn ghost" @click="cancelarAgregarNodo">Cancelar</button>
        </div>
      </div>
    </transition>

    <!-- Pizarra -->
    <div class="contenedor-grafo" ref="contenedorRed"></div>

    <!-- Menú contextual -->
    <transition name="fade-scale">
      <div
        v-if="menuVisible"
        :style="{ top: menuY + 'px', left: menuX + 'px' }"
        class="menu-contextual"
      >
        <button class="btn azul" @click="editarNodo"> Cambiar nombre</button>
        <button class="btn violeta" @click="iniciarArista"> Crear arista</button>
        <button class="btn rojo" @click="eliminarNodo"> Eliminar</button>
      </div>
    </transition>

    <!-- Menú contextual de arista -->
    <transition name="fade-scale">
      <div
        v-if="menuAristaVisible"
        :style="{ top: menuAristaY + 'px', left: menuAristaX + 'px' }"
        class="menu-contextual"
      >
        <button class="btn azul" @click="modificarPesoArista"> Cambiar Peso</button>
        <button class="btn rojo" @click="eliminarArista"> Eliminar</button>
      </div>
    </transition>

    <!-- Modal de nueva arista -->
    <transition name="fade-scale">
      <div v-if="ventanaArista" class="ventana-flotante">
        <h3>➕ Nueva arista</h3>
        <label>
          Peso:
          <input type="number" v-model.number="nuevoPeso" min="0" step="1" />
        </label>
        <label class="check">
          <input type="checkbox" v-model="esDirigida" />
          Dirigida
        </label>

        <div class="acciones">
          <button class="btn verde" @click="confirmarArista">✅ Confirmar</button>
          <button class="btn rojo" @click="cancelarArista">❌ Cancelar</button>
        </div>
      </div>
    </transition>

    <!-- Modal de matriz -->
    <transition name="fade-scale">
      <div v-if="ventanaMatriz" class="ventana-flotante matriz">
        <h3>Matriz de adyacencia</h3>
        <table>
          <thead>
            <tr>
              <th></th>
              <th v-for="n in nodosOrdenados" :key="'col-' + n">{{ labels[n] }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="i in nodosOrdenados" :key="'row-' + i">
              <th>{{ labels[i] }}</th>
              <td v-for="j in nodosOrdenados" :key="i + '-' + j">
                {{ matriz[iIndex[i]][jIndex[j]] }}
              </td>
            </tr>
          </tbody>
        </table>
        <div class="acciones">
            <button class="btn azul" @click="descargarMatrizJson">⬇️ Descargar JSON</button>
            <button class="btn rojo" @click="ventanaMatriz = false">❌ Cerrar</button>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import { Network, DataSet } from "vis-network/standalone";

// Descargar matriz como JSON
function descargarMatrizJson() {
  const data = {
    labels: labels.value,
    nodosOrdenados: nodosOrdenados.value,
    matriz: matriz.value
  };
  const json = JSON.stringify(data, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'matriz_adyacencia.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

const contenedorRed = ref(null);
let nodos = null;
let aristas = null;
let network = null;

/* ===== UI: Menú contextual ===== */
const menuVisible = ref(false);
const menuX = ref(0);
const menuY = ref(0);
let nodoSeleccionado = null;

/* ===== UI: Menú de arista ===== */
const menuAristaVisible = ref(false);
const aristaSeleccionada = ref(null);
const menuAristaX = ref(0);
const menuAristaY = ref(0);

/* ===== UI: Añadir nodo ===== */
const mostrarFormNodo = ref(false);
const nuevoNombreNodo = ref("");

/* ===== Crear arista ===== */
const ventanaArista = ref(false);
const nuevoPeso = ref(1);
const esDirigida = ref(false);
let edgeTemp = null;
let esperandoDestino = false;
let origenArista = null;
let tempEdgeId = null;

// handlers
let hoverHandler = null;
let blurHandler = null;
let clickOnceHandler = null;

/* ===== Matriz ===== */
const ventanaMatriz = ref(false);
const matriz = ref([]);
const nodosOrdenados = ref([]);
const labels = ref({});
const iIndex = ref({});
const jIndex = ref({});





onMounted(() => {
  nodos = new DataSet([
    { id: 1, label: "Nodo 1" },
    { id: 2, label: "Nodo 2" }
  ]);

  aristas = new DataSet([
    { id: "e-1-2", from: 1, to: 2, label: "1", arrows: "to" }
  ]);

  const datos = { nodes: nodos, edges: aristas };

  const opciones = {
    interaction: { hover: true },
    manipulation: { enabled: false },
    physics: { stabilization: true }
  };

  network = new Network(contenedorRed.value, datos, opciones);

  // Click: abrir menú contextual SOLO en nodos, no en aristas
  network.on("click", (params) => {
    if (params.nodes?.length > 0) {
      nodoSeleccionado = params.nodes[0];
      const { x, y } = getNodeScreenPosition(nodoSeleccionado);
      menuX.value = x;
      menuY.value = y;
      menuVisible.value = true;
      menuAristaVisible.value = false;
    } else if (params.edges?.length > 0) {
      mostrarMenuArista(params.edges[0]);
      menuVisible.value = false;
      nodoSeleccionado = null;
    } else {
      menuVisible.value = false;
      menuAristaVisible.value = false;
      nodoSeleccionado = null;
    }
  });
});

function handleGlobalClick(e) {
  // Si el menú está abierto y el click no fue dentro del menú, ciérralo
  const menus = document.querySelectorAll('.menu-contextual');
  let clickedInsideMenu = false;
  menus.forEach(menu => {
    if (menu.contains(e.target)) clickedInsideMenu = true;
  });
  if (!clickedInsideMenu) {
    menuVisible.value = false;
    menuAristaVisible.value = false;
  }
}

onMounted(() => {
  window.addEventListener('mousedown', handleGlobalClick);
});

onBeforeUnmount(() => {
  window.removeEventListener('mousedown', handleGlobalClick);
});

/* ===== Helpers ===== */
const nextId = () => {
  const ids = nodos.getIds();
  return ids.length ? Math.max(...ids) + 1 : 1;
};

function getNodeScreenPosition(nodeId) {
  if (!network) return { x: 0, y: 0 };
  const pos = network.getPositions([nodeId])[nodeId];
  const canvasPos = network.canvasToDOM(pos);
  // Ajuste para centrar el menú al lado derecho del nodo
  return { x: canvasPos.x + 30, y: canvasPos.y - 10 };
}

function getEdgeScreenPosition(edgeId) {
  if (!network) return { x: 0, y: 0 };
  const edge = aristas.get(edgeId);
  if (!edge) return { x: 0, y: 0 };
  const fromPos = network.getPositions([edge.from])[edge.from];
  const toPos = network.getPositions([edge.to])[edge.to];
  // Punto medio de la arista
  const x = (fromPos.x + toPos.x) / 2;
  const y = (fromPos.y + toPos.y) / 2;
  const canvasPos = network.canvasToDOM({ x, y });
  return { x: canvasPos.x + 20, y: canvasPos.y - 10 };
}

/* ===== Añadir nodo ===== */
const agregarNodoConfirmar = () => {
  const nombre = (nuevoNombreNodo.value || "").trim();
  const id = nextId();
  nodos.add({ id, label: nombre || `Nodo ${id}` });
  nuevoNombreNodo.value = "";
  mostrarFormNodo.value = false;
};

const cancelarAgregarNodo = () => {
  nuevoNombreNodo.value = "";
  mostrarFormNodo.value = false;
};

/* ===== Menú contextual ===== */
const editarNodo = () => {
  if (!nodoSeleccionado) return;
  const actual = nodos.get(nodoSeleccionado)?.label ?? "";
  const nuevo = window.prompt("Nuevo nombre:", actual);
  if (nuevo && nuevo.trim()) {
    nodos.update({ id: nodoSeleccionado, label: nuevo.trim() });
  }
  menuVisible.value = false;
};

const eliminarNodo = () => {
  if (!nodoSeleccionado) return;
  nodos.remove(nodoSeleccionado);
  aristas.remove(
    aristas.get().filter((a) => a.from === nodoSeleccionado || a.to === nodoSeleccionado).map(e=>e.id)
  );
  menuVisible.value = false;
};

/* ===== Crear arista ===== */
const iniciarArista = () => {
  if (!nodoSeleccionado) return;
  cancelarListenersArista();

  origenArista = nodoSeleccionado;
  esperandoDestino = true;
  menuVisible.value = false;

  network.selectNodes([origenArista], false);

  hoverHandler = (params) => {
    const hovered = params.node;
    if (!esperandoDestino) return;

    // permitir mismo nodo como destino (loop)
    if (tempEdgeId && aristas.get(tempEdgeId)) {
      aristas.remove(tempEdgeId);
      tempEdgeId = null;
    }

    tempEdgeId = `temp-${origenArista}-${hovered}-${Date.now()}`;
    aristas.add({
      id: tempEdgeId,
      from: origenArista,
      to: hovered,
      dashes: true,
      label: "",
      color: { color: "#9ca3af" },
      physics: false,
      length: 150
    });
  };

  blurHandler = (params) => {
    const hovered = params.node;
    const idGuessPrefix = `temp-${origenArista}-${hovered}`;
    const found = aristas.get().find(e => e.id && String(e.id).startsWith(idGuessPrefix));
    if (found) {
      try { aristas.remove(found.id); } catch(e){
        console.error("Error al eliminar arista temporal:", e);
      }
      if (tempEdgeId === found.id) tempEdgeId = null;
    }
  };

  clickOnceHandler = (params) => {
    if (!esperandoDestino) return;
    if (params.nodes?.length > 0) {
      const destino = params.nodes[0];
      edgeTemp = { from: origenArista, to: destino };

      if (tempEdgeId && aristas.get(tempEdgeId)) {
        try { aristas.remove(tempEdgeId); } catch(e){
          console.error("Error al eliminar arista temporal:", e);
        }
        tempEdgeId = null;
      }

      ventanaArista.value = true;
      esperandoDestino = false;
      network.off("hoverNode", hoverHandler);
      network.off("blurNode", blurHandler);
    } else {
      cancelarArista();
    }
  };

  network.on("hoverNode", hoverHandler);
  network.on("blurNode", blurHandler);
  network.on("click", clickOnceHandler);
};

const confirmarArista = () => {
  if (edgeTemp && edgeTemp.from != null && edgeTemp.to != null) {
    if (nuevoPeso.value < 0) {
      alert("El peso no puede ser negativo.");
      return;
    }
    if(nuevoPeso.value === "") {
      alert("El peso no puede estar vacío.");
      return;
    }
    if(nuevoPeso.value === 0) {
      alert("El peso no puede ser cero.");
      return;
    }
    if(typeof nuevoPeso.value !== "number") {
      alert("El peso debe ser un número válido.");
      return;
    }
    const id = `e-${edgeTemp.from}-${edgeTemp.to}-${Date.now()}`;
    aristas.add({
      id,
      from: edgeTemp.from,
      to: edgeTemp.to,
      label: String(nuevoPeso.value ?? 1),
      arrows: esDirigida.value ? "to" : "",
      dashes: false
    });
  }
  cancelarArista();
};

const cancelarArista = () => {
  ventanaArista.value = false;
  edgeTemp = null;
  nuevoPeso.value = 1;
  esDirigida.value = false;
  esperandoDestino = false;
  origenArista = null;

  if (tempEdgeId && aristas.get(tempEdgeId)) {
    try { aristas.remove(tempEdgeId); } catch (e){
      console.error("Error al eliminar arista temporal:", e);
    }
    tempEdgeId = null;
  }
  cancelarListenersArista();
  network.unselectAll();
};

function cancelarListenersArista() {
  try {
    if (hoverHandler) network.off("hoverNode", hoverHandler);
    if (blurHandler) network.off("blurNode", blurHandler);
    if (clickOnceHandler) network.off("click", clickOnceHandler);
  } catch (e) {
    console.error("Error al cancelar listeners de arista:", e);
  }
  hoverHandler = null;
  blurHandler = null;
  clickOnceHandler = null;
}

// ===== Eliminar todo el grafo =====
const confirmarEliminar = ref(false);

const limpiarGrafo = () => {
  confirmarEliminar.value = true;
};

const confirmarEliminarGrafo = () => {
  nodos.clear();
  aristas.clear();
  cancelarArista();
  confirmarEliminar.value = false;
};

/* ===== Matriz ===== */
const mostrarMatriz = () => {
  const lista = nodos.get().map((n) => n.id).sort((a, b) => a - b);
  const n = lista.length;
  const M = Array.from({ length: n }, () => Array(n).fill(0));
  const dicLabels = {};
  nodos.get().forEach((nd) => (dicLabels[nd.id] = nd.label));

  aristas.get().forEach((e) => {
    if (String(e.id).startsWith("temp-")) return;
    const i = lista.indexOf(e.from);
    const j = lista.indexOf(e.to);
    const w = Number(e.label) || 1;
    if (i >= 0 && j >= 0) {
      M[i][j] = w;
      if (!e.arrows) M[j][i] = w;
    }
  });

  labels.value = dicLabels;
  nodosOrdenados.value = lista;

  const idx = {};
  lista.forEach((id, k) => (idx[id] = k));
  iIndex.value = idx;
  jIndex.value = idx;

  matriz.value = M;
  ventanaMatriz.value = true;
};

const mostrarMenuArista = (edgeId) => {
  aristaSeleccionada.value = edgeId;
  const { x, y } = getEdgeScreenPosition(edgeId);
  menuAristaX.value = x;
  menuAristaY.value = y;
  menuAristaVisible.value = true;
};

const eliminarArista = () => {
  if (aristaSeleccionada.value) {
    aristas.remove(aristaSeleccionada.value);
    menuAristaVisible.value = false;
  }
};

const modificarPesoArista = () => {
  if (aristaSeleccionada.value) {
    const arista = aristas.get(aristaSeleccionada.value);
    const nuevoPeso = window.prompt("Nuevo peso:", arista.label);
    if (nuevoPeso !== null && nuevoPeso !== "") {
      aristas.update({ id: aristaSeleccionada.value, label: String(nuevoPeso) });
    }
    if(nuevoPeso < 0) {
      alert("El peso no puede ser negativo.");
      aristas.update({ id: aristaSeleccionada.value, label: String(arista.label) });
      return;
    }
    if(nuevoPeso === "") {
      alert("El peso no puede estar vacío.");
      aristas.update({ id: aristaSeleccionada.value, label: String(arista.label) });
      return;
    }
    if(nuevoPeso === 0) {
      alert("El peso no puede ser cero.");
      aristas.update({ id: aristaSeleccionada.value, label: String(arista.label) });
      return;
    }
    
    menuAristaVisible.value = false;
  }
};
</script>

<style scoped>
.grafo-container {
  max-width: 1100px;
  margin: auto;
  padding: 10px;
}
.titulo {
  margin-bottom: 15px;
  text-align: center;
  font-size: 26px;
  font-weight: 700;
  background: linear-gradient(90deg, #6366f1, #3b82f6);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

/* Botones */
.btn {
  padding: 8px 16px;
  border-radius: 12px;
  border: none;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.2s ease-in-out;
  box-shadow: 0 4px 12px rgba(0,0,0,.1);
}
.btn:hover { transform: scale(1.05); }
.btn:active { transform: scale(0.95); }

.azul { background: #3b82f6; color: #fff; }
.violeta { background: #8b5cf6; color: #fff; }
.rojo { background: #ef4444; color: #fff; }
.verde { background: #22c55e; color: #fff; }
.ghost {
  background: transparent;
  color: #374151;
  border: 1px solid #d1d5db;
}

/* Barra de acciones */
.barra-acciones {
  display: flex;
  gap: 10px;
  justify-content: center;
  margin-bottom: 10px;
}

/* Panel inline */
.panel-inline {
  display: flex; gap: 10px; align-items: center; justify-content: center;
  background: #f9fafb; padding: 12px; border: 1px solid #e5e7eb;
  border-radius: 14px; margin-bottom: 12px;
  box-shadow: 0 3px 10px rgba(0,0,0,.05);
}
.panel-inline input {
  min-width: 220px; padding: 8px 12px;
  border-radius: 10px; border: 1px solid #cbd5e1;
}

/* Lienzo */
.contenedor-grafo {
  width: 100%; height: 600px;
  border: 2px solid #6366f1;
  border-radius: 18px;
  backdrop-filter: blur(12px);
  background: rgba(255,255,255,0.75);
  box-shadow: 0 8px 24px rgba(99,102,241,0.15);
  margin-bottom: 15px;
}

/* Menú contextual */
.menu-contextual {
  position: absolute;
  background: #fff;
  border: 1px solid #e5e7eb;
  padding: 8px;
  border-radius: 12px;
  display: flex; flex-direction: column; gap: 6px;
  z-index: 1000;
  box-shadow: 0 12px 30px rgba(0,0,0,.15);
}

/* Ventanas flotantes */
.ventana-flotante {
  position: fixed;
  top: 25%; left: 50%; transform: translateX(-50%);
  background: #fff;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 20px;
  z-index: 2000;
  box-shadow: 0 15px 40px rgba(0,0,0,.2);
  animation: pop 0.25s ease;
}
.ventana-flotante h3 {
  margin-bottom: 10px;
  text-align: center;
}
.check { display: inline-flex; align-items: center; gap: 6px; margin-left: 10px; }
.ventana-flotante .acciones { margin-top: 15px; justify-content: center; }

/* Tabla de matriz */
.matriz table {
  margin: auto;
  border-collapse: collapse;
  overflow: hidden;
  border-radius: 12px;
}
.matriz th, .matriz td {
  border: 1px solid #e5e7eb;
  padding: 8px 12px;
  text-align: center;
}
.matriz th {
  background: #6366f1;
  color: #fff;
}
.matriz tr:nth-child(even) {
  background: #f9fafb;
}

/* eliminar */
.mensaje-confirmacion {
  padding: 15px;
  border: 1px solid #ccc;
  border-radius: 12px;
  background: #fef2f2;
  margin: 10px auto;
  max-width: 400px;
  text-align: center;
  box-shadow: 0 4px 12px rgba(0,0,0,.1);
}
.mensaje-confirmacion p {
  margin-bottom: 12px;
}
.mensaje-confirmacion .btn {
  margin: 0 5px;
}



/* Animaciones */
@keyframes pop {
  from { transform: translateX(-50%) scale(0.8); opacity: 0; }
  to { transform: translateX(-50%) scale(1); opacity: 1; }
}
.fade-enter-active, .fade-leave-active { transition: opacity .2s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.fade-scale-enter-active, .fade-scale-leave-active {
  transition: all .25s ease;
}
.fade-scale-enter-from, .fade-scale-leave-to {
  opacity: 0; transform: scale(0.9);
}
</style>

