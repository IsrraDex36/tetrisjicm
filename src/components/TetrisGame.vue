<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

 

// Configuración del juego
const ROWS = 20;
const COLS = 10;
const BLOCK_SIZE = 30;
const SHAPES = [
  [[-1, 1], [0, 1], [1, 1], [0, 0]], // T
  [[-1, 0], [0, 0], [1, 0], [2, 0]], // I
  [[-1, -1], [-1, 0], [0, 0], [1, 0]], // L
  [[1, -1], [-1, 0], [0, 0], [1, 0]], // J
  [[0, -1], [1, -1], [-1, 0], [0, 0]], // S
  [[-1, -1], [0, -1], [0, 0], [1, 0]], // Z
  [[0, -1], [1, -1], [0, 0], [1, 0]]  // O
];

// Audio
const audio = ref(null);
const isMuted = ref(false);
const volume = ref(0.5);
const showVolumeControl = ref(false);
const audioInitialized = ref(false);

// Variables score
const highScores = ref([]); 

// Cargar puntajes altos al iniciar
function loadHighScores() {
  const savedScores = localStorage.getItem('tetrisHighScores');
  if (savedScores) {
    highScores.value = JSON.parse(savedScores);
  } else {
    highScores.value = Array(5).fill({ name: '---', score: 0 });
  }
}

// Guardar nuevo puntaje
function saveHighScore(name) {
  const newScore = { name, score: score.value };
  
  // Añadir a la lista y ordenar
  const updatedScores = [...highScores.value, newScore]
    .sort((a, b) => b.score - a.score)
    .slice(0, 5);
  
  highScores.value = updatedScores;
  localStorage.setItem('tetrisHighScores', JSON.stringify(updatedScores));
}

// Verificar si el puntaje actual es un récord
function isNewHighScore() {
  return highScores.value.length < 5 || 
         score.value > highScores.value[highScores.value.length - 1].score;
}

// Estado del juego
const board = ref(Array(ROWS * COLS).fill(0));
const currentPiece = ref(null);
const nextPiece = ref(null);
const gameOver = ref(false);
const score = ref(0);
const level = ref(1);
const lines = ref(0);
const isPaused = ref(false);
const isStarted = ref(false);
const tempShapes = ref([]);
const speed = ref(700);

// Animaciones
const clearingLines = ref([]);
const particles = ref([]);
const isRotating = ref(false);

// Computed properties
const visibleCurrentBlocks = computed(() => {
  if (!currentPiece.value || !isStarted.value || gameOver.value) return [];
  return currentPiece.value.shape.map((block, i) => ({
    ...block,
    style: {
      left: `${(currentPiece.value.x + block[0]) * BLOCK_SIZE}px`,
      top: `${(currentPiece.value.y + block[1]) * BLOCK_SIZE}px`,
      backgroundColor: `var(--color-block-${currentPiece.value.index + 1})`,
      filter: isRotating.value ? 'brightness(1.2)' : 'brightness(1)'
    }
  }));
});

const visibleNextBlocks = computed(() => {
  if (!nextPiece.value) return [];
  return nextPiece.value.shape.map((block, i) => ({
    ...block,
    style: {
      left: `${(2 + block[0]) * BLOCK_SIZE}px`,
      top: `${(2 + block[1]) * BLOCK_SIZE}px`,
      backgroundColor: `var(--color-block-${nextPiece.value.index + 1})`
    }
  }));
});

// Inicializar el juego
function initGame() {
  board.value = Array(ROWS * COLS).fill(0);
  gameOver.value = false;
  score.value = 0;
  level.value = 1;
  lines.value = 0;
  speed.value = 700;
  isStarted.value = true;
  isPaused.value = false;
  particles.value = [];
  initTempShapes();
  spawnPiece();
  loadHighScores();
  startGameLoop();
  playAudio();
}

// Inicializar las piezas temporales
function initTempShapes() {
  tempShapes.value = Array.from({ length: SHAPES.length }, (_, i) => i);
  
  // Fisher-Yates shuffle
  for (let i = tempShapes.value.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [tempShapes.value[i], tempShapes.value[j]] = [tempShapes.value[j], tempShapes.value[i]];
  }
}

// Generar nueva pieza
function spawnPiece() {
  if (!tempShapes.value.length || tempShapes.value.length < 2) {
    initTempShapes();
  }
  
  const shapeIndex = tempShapes.value[0];
  currentPiece.value = {
    shape: SHAPES[shapeIndex],
    index: shapeIndex,
    x: Math.floor(COLS / 2) - 1,
    y: 0
  };
  
  // Configurar siguiente pieza
  const nextShapeIndex = tempShapes.value[1];
  nextPiece.value = {
    shape: SHAPES[nextShapeIndex],
    index: nextShapeIndex
  };
  
  tempShapes.value.shift();
  
  // Verificar colisión al spawnear (game over)
  if (checkCollision(currentPiece.value.x, currentPiece.value.y, currentPiece.value.shape)) {
    gameOver.value = true;
    clearInterval(gameInterval);
    createGameOverParticles();

    if (isNewHighScore()) {  
      saveHighScore('Player');
    }
  }
}

// Verificar colisiones
function checkCollision(x, y, shape) {
  return shape.some(([dx, dy]) => {
    const newX = x + dx;
    const newY = y + dy;
    return (
      newX < 0 || 
      newX >= COLS || 
      newY >= ROWS ||
      (newY >= 0 && board.value[newY * COLS + newX] !== 0)
    );
  });
}

// Mover pieza
function movePiece(direction) {
  if (gameOver.value || isPaused.value || !isStarted.value) return false;

  const piece = {...currentPiece.value};
  
  switch (direction) {
    case 'left':
      piece.x--;
      break;
    case 'right':
      piece.x++;
      break;
    case 'down':
      piece.y++;
      break;
    case 'rotate':
      return rotatePiece();
    default:
      return false;
  }
  
  if (!checkCollision(piece.x, piece.y, piece.shape)) {
    currentPiece.value = piece;
    return true;
  } else if (direction === 'down') {
    createImpactParticles();
    lockPiece();
    return false;
  }
  
  return false;
}

// Rotar pieza
function rotatePiece() {
  if (gameOver.value || isPaused.value || currentPiece.value.index === 6) return false;

  isRotating.value = true;
  const piece = {...currentPiece.value};
  const newShape = piece.shape.map(([x, y]) => [y * -1, x]);
  
  // Kick mechanism para evitar rotaciones en espacios estrechos
  const kicks = [
    [0, 0], [1, 0], [-1, 0], [0, 1], [0, -1]
  ];
  
  let rotated = false;
  
  for (const [dx, dy] of kicks) {
    if (!checkCollision(piece.x + dx, piece.y + dy, newShape)) {
      currentPiece.value = {
        ...piece,
        shape: newShape,
        x: piece.x + dx,
        y: piece.y + dy
      };
      rotated = true;
      break;
    }
  }
  
  setTimeout(() => {
    isRotating.value = false;
  }, 200);
  
  return rotated;
}

// Fijar pieza en el tablero
function lockPiece() {
  const { shape, x, y, index } = currentPiece.value;
  
  shape.forEach(([dx, dy]) => {
    const newX = x + dx;
    const newY = y + dy;
    if (newY >= 0) {
      board.value[newY * COLS + newX] = index + 1;
    }
  });
  
  checkLines();
  spawnPiece();
}

// Verificar líneas completas
function checkLines() {
  let linesCleared = 0;
  const linesToClear = [];
  
  for (let y = ROWS - 1; y >= 0; y--) {
    const rowStart = y * COLS;
    const row = board.value.slice(rowStart, rowStart + COLS);
    
    if (row.every(cell => cell !== 0)) {
      linesToClear.push(y);
      linesCleared++;
    }
  }
  
  if (linesToClear.length > 0) {
    // Activar animación de limpieza
    clearingLines.value = linesToClear;
    
    // Generar partículas
    generateParticles(linesToClear);
    
    // Retrasar la actualización del tablero para la animación
    setTimeout(() => {
      clearLines(linesToClear);
      clearingLines.value = [];
      
      // Actualizar puntuación después de la animación
      lines.value += linesCleared;
      const points = [0, 40, 100, 300, 1200][linesCleared] * level.value;
      score.value += points;
      
      // Aumentar nivel
      const newLevel = Math.floor(lines.value / 10) + 1;
      if (newLevel > level.value) {
        level.value = newLevel;
        speed.value = Math.max(100, 700 - (level.value - 1) * 50);
        startGameLoop();
      }
    }, 500); // Duración de la animación
  }
}

function clearLines(linesToClear) {
  const newBoard = [...board.value];
  
  linesToClear.forEach(y => {
    const rowStart = y * COLS;
    newBoard.splice(rowStart, COLS);
    newBoard.unshift(...Array(COLS).fill(0));
  });
  
  board.value = newBoard;
}

// Sistema de partículas
function generateParticles(linesToClear) {
  const newParticles = [];
  
  linesToClear.forEach(y => {
    for (let x = 0; x < COLS; x++) {
      const cellIndex = y * COLS + x;
      if (board.value[cellIndex] !== 0) {
        // Crear 3-5 partículas por bloque
        const count = 3 + Math.floor(Math.random() * 3);
        for (let i = 0; i < count; i++) {
          newParticles.push({
            x: x * BLOCK_SIZE + BLOCK_SIZE / 2,
            y: y * BLOCK_SIZE + BLOCK_SIZE / 2,
            color: `var(--color-block-${board.value[cellIndex]})`,
            size: Math.random() * 4 + 2,
            speedX: (Math.random() - 0.5) * 10,
            speedY: Math.random() * -15 - 5,
            life: 1,
            decay: Math.random() * 0.02 + 0.01
          });
        }
      }
    }
  });
  
  particles.value = [...particles.value, ...newParticles];
  updateParticles();
}

function createImpactParticles() {
  const { shape, x, y, index } = currentPiece.value;
  const newParticles = [];
  
  shape.forEach(([dx, dy]) => {
    const newX = x + dx;
    const newY = y + dy;
    if (newY >= 0) {
      // Crear 2-3 partículas por bloque
      const count = 2 + Math.floor(Math.random() * 2);
      for (let i = 0; i < count; i++) {
        newParticles.push({
          x: newX * BLOCK_SIZE + BLOCK_SIZE / 2,
          y: newY * BLOCK_SIZE + BLOCK_SIZE / 2,
          color: `var(--color-block-${index + 1})`,
          size: Math.random() * 3 + 1,
          speedX: (Math.random() - 0.5) * 5,
          speedY: Math.random() * -3 - 1,
          life: 1,
          decay: Math.random() * 0.02 + 0.01
        });
      }
    }
  });
  
  particles.value = [...particles.value, ...newParticles];
  updateParticles();
}

function createGameOverParticles() {
  const newParticles = [];
  
  for (let y = 0; y < ROWS; y++) {
    for (let x = 0; x < COLS; x++) {
      const cellIndex = y * COLS + x;
      if (board.value[cellIndex] !== 0) {
        // Crear partículas para todos los bloques
        const count = 2 + Math.floor(Math.random() * 2);
        for (let i = 0; i < count; i++) {
          newParticles.push({
            x: x * BLOCK_SIZE + BLOCK_SIZE / 2,
            y: y * BLOCK_SIZE + BLOCK_SIZE / 2,
            color: `var(--color-block-${board.value[cellIndex]})`,
            size: Math.random() * 4 + 2,
            speedX: (Math.random() - 0.5) * 8,
            speedY: Math.random() * -10 - 5,
            life: 1,
            decay: Math.random() * 0.01 + 0.005
          });
        }
      }
    }
  }
  
  particles.value = newParticles;
  updateParticles();
}

// Animación de partículas
function updateParticles() {
  particles.value = particles.value
    .map(p => ({
      ...p,
      x: p.x + p.speedX,
      y: p.y + p.speedY,
      speedY: p.speedY + 0.2, // Gravedad
      life: p.life - p.decay
    }))
    .filter(p => p.life > 0);
  
  if (particles.value.length > 0) {
    requestAnimationFrame(updateParticles);
  }
}

// Bucle del juego
let gameInterval;
function startGameLoop() {
  clearInterval(gameInterval);
  gameInterval = setInterval(() => {
    if (!isPaused.value && isStarted.value && !gameOver.value) {
      movePiece('down');
    }
  }, speed.value);
}

function togglePause() {
  isPaused.value = !isPaused.value;
  if (!isPaused.value && isStarted.value && !gameOver.value) {
    startGameLoop();
  }
}

// Control del teclado
function handleKeyDown(e) {
  const gameKeys = ['ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown', ' ', 'p', 'P', 'r', 'R', 'Enter'];
  
  if (gameKeys.includes(e.key)) {
    e.preventDefault();
  }

  if (!isStarted.value && e.key !== 'Enter') return;

  switch (e.key) {
    case 'ArrowLeft':
      movePiece('left');
      break;
    case 'ArrowRight':
      movePiece('right');
      break;
    case 'ArrowDown':
      movePiece('down');
      break;
    case 'ArrowUp':
      movePiece('rotate');
      break;
    case ' ':
      while (movePiece('down')) {}
      break;
    case 'p':
    case 'P':
      togglePause();
      break;
    case 'r':
    case 'R':
      initGame();
      break;
    case 'Enter':
      if (!isStarted.value || gameOver.value) initGame();
      break;
  }
}

// Audio functions
function initAudio() {
  try {
    audio.value = new Audio(new URL('../assets/audio/tetri.mp3', import.meta.url).href);
    audio.value.loop = true;
    audio.value.volume = volume.value;
    audioInitialized.value = true;
    
    audio.value.addEventListener('error', (e) => {
      console.error('Error en audio:', e);
    });
    
    // Intenta precargar el audio
    audio.value.load();
  } catch (error) {
    console.error('Error al inicializar audio:', error);
  }
}

function playAudio() {
  if (!audioInitialized.value) initAudio();
  
  if (audio.value && !isMuted.value) {
    audio.value.currentTime = 0;
    audio.value.play()
      .then(() => console.log('Audio iniciado'))
      .catch(e => console.log('Autoplay bloqueado:', e));
  }
}

function toggleMute() {
  if (!audioInitialized.value) initAudio();
  
  isMuted.value = !isMuted.value;
  if (audio.value) {
    if (isMuted.value) {
      audio.value.volume = 0;
      audio.value.pause();
    } else {
      audio.value.volume = volume.value;
      audio.value.play().catch(e => console.log('Error al reanudar audio:', e));
    }
  }
}

function setVolume(newVolume) {
  volume.value = newVolume;
  if (audio.value && !isMuted.value) {
    audio.value.volume = volume.value;
  }
}

// Ciclo de vida del componente
onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  initAudio();
});

onUnmounted(() => {
  clearInterval(gameInterval);
  window.removeEventListener('keydown', handleKeyDown);
  
  if (audio.value) {
    audio.value.pause();
    audio.value = null;
  }
});

onMounted(() => {
  if (window.lucide) {
    window.lucide.createIcons();
  }
});

</script>

<template>
  <div class="tetris-container"> 
    
    <!-- Controles de audio flotantes -->
    <div class="audio-controls">
      <button 
        @click="toggleMute" 
        class="audio-btn"
        :class="{ 'muted': isMuted }"
        aria-label="Silenciar/Activar sonido"
      >
        {{ isMuted ? '🔇' : '🔊' }}
      </button>
      
      <button 
        @click="showVolumeControl = !showVolumeControl" 
        class="audio-btn"
        aria-label="Ajustar volumen"
      >
        🎚️
      </button>
      
      <div 
        v-if="showVolumeControl" 
        class="volume-control"
        @mouseleave="showVolumeControl = false"
      >
        <input
          type="range"
          min="0"
          max="1"
          step="0.1"
          v-model.number="volume"
          @input="setVolume(volume)"
          class="volume-slider"
          aria-label="Control de volumen"
        >
        <span class="volume-value">{{ Math.round(volume * 100) }}%</span>
      </div>
    </div>
    
    <div class="game-wrapper">
      <!-- Columna izquierda - Puntajes altos --> 
      <div class="left-panel" v-if="isStarted">
        <div class="high-scores">
            <div class="high-scores-header">
            <i data-lucide="trophy" class="trophy-icon"></i>
            <h3>Mejores Puntajes</h3>
            </div>
            <div class="scores-list">
            <div 
                v-for="(score, index) in highScores" 
                :key="index" 
                class="score-item"
                :class="{
                'first-place': index === 0,
                'second-place': index === 1,
                'third-place': index === 2
                }"
            >
                <div class="score-rank">
                <i v-if="index === 0" data-lucide="medal" class="gold"></i>
                <i v-else-if="index === 1" data-lucide="medal" class="silver"></i>
                <i v-else-if="index === 2" data-lucide="medal" class="bronze"></i>
                <span v-else>{{ index + 1 }}</span>
                </div>
                <div class="score-info">
                <span class="score-name">
                    <i data-lucide="user" class="user-icon"></i>
                    {{ score.name }}
                </span>
                <span class="score-value">
                    <i data-lucide="star" class="star-icon"></i>
                    {{ score.score }}
                </span>
                </div>
            </div>
            </div>
        </div>
    </div>
      
      <!-- Centro - Tablero de juego -->
      <div class="game-board" :style="{
        width: `${COLS * BLOCK_SIZE}px`,
        height: `${ROWS * BLOCK_SIZE}px`,
        position: 'relative',
      }">
        <!-- Fondo del tablero -->
        <div class="board-background"></div>
        
        <!-- Bloques del tablero con animación -->
        <div 
          v-for="(cell, index) in board" 
          :key="`cell-${index}`"
          class="board-cell"
          :class="{
            'clearing': clearingLines.includes(Math.floor(index / COLS)),
            'filled': cell !== 0
          }"
          :style="{
            position: 'absolute',
            left: `${(index % COLS) * BLOCK_SIZE}px`,
            top: `${Math.floor(index / COLS) * BLOCK_SIZE}px`,
            width: `${BLOCK_SIZE}px`,
            height: `${BLOCK_SIZE}px`,
            backgroundColor: cell ? `var(--color-block-${cell})` : 'transparent',
            '--delay': `${(index % COLS) * 0.03}s`,
            'z-index': cell !== 0 ? 2 : 1
          }"
        ></div>
        
        <!-- Pieza actual -->
        <div 
          v-for="(block, i) in visibleCurrentBlocks"
          :key="`current-${i}`"
          class="current-block"
          :style="{
            position: 'absolute',
            left: block.style.left,
            top: block.style.top,
            width: `${BLOCK_SIZE}px`,
            height: `${BLOCK_SIZE}px`,
            backgroundColor: block.style.backgroundColor,
            filter: block.style.filter,
            'z-index': 3
          }"
        ></div>
        
        <!-- Partículas -->
        <div
          v-for="(particle, i) in particles"
          :key="`particle-${i}`"
          class="particle"
          :style="{
            left: `${particle.x}px`,
            top: `${particle.y}px`,
            width: `${particle.size}px`,
            height: `${particle.size}px`,
            backgroundColor: particle.color,
            opacity: particle.life,
            'z-index': 10
          }"
        ></div>
        
        <!-- Efecto de game over -->
        <div v-if="gameOver" class="game-over-overlay">
          <div class="game-over-content">
            <h2>¡JUEGO TERMINADO!</h2>
            <div class="final-score">Puntuación: {{ score }}</div>
            <button @click="initGame" class="control-btn">Jugar de nuevo</button>
          </div>
        </div>
      </div>
      
      <!-- Columna derecha - Información del juego -->
      <div class="right-panel">
        <div v-if="!isStarted" class="start-screen">
           <div class="controls-info">
                <h3><i class="icon-gamepad"></i> Controles</h3>
                <div class="controls-grid">
                    <div class="control-item">
                    <div class="key-combo">
                        <span class="key"><i data-lucide="arrow-left"></i></span>
                        <span class="key"><i data-lucide="arrow-right"></i></span>
                    </div>
                    <span class="control-label">Mover pieza</span>
                    </div>
                    
                    <div class="control-item">
                    <div class="key"><i data-lucide="arrow-up"></i></div>
                    <span class="control-label">Rotar pieza</span>
                    </div>
                    
                    <div class="control-item">
                    <div class="key"><i data-lucide="arrow-down"></i></div>
                    <span class="control-label">Bajar rápido</span>
                    </div>
                    
                    <div class="control-item">
                    <div class="key space"><i data-lucide="space"></i></div>
                    <span class="control-label">Caída instantánea</span>
                    </div>
                    
                    <div class="control-item">
                    <div class="key">P</div>
                    <span class="control-label">Pausar juego</span>
                    </div>
                    
                    <div class="control-item">
                    <div class="key">R</div>
                    <span class="control-label">Reiniciar</span>
                    </div>
                </div>
            </div>
          <button @click="initGame" class="start-btn">Comenzar Juego</button>
        </div>
        
        <div v-else class="game-stats">
          <div class="info-item">
            <h3>Puntuación</h3>
            <div class="info-value">{{ score }}</div>
          </div>
          
          <div class="info-item">
            <h3>Líneas</h3>
            <div class="info-value">{{ lines }}</div>
          </div>
          
          <div class="info-item">
            <h3>Nivel</h3>
            <div class="info-value">{{ level }}</div>
          </div>
          
          <div class="next-piece">
            <h3>Siguiente</h3>
            <div class="next-piece-container" :style="{
              width: `${4 * BLOCK_SIZE}px`,
              height: `${4 * BLOCK_SIZE}px`
            }">
              <div 
                v-for="(block, i) in visibleNextBlocks"
                :key="`next-${i}`"
                class="next-block"
                :style="{
                  position: 'absolute',
                  left: block.style.left,
                  top: block.style.top,
                  width: `${BLOCK_SIZE}px`,
                  height: `${BLOCK_SIZE}px`,
                  backgroundColor: block.style.backgroundColor,
                  'z-index': 2
                }"
              ></div>
            </div>
          </div>
          
          <div class="controls">
            <button @click="togglePause" class="control-btn">
              {{ isPaused ? 'Continuar' : 'Pausa' }}
            </button>
            <button @click="initGame" class="control-btn">Reiniciar</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  --color-block-1: #FF0D72; /* T */
  --color-block-2: #0DC2FF; /* I */
  --color-block-3: #0DFF72; /* L */
  --color-block-4: #F538FF; /* J */
  --color-block-5: #FF8E0D; /* S */
  --color-block-6: #FFE138; /* Z */
  --color-block-7: #3877FF; /* O */
  
  --color-board: #111;
  --color-border: #444;
  --color-text: #eee;
  --color-accent: #4CAF50;
  --color-danger: #FF5722;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.tetris-container { 
  text-align: center;
  padding: 20px;
  color: var(--color-text);
  background-color: #ee5522;
background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='63' height='63' viewBox='0 0 200 200'%3E%3Cdefs%3E%3ClinearGradient id='a' gradientUnits='userSpaceOnUse' x1='100' y1='33' x2='100' y2='-3'%3E%3Cstop offset='0' stop-color='%23000' stop-opacity='0'/%3E%3Cstop offset='1' stop-color='%23000' stop-opacity='1'/%3E%3C/linearGradient%3E%3ClinearGradient id='b' gradientUnits='userSpaceOnUse' x1='100' y1='135' x2='100' y2='97'%3E%3Cstop offset='0' stop-color='%23000' stop-opacity='0'/%3E%3Cstop offset='1' stop-color='%23000' stop-opacity='1'/%3E%3C/linearGradient%3E%3C/defs%3E%3Cg fill='%23d23d09' fill-opacity='0.34'%3E%3Crect x='100' width='100' height='100'/%3E%3Crect y='100' width='100' height='100'/%3E%3C/g%3E%3Cg fill-opacity='0.34'%3E%3Cpolygon fill='url(%23a)' points='100 30 0 0 200 0'/%3E%3Cpolygon fill='url(%23b)' points='100 100 0 130 0 100 200 100 200 130'/%3E%3C/g%3E%3C/svg%3E");
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: relative;
}

.controls-info {
  background: rgba(0, 0, 0, 0.7);
  padding: 20px; 
  border-radius: 12px;
  border: 2px solid var(--color-accent);
  margin-bottom: 20px; 
}

.controls-info h3 {
  color: var(--color-accent);
  margin-bottom: 15px;
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 2.0em; 
}

.controls-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 15px;
}

.control-item {
  display: flex;
  align-items: center;
  gap: 15px;
}

.key-combo {
  display: flex;
  gap: 5px;
}

.key {
  width: 40px;
  height: 40px;
  background: rgba(255, 255, 255, 0.1);
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-family: monospace;
  font-size: 1.2em;
}

.key.space {
  width: 80px;
}

.key i {
  width: 20px;
  height: 20px;
  stroke: currentColor;
}

.control-label {
  font-size: 0.9em;
  opacity: 0.9;
}

/* Animación para los iconos */
.key:hover {
  transform: scale(1.1);
  background: rgba(255, 255, 255, 0.2);
  transition: all 0.2s ease;
}

.game-title {
  color: #fff;
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
  margin-bottom: 20px;
  font-size: 2.5rem;
  background: linear-gradient(90deg, #0DC2FF, #F538FF);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  animation: titleGlow 3s infinite alternate;
}

@keyframes titleGlow {
  0% { text-shadow: 0 0 10px rgba(13, 194, 255, 0.5); }
  50% { text-shadow: 0 0 15px rgba(245, 56, 255, 0.7); }
  100% { text-shadow: 0 0 10px rgba(255, 13, 114, 0.5); }
}

.game-wrapper {
  display: flex;
  justify-content: center;
  gap: 30px;
  margin-top: 20px;
  width: 100%;
  max-width: 900px;
}

.left-panel {
  width: 220px;
  background: rgba(0, 0, 0, 0.7);
  border-radius: 12px;
  padding: 15px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  border: 2px solid rgba(255, 215, 0, 0.3);
}

.high-scores-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.high-scores-header h3 {
  color: #FFD700;
  margin: 0;
  font-size: 1.2rem;
  text-shadow: 0 0 8px rgba(255, 215, 0, 0.3);
}

.trophy-icon {
  width: 24px;
  height: 24px;
  color: #FFD700;
}

.scores-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.score-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.05);
  transition: all 0.2s ease;
}

.score-item:hover {
  background: rgba(255, 255, 255, 0.1);
  transform: translateX(3px);
}

.score-rank {
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 0.9rem;
}

.score-info {
  flex: 1;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.score-name {
  display: flex;
  align-items: center;
  gap: 6px;
}

.score-value {
  display: flex;
  align-items: center;
  gap: 4px;
  font-family: 'Courier New', monospace;
  font-weight: bold;
}

.user-icon, .star-icon {
  width: 16px;
  height: 16px;
  opacity: 0.7;
}

/* Estilos para los primeros puestos */
.first-place {
  border-left: 3px solid #FFD700;
}

.second-place {
  border-left: 3px solid #C0C0C0;
}

.third-place {
  border-left: 3px solid #CD7F32;
}

.gold {
  color: #FFD700;
}

.silver {
  color: #C0C0C0;
}

.bronze {
  color: #CD7F32;
}


.game-board {
  position: relative;
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.7);
  border-radius: 8px;
  overflow: hidden;
}

.board-background {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: var(--color-board);
  opacity: 0.9;
  z-index: 0;
}

.board-cell {
  transition: 
    background-color 0.2s ease,
    transform 0.3s ease,
    opacity 0.3s ease;
  border-radius: 2px;
}

.board-cell.filled {
  box-shadow: 
    inset 0 0 10px rgba(0, 0, 0, 0.5),
    0 0 5px rgba(255, 255, 255, 0.2);
}

/* Animación de limpieza de líneas */
.board-cell.clearing {
  animation: clearAnimation 0.5s ease-in-out var(--delay) forwards;
  transform-origin: center;
}

@keyframes clearAnimation {
  0% {
    transform: scale(1);
    opacity: 1;
    filter: brightness(1);
  }
  50% {
    transform: scale(1.2);
    opacity: 0.8;
    filter: brightness(1.5);
  }
  100% {
    transform: scale(0);
    opacity: 0;
    filter: brightness(2);
  }
}

/* Estilos para partículas */
.particle {
  position: absolute;
  border-radius: 50%;
  pointer-events: none;
  transition: 
    transform 0.1s ease-out,
    opacity 0.1s linear;
  transform: translate(-50%, -50%);
}

/* Transiciones para piezas */
.current-block {
  transition: 
    left 0.1s ease-out,
    top 0.1s ease-out,
    transform 0.1s ease-out,
    filter 0.1s ease;
  border-radius: 3px;
  transform: translateZ(0);
  box-shadow: 
    0 0 10px rgba(255, 255, 255, 0.3),
    inset 0 0 5px rgba(255, 255, 255, 0.2);
}

.next-block {
  border-radius: 3px;
  box-shadow: 
    inset 0 0 5px rgba(0, 0, 0, 0.3),
    0 0 3px rgba(255, 255, 255, 0.2);
}

.high-scores {
  background-color: rgba(0, 0, 0, 0.7);
  padding: 15px;
  border-radius: 8px;
  border: 1px solid var(--color-accent);
  margin-bottom: 20px;
}

.high-scores h3 {
  color: var(--color-accent);
  margin-bottom: 15px;
  text-align: center;
}

.scores-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.score-item {
  display: flex;
  justify-content: space-between;
  padding: 5px 10px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.score-rank {
  font-weight: bold;
  color: var(--color-accent);
}

.score-name {
  flex-grow: 1;
  text-align: left;
  padding-left: 10px;
}

.score-value {
  font-family: monospace;
  font-weight: bold;
}

.start-screen {
  display: flex;
  flex-direction: column;
  gap: 30px;
  align-items: center;
  justify-content: center;
  height: 100%;
}

.controls-info {
  text-align: left;
  background-color: rgba(0, 0, 0, 0.3);
  padding: 15px;
  border-radius: 8px;
  font-size: 0.9em;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.controls-info h3 {
  color: var(--color-accent);
  margin-bottom: 10px;
  text-align: center;
}

.controls-info ul {
  padding-left: 20px;
  margin: 10px 0 0 0;
  list-style-type: none;
}

.controls-info li {
  margin-bottom: 8px;
  position: relative;
  padding-left: 15px;
}

.controls-info li::before {
  content: '•';
  color: var(--color-accent);
  position: absolute;
  left: 0;
}

.info-item {
  background-color: rgba(0, 0, 0, 0.3);
  padding: 12px;
  border-radius: 8px;
  border: 12px solid rgba(255, 255, 255, 0.1);
  width: 180px;
}

.info-item h3 {
  margin: 0 0 8px 0;
  font-size: 1em;
  color: #aaa;
}

.info-value {
  font-size: 1.8em;
  font-weight: bold;
  color: #fff;
}

.next-piece {
  background-color: rgba(0, 0, 0, 0.3);
  padding: 12px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.next-piece h3 {
  margin: 0 0 12px 0;
  font-size: 1em;
  color: #aaa;
}

.next-piece-container {
  position: relative;
  margin: 0 auto;
  background-color: rgba(0, 0, 0, 0.2);
  border-radius: 4px;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 10px;
}

.start-btn, .control-btn {
  padding: 12px 12px;
  background: linear-gradient(145deg, var(--color-accent), #45a049);
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 1em;
  font-weight: bold;
  transition: all 0.2s;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.start-btn:hover, .control-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.start-btn:active, .control-btn:active {
  transform: translateY(0);
}

.start-btn {
  font-size: 1.2em;
  padding: 15px 30px;
  background: linear-gradient(145deg, var(--color-danger), #E64A19);
}

.game-over-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 20;
}

.game-over-content {
  background-color: rgba(0, 0, 0, 0.8);
  padding: 30px;
  border-radius: 10px;
  border: 1px solid var(--color-danger);
  max-width: 80%;
  animation: fadeIn 0.5s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

.game-over-content h2 {
  color: var(--color-danger);
  margin-top: 0;
  text-shadow: 0 0 5px rgba(255, 0, 0, 0.5);
}

.final-score {
  font-size: 1.2em;
  margin: 15px 0;
  color: #fff;
}

/* Controles de audio */
.audio-controls {
  position: fixed;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 8px;
  z-index: 1000;
}

.audio-btn {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.3);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 1.2rem;
  transition: all 0.2s;
}

.audio-btn:hover {
  background: rgba(0, 0, 0, 0.9);
  transform: scale(1.1);
}

.audio-btn.muted {
  opacity: 0.6;
}

.volume-control {
  position: absolute;
  right: 50px;
  top: 0;
  background: rgba(0, 0, 0, 0.8);
  padding: 10px 15px;
  border-radius: 20px;
  display: flex;
  align-items: center;
  gap: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
}

.volume-slider {
  width: 80px;
  cursor: pointer;
}

.volume-value {
  font-size: 0.8rem;
  min-width: 30px;
  text-align: center;
}

/* Efecto de brillo al rotar */
@keyframes pulse {
  0% { filter: brightness(1); }
  50% { filter: brightness(1.3); }
  100% { filter: brightness(1); }
}

/* Responsive */
/* Estilos responsivos para tablets (768px o menos) */
@media (max-width: 768px) {
  .game-wrapper {
    flex-direction: column;
    align-items: center;
    gap: 20px;
  }

  .left-panel {
    width: 100%;
    max-width: 400px;
    order: -1;
    margin-bottom: 20px;
    padding: 12px;
  }

  .high-scores-header {
    flex-direction: column;
    align-items: center;
    text-align: center;
    gap: 8px;
  }

  .high-scores-header h3 {
    font-size: 1.4rem;
  }

  .trophy-icon {
    width: 32px;
    height: 32px;
  }

  .scores-list {
    gap: 8px;
  }

  .score-item {
    padding: 8px;
  }

  .score-rank {
    width: 20px;
    height: 20px;
    font-size: 0.8rem;
  }

  .user-icon, .star-icon {
    width: 14px;
    height: 14px;
  }
}

/* Estilos responsivos para teléfonos (480px o menos) */
@media (max-width: 480px) {
  .left-panel {
    max-width: 100%;
    padding: 10px;
  }

  .high-scores-header h3 {
    font-size: 1.2rem;
  }

  .trophy-icon {
    width: 28px;
    height: 28px;
  }

  .score-item {
    padding: 6px 8px;
  }

  .score-info {
    flex-direction: column;
    align-items: flex-start;
    gap: 2px;
  }

  .score-value {
    margin-left: 30px; /* Alineación con el nombre */
  }

  /* Ajustar tamaño de fuente para mejor legibilidad */
  .score-name, .score-value {
    font-size: 0.9rem;
  }

  /* Ocultar iconos en pantallas muy pequeñas */
  @media (max-width: 360px) {
    .user-icon, .star-icon {
      display: none;
    }
    
    .score-name, .score-value {
      gap: 0;
    }
  }
}

/* Ajustes específicos para orientación horizontal en móviles */
@media (max-width: 768px) and (orientation: landscape) {
  .left-panel {
    max-width: 250px;
    margin-right: 20px;
    order: 0; /* Mantener orden original en horizontal */
  }
  
  .game-wrapper {
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
  }
  
  .high-scores-header {
    flex-direction: row;
  }
}
</style>