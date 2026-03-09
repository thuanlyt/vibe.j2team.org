<template>
  <div class="flex flex-col items-center justify-center min-h-[80vh] bg-gray-900 text-white font-sans p-4 rounded-xl relative">
    
    <router-link to="/" class="absolute top-4 left-4 flex items-center gap-2 text-gray-400 hover:text-white transition-colors bg-gray-800 px-3 py-1.5 rounded-lg text-sm">
      <span>←</span> Quay lại
    </router-link>

    <h1 class="text-3xl font-bold mb-2 text-yellow-400 mt-8">💣 Mini Bomberman</h1>
    <p class="mb-4 text-gray-300 text-sm text-center">Di chuyển: W A S D (hoặc Phím mũi tên) | Đặt bom: SPACE hoặc ENTER</p>
    
    <div v-if="gameOver" class="mb-4 text-xl font-bold p-2 rounded" :class="won ? 'bg-green-500' : 'bg-red-500'">
      {{ won ? '🏆 BẠN ĐÃ THẮNG!' : '💀 GAME OVER!' }}
      <button @click="initGame" class="ml-4 bg-white text-black px-3 py-1 rounded text-sm hover:bg-gray-200">Chơi lại</button>
    </div>

    <div 
      class="bg-gray-800 p-2 rounded-lg shadow-2xl outline-none focus:ring-2 focus:ring-yellow-400" 
      tabindex="0" 
      @keydown.prevent="handleKeydown"
      ref="gameBoard"
    >
      <div v-for="(row, y) in board" :key="y" class="flex">
        <div 
          v-for="(cell, x) in row" 
          :key="x" 
          class="w-8 h-8 md:w-10 md:h-10 flex items-center justify-center text-xl md:text-2xl"
          :class="getCellBg(cell)"
        >
          {{ getEmoji(x, y, cell) }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

const size = 11;
const board = ref<number[][]>([]);
// 0: Rỗng, 1: Tường cứng, 2: Thùng mềm, 3: Bom, 4: Lửa

const player = ref({ x: 0, y: 0, alive: true });
const bot = ref({ x: size - 1, y: size - 1, alive: true });
const gameOver = ref(false);
const won = ref(false);
let botInterval: any = null;

const initGame = () => {
  let newBoard = Array(size).fill(null).map(() => Array(size).fill(0));
  for (let y = 0; y < size; y++) {
    for (let x = 0; x < size; x++) {
      if (x % 2 !== 0 && y % 2 !== 0) {
        newBoard[y][x] = 1;
      } else if (Math.random() < 0.3 && !(x < 2 && y < 2) && !(x > size - 3 && y > size - 3)) {
        newBoard[y][x] = 2;
      }
    }
  }
  board.value = newBoard;
  player.value = { x: 0, y: 0, alive: true };
  bot.value = { x: size - 1, y: size - 1, alive: true };
  gameOver.value = false;
  won.value = false;
  
  if (botInterval) clearInterval(botInterval);
  botInterval = setInterval(moveBot, 800);
};

const getEmoji = (x: number, y: number, cell: number) => {
  if (cell === 4) return '💥';
  if (player.value.alive && player.value.x === x && player.value.y === y) return '👦';
  if (bot.value.alive && bot.value.x === x && bot.value.y === y) return '👾';
  if (cell === 1) return '⬛';
  if (cell === 2) return '🟫';
  if (cell === 3) return '💣';
  return '';
};

const getCellBg = (cell: number) => {
  if (cell === 1) return 'bg-gray-600';
  if (cell === 2) return 'bg-amber-800';
  if (cell === 4) return 'bg-orange-500 bg-opacity-50';
  return 'bg-green-700 bg-opacity-30 border border-green-800 border-opacity-20';
};

const move = (dx: number, dy: number) => {
  if (gameOver.value) return;
  const nx = player.value.x + dx;
  const ny = player.value.y + dy;
  if (nx >= 0 && nx < size && ny >= 0 && ny < size && board.value[ny][nx] === 0) {
    player.value.x = nx;
    player.value.y = ny;
    checkCollision();
  }
};

const placeBomb = () => {
  if (gameOver.value) return;
  const bx = player.value.x;
  const by = player.value.y;
  if (board.value[by][bx] !== 0) return;
  
  board.value[by][bx] = 3;
  setTimeout(() => explode(bx, by), 2000);
};

const explode = (cx: number, cy: number) => {
  board.value[cy][cx] = 4;
  const dirs = [[0,1], [0,-1], [1,0], [-1,0]];
  
  dirs.forEach(([dx, dy]) => {
    const nx = cx + dx, ny = cy + dy;
    if (nx >= 0 && nx < size && ny >= 0 && ny < size && board.value[ny][nx] !== 1) {
      board.value[ny][nx] = 4;
    }
  });

  checkCasualties();

  setTimeout(() => {
    board.value[cy][cx] = 0;
    dirs.forEach(([dx, dy]) => {
      const nx = cx + dx, ny = cy + dy;
      if (nx >= 0 && nx < size && ny >= 0 && ny < size && board.value[ny][nx] === 4) {
        board.value[ny][nx] = 0;
      }
    });
  }, 500);
};

const moveBot = () => {
  if (gameOver.value || !bot.value.alive) return;
  const dirs = [[0,1], [0,-1], [1,0], [-1,0]];
  const validMoves = dirs.map(([dx, dy]) => ({ x: bot.value.x + dx, y: bot.value.y + dy }))
    .filter(m => m.x >= 0 && m.x < size && m.y >= 0 && m.y < size && board.value[m.y][m.x] === 0);
  
  if (validMoves.length > 0) {
    const move = validMoves[Math.floor(Math.random() * validMoves.length)];
    bot.value.x = move.x;
    bot.value.y = move.y;
    checkCollision();
  }
};

const checkCollision = () => {
  if (player.value.x === bot.value.x && player.value.y === bot.value.y) {
    endGame(false);
  }
};

const checkCasualties = () => {
  if (board.value[player.value.y][player.value.x] === 4) endGame(false);
  if (board.value[bot.value.y][bot.value.x] === 4) {
    bot.value.alive = false;
    endGame(true);
  }
};

const endGame = (isWin: boolean) => {
  gameOver.value = true;
  won.value = isWin;
  player.value.alive = isWin;
  if (botInterval) clearInterval(botInterval);
};

const handleKeydown = (e: KeyboardEvent) => {
  switch(e.key) {
    case 'ArrowUp': case 'w': case 'W': move(0, -1); break;
    case 'ArrowDown': case 's': case 'S': move(0, 1); break;
    case 'ArrowLeft': case 'a': case 'A': move(-1, 0); break;
    case 'ArrowRight': case 'd': case 'D': move(1, 0); break;
    case ' ': case 'Enter': placeBomb(); break;
  }
};

const gameBoard = ref<HTMLElement | null>(null);
onMounted(() => {
  initGame();
  gameBoard.value?.focus();
});
onUnmounted(() => {
  if (botInterval) clearInterval(botInterval);
});
</script>