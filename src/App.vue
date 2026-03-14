<script setup lang="ts">
import { ref, onUnmounted, onMounted, watch } from 'vue';
import './assets/main.css';

// Increased scale for Canvas
const ROWS = 150;
const COLS = 200;
const CELL_SIZE = 5;
const MAX_STATES = 100;

const createEmptyGrid = () => new Int32Array(ROWS * COLS).fill(0);

const grid = ref(createEmptyGrid());
const isRunning = ref(false);
const speed = ref(50); // ms
const intervalId = ref<number | null>(null);

const isRandomActive = ref(true);
const mutationInterval = ref(10); // generations
const generationCount = ref(0);

const canvasRef = ref<HTMLCanvasElement | null>(null);

const getIndex = (r: number, c: number) => r * COLS + c;

const toggleCell = (e: MouseEvent) => {
  if (isRunning.value || !canvasRef.value) return;
  const rect = canvasRef.value.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  const c = Math.floor(x / CELL_SIZE);
  const r = Math.floor(y / CELL_SIZE);
  
  if (r >= 0 && r < ROWS && c >= 0 && c < COLS) {
    const idx = getIndex(r, c);
    grid.value[idx] = grid.value[idx] === 1 ? 0 : 1;
    draw();
  }
};

const countNeighbors = (row: number, col: number, currentGrid: Int32Array) => {
  let count = 0;
  for (let i = -1; i <= 1; i++) {
    for (let j = -1; j <= 1; j++) {
      if (i === 0 && j === 0) continue;
      const newRow = (row + i + ROWS) % ROWS;
      const newCol = (col + j + COLS) % COLS;
      if (currentGrid[getIndex(newRow, newCol)] === 1) count++;
    }
  }
  return count;
};

const runSimulation = () => {
  const currentGrid = grid.value;
  const newGrid = createEmptyGrid();
  
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const idx = getIndex(r, c);
      const neighbors = countNeighbors(r, c, currentGrid);
      const currentStatus = currentGrid[idx];
      
      if (neighbors === 3) {
        newGrid[idx] = 1;
      } else if (currentStatus === 1 && neighbors === 2) {
        newGrid[idx] = 1;
      } else if (currentStatus >= 1 && currentStatus < MAX_STATES) {
        newGrid[idx] = currentStatus + 1;
      } else {
        newGrid[idx] = 0;
      }
    }
  }

  generationCount.value++;
  if (isRandomActive.value && generationCount.value % mutationInterval.value === 0) {
    const r = Math.floor(Math.random() * ROWS);
    const c = Math.floor(Math.random() * COLS);
    newGrid[getIndex(r, c)] = 1;
  }

  grid.value = newGrid;
  draw();
};

const getCellColor = (status: number) => {
  if (status === 0) return '#0d0d12';
  if (status === 1) return '#ffff00';
  const t = (status - 2) / (MAX_STATES - 2);
  let h, s, l;
  if (t < 0.5) {
    const factor = t * 2;
    h = 0 + factor * 25;
    s = 100 - factor * 40;
    l = 27 + factor * 3;
  } else {
    const factor = (t - 0.5) * 2;
    h = 25 + factor * 95;
    s = 60 + factor * 40;
    l = 30 - factor * 5;
  }
  return `hsl(${h}, ${s}%, ${l}%)`;
};

const draw = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  ctx.fillStyle = '#0d0d12';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const status = grid.value[getIndex(r, c)];
      if (status !== 0) {
        ctx.fillStyle = getCellColor(status);
        ctx.fillRect(c * CELL_SIZE, r * CELL_SIZE, CELL_SIZE - 0.5, CELL_SIZE - 0.5);
      }
    }
  }
};

const startGame = () => {
  if (isRunning.value) return;
  isRunning.value = true;
  intervalId.value = window.setInterval(runSimulation, speed.value);
};

const stopGame = () => {
  isRunning.value = false;
  if (intervalId.value) {
    clearInterval(intervalId.value);
    intervalId.value = null;
  }
};

const randomizeGrid = () => {
  stopGame();
  const newGrid = createEmptyGrid();
  for (let i = 0; i < newGrid.length; i++) {
    if (Math.random() > 0.9) newGrid[i] = 1;
  }
  grid.value = newGrid;
  generationCount.value = 0;
  draw();
};

const restartGame = () => {
  randomizeGrid();
  startGame();
};

const updateSpeed = (newSpeed: number) => {
  speed.value = newSpeed;
  if (isRunning.value) {
    stopGame();
    startGame();
  }
};

onMounted(() => {
  restartGame();
});

onUnmounted(() => {
  stopGame();
});

watch(speed, () => {
  if (isRunning.value) {
    stopGame();
    startGame();
  }
});
</script>

<template>
  <div class="app-container">
    <h1>Conway's Game of Life</h1>
    
    <div class="controls">
      <button @click="restartGame" class="primary">Restart</button>
      <button 
        @click="isRandomActive = !isRandomActive" 
        :class="{ primary: isRandomActive }"
      >
        Random Mutation: {{ isRandomActive ? 'ON' : 'OFF' }}
      </button>
    </div>

    <div class="canvas-container">
      <canvas
        ref="canvasRef"
        :width="COLS * CELL_SIZE"
        :height="ROWS * CELL_SIZE"
        @mousedown="toggleCell"
      ></canvas>
    </div>

    <div class="slider-container">
      <div class="control-group">
        <label for="speed">Speed: {{ speed }}ms</label>
        <input
          type="range"
          id="speed"
          min="10"
          max="500"
          step="10"
          :value="speed"
          @input="(e) => updateSpeed(parseInt((e.target as HTMLInputElement).value))"
        />
      </div>
      
      <div class="control-group">
        <label for="mutation">Mutation Every: {{ mutationInterval }} gens</label>
        <input
          type="range"
          id="mutation"
          min="1"
          max="100"
          step="1"
          v-model.number="mutationInterval"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.canvas-container {
  display: flex;
  justify-content: center;
  margin: 0 auto;
  border: 2px solid #00ffcc;
  box-shadow: 0 0 20px rgba(0, 255, 204, 0.2);
  border-radius: 4px;
  background-color: #0d0d12;
  line-height: 0;
}

canvas {
  cursor: crosshair;
}

.control-group {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}
</style>
