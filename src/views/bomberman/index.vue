<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import { RouterLink } from 'vue-router'

const size = 11

const board = ref<number[][]>([])

/*
0 empty
1 wall
2 crate
3 bomb
4 fire
*/

const player = ref({ x: 0, y: 0, alive: true })
const bot = ref({ x: size - 1, y: size - 1, alive: true })

const gameOver = ref(false)
const won = ref(false)

// ĐỒNG HỒ TRỌNG TÀI - Cố định tốc độ cho cả Bot và Người (ms/bước)
const GAME_SPEED = 400 

let botInterval: ReturnType<typeof setInterval> | null = null
let botLastBombTime = 0

let playerMoveInterval: ReturnType<typeof setInterval> | null = null
const activeKeys = new Set<string>()
let lastPlayerMoveTime = 0 // Biến để khóa tốc độ người chơi

const dirs: [number, number][] = [
  [0, 1],
  [0, -1],
  [1, 0],
  [-1, 0],
]

function initGame(): void {
  const newBoard = Array(size).fill(null).map(() => Array(size).fill(0))

  for (let y = 0; y < size; y++) {
    for (let x = 0; x < size; x++) {
      if (x % 2 !== 0 && y % 2 !== 0) {
        newBoard[y][x] = 1
      } else if (Math.random() < 0.3) {
        newBoard[y][x] = 2
      }
    }
  }

  // Clear khu vực spawn để không bị kẹt
  newBoard[0][0] = 0; newBoard[0][1] = 0; newBoard[1][0] = 0;
  newBoard[size - 1][size - 1] = 0; newBoard[size - 1][size - 2] = 0; newBoard[size - 2][size - 1] = 0;

  board.value = newBoard
  player.value = { x: 0, y: 0, alive: true }
  bot.value = { x: size - 1, y: size - 1, alive: true }
  gameOver.value = false
  won.value = false
  botLastBombTime = 0
  activeKeys.clear()
  lastPlayerMoveTime = 0

  // 1. Bot di chuyển theo đúng GAME_SPEED
  if (botInterval) clearInterval(botInterval)
  botInterval = setInterval(botThink, GAME_SPEED)

  // 2. Vòng lặp nhận phím người chơi (chạy nhanh để mượt, nhưng tốc độ di chuyển bị giới hạn bởi GAME_SPEED)
  if (playerMoveInterval) clearInterval(playerMoveInterval)
  playerMoveInterval = setInterval(() => {
    if (activeKeys.size > 0) processPlayerMove()
  }, 50)

  // Đảm bảo gameBoard tự động nhận phím sau khi khởi tạo
  nextTick(() => {
    gameBoard.value?.focus()
  })
}

function getEmoji(x: number, y: number, cell: number) {
  if (cell === 4) return "🔥"
  if (player.value.alive && player.value.x === x && player.value.y === y) return "🧑‍🚀"
  if (bot.value.alive && bot.value.x === x && bot.value.y === y) return "🤖"
  if (cell === 1) return ""
  if (cell === 2) return "📦"
  if (cell === 3) return "💣"
  return ""
}

function getTileClass(cell: number) {
  if (cell === 1) return "bg-gray-700 border-gray-900 shadow-inner"
  if (cell === 2) return "bg-yellow-800 border-yellow-900"
  if (cell === 4) return "bg-orange-500 opacity-80"
  return "bg-gray-800 bg-opacity-40 border-gray-800 border-opacity-50"
}

function move(dx: number, dy: number) {
  if (gameOver.value) return
  
  // CHỐT CHẶN: Ép người chơi phải tuân thủ GAME_SPEED để công bằng với Bot
  const now = Date.now()
  if (now - lastPlayerMoveTime < GAME_SPEED) return 

  const nx = player.value.x + dx
  const ny = player.value.y + dy

  if (nx >= 0 && nx < size && ny >= 0 && ny < size && board.value[ny]![nx] === 0) {
    player.value.x = nx
    player.value.y = ny
    lastPlayerMoveTime = now // Lưu lại thời gian vừa bước đi
    checkCollision()
  }
}

function placeBomb(px: number, py: number) {
  if (board.value[py]![px] !== 0) return
  board.value[py]![px] = 3
  setTimeout(() => explode(px, py), 1800)
}

function explode(cx: number, cy: number) {
  board.value[cy]![cx] = 4

  dirs.forEach(([dx, dy]) => {
    for (let i = 1; i <= 2; i++) {
      const nx = cx + dx * i
      const ny = cy + dy * i

      if (nx < 0 || ny < 0 || nx >= size || ny >= size) break
      if (board.value[ny]![nx] === 1) break
      if (board.value[ny]![nx] === 2) {
        board.value[ny]![nx] = 4
        break
      }
      board.value[ny]![nx] = 4
    }
  })

  checkCasualties()
  board.value = [...board.value]

  setTimeout(() => {
    board.value[cy]![cx] = 0
    dirs.forEach(([dx, dy]) => {
      for (let i = 1; i <= 2; i++) {
        const nx = cx + dx * i
        const ny = cy + dy * i
        if (nx < 0 || ny < 0 || nx >= size || ny >= size) break
        if (board.value[ny]![nx] === 4) board.value[ny]![nx] = 0
      }
    })
    board.value = [...board.value]
  }, 600)
}

function checkCasualties() {
  if (board.value[player.value.y]![player.value.x] === 4) endGame(false)
  if (board.value[bot.value.y]![bot.value.x] === 4) {
    bot.value.alive = false
    endGame(true)
  }
}

function checkCollision() {
  if (player.value.x === bot.value.x && player.value.y === bot.value.y) {
    endGame(false)
  }
}

function endGame(win: boolean) {
  gameOver.value = true
  won.value = win
  if (botInterval) clearInterval(botInterval)
  if (playerMoveInterval) clearInterval(playerMoveInterval)
  activeKeys.clear()
}

/* ---------------- */
/* PLAYER MOVEMENT  */
/* ---------------- */

function processPlayerMove() {
  if (gameOver.value) return
  let dx = 0, dy = 0
  if (activeKeys.has('ArrowUp') || activeKeys.has('w') || activeKeys.has('W')) dy = -1
  else if (activeKeys.has('ArrowDown') || activeKeys.has('s') || activeKeys.has('S')) dy = 1
  else if (activeKeys.has('ArrowLeft') || activeKeys.has('a') || activeKeys.has('A')) dx = -1
  else if (activeKeys.has('ArrowRight') || activeKeys.has('d') || activeKeys.has('D')) dx = 1

  if (dx !== 0 || dy !== 0) {
    move(dx, dy) // Truyền vào hàm move đã được bảo vệ bởi chốt chặn GAME_SPEED
  }
}

function handleKeydown(e: KeyboardEvent) {
  if (['ArrowUp', 'w', 'W', 'ArrowDown', 's', 'S', 'ArrowLeft', 'a', 'A', 'ArrowRight', 'd', 'D'].includes(e.key)) {
    if (!activeKeys.has(e.key)) {
      activeKeys.add(e.key)
      processPlayerMove()
    }
  } else if (e.key === ' ' || e.key === 'Enter') {
    placeBomb(player.value.x, player.value.y)
  }
}

function handleKeyup(e: KeyboardEvent) {
  activeKeys.delete(e.key)
}

/* ---------------- */
/* BOT AI           */
/* ---------------- */

function botThink() {
  if (gameOver.value || !bot.value.alive) return

  const path = findPath(bot.value.x, bot.value.y, player.value.x, player.value.y)

  if (path.length > 1) {
    const next = path[1]
    const distanceToPlayer = Math.abs(player.value.x - bot.value.x) + Math.abs(player.value.y - bot.value.y)

    // Đặt bom nếu gần người chơi và đã hết cooldown (3 giây)
    if (distanceToPlayer <= 2 && Date.now() - botLastBombTime > 3000) {
      placeBomb(bot.value.x, bot.value.y)
      botLastBombTime = Date.now()
    }

    // Di chuyển
    if (next && board.value[next.y]![next.x] === 0) {
      bot.value.x = next.x
      bot.value.y = next.y
      checkCollision()
    }
  } else {
    // Kịch bản né bom cơ bản
    const safeMoves = dirs
      .map(([dx, dy]) => ({ x: bot.value.x + dx, y: bot.value.y + dy }))
      .filter(m => m.x >= 0 && m.x < size && m.y >= 0 && m.y < size && board.value[m.y]![m.x] === 0)
    
    if (safeMoves.length > 0) {
      const move = safeMoves[Math.floor(Math.random() * safeMoves.length)]!
      bot.value.x = move.x
      bot.value.y = move.y
      checkCollision()
    }
  }
}

function findPath(sx: number, sy: number, tx: number, ty: number) {
  const queue = [{ x: sx, y: sy, path: [] as {x: number, y: number}[] }]
  const visited = new Set()

  while (queue.length) {
    const node = queue.shift()!
    const key = node.x + "," + node.y

    if (visited.has(key)) continue
    visited.add(key)

    const newPath = [...node.path, { x: node.x, y: node.y }]

    if (node.x === tx && node.y === ty) return newPath

    for (const [dx, dy] of dirs) {
      const nx = node.x + dx
      const ny = node.y + dy

      if (nx < 0 || ny < 0 || nx >= size || ny >= size) continue
      if (board.value[ny]![nx] !== 0) continue

      queue.push({ x: nx, y: ny, path: newPath })
    }
  }
  return []
}

onMounted(() => {
  initGame()
})

onUnmounted(() => {
  if (botInterval) clearInterval(botInterval)
  if (playerMoveInterval) clearInterval(playerMoveInterval)
})
</script>

<template>
  <div class="min-h-[100dvh] flex flex-col items-center justify-center text-green-400 bg-gray-900 bg-[linear-gradient(#111827_1px,transparent_1px),linear-gradient(90deg,#111827_1px,transparent_1px)] bg-[size:40px_40px] p-2 sm:p-4">
    
    <div class="w-full max-w-xl">
      <div class="flex items-center justify-between mb-6">
        <RouterLink to="/" class="border border-green-500 px-3 py-1.5 hover:bg-green-500 hover:text-gray-900 transition-colors text-xs font-bold rounded flex items-center gap-2 bg-gray-900">
          <span>←</span> Trang chủ
        </RouterLink>
        <div class="text-xs sm:text-sm tracking-widest font-bold text-yellow-400 drop-shadow-md">
          BOMBERMAN
        </div>
      </div>

      <div v-if="gameOver" class="mb-4 border-2 p-3 text-center rounded-lg shadow-lg font-bold flex flex-col sm:flex-row items-center justify-center gap-4 z-10" :class="won ? 'border-green-500 text-green-400 bg-green-900/30' : 'border-red-500 text-red-400 bg-red-900/30'">
        <span class="text-lg">{{ won ? "🏆 MISSION SUCCESS" : "💀 MISSION FAILED" }}</span>
        <button class="border border-current px-4 py-1.5 rounded hover:bg-white/10 transition-colors" @click="initGame">
          RESTART
        </button>
      </div>

      <div class="flex justify-center mb-6">
        <div 
          ref="gameBoard" 
          tabindex="0" 
          @keydown.prevent="handleKeydown"
          @keyup.prevent="handleKeyup"
          @blur="activeKeys.clear()"
          class="bg-gray-800 p-2 sm:p-3 rounded-xl shadow-2xl border-2 border-gray-700 outline-none focus:border-green-500 transition-colors"
        >
          <div v-for="(row, y) in board" :key="y" class="flex">
            <div 
              v-for="(cell, x) in row" 
              :key="x" 
              class="flex items-center justify-center w-8 h-8 sm:w-10 sm:h-10 text-xl sm:text-2xl border border-black/20"
              :class="getTileClass(cell)"
            >
              {{ getEmoji(x, y, cell) }}
            </div>
          </div>
        </div>
      </div>

      <div class="md:hidden flex flex-col items-center gap-4 w-full max-w-[280px] mx-auto bg-gray-800/50 p-4 rounded-2xl border border-gray-700">
        <div class="grid grid-cols-3 gap-2 w-full">
          <div></div>
          <button class="bg-gray-700 h-12 rounded-lg active:bg-gray-600 active:scale-95 transition-all shadow-md flex items-center justify-center text-xl" @click="move(0,-1)">🔼</button>
          <div></div>
          
          <button class="bg-gray-700 h-12 rounded-lg active:bg-gray-600 active:scale-95 transition-all shadow-md flex items-center justify-center text-xl" @click="move(-1,0)">◀️</button>
          <button class="bg-red-900/80 border-2 border-red-500 h-12 rounded-lg active:bg-red-700 active:scale-95 transition-all shadow-lg flex items-center justify-center text-xl animate-pulse" @click="placeBomb(player.x, player.y)">💣</button>
          <button class="bg-gray-700 h-12 rounded-lg active:bg-gray-600 active:scale-95 transition-all shadow-md flex items-center justify-center text-xl" @click="move(1,0)">▶️</button>
          
          <div></div>
          <button class="bg-gray-700 h-12 rounded-lg active:bg-gray-600 active:scale-95 transition-all shadow-md flex items-center justify-center text-xl" @click="move(0,1)">🔽</button>
          <div></div>
        </div>
      </div>

      <p class="hidden md:block text-center text-gray-400 text-sm mt-4">
        Sử dụng <span class="text-white font-bold">W A S D</span> hoặc <span class="text-white font-bold">Phím mũi tên</span> để di chuyển<br>
        Nhấn <span class="text-white font-bold">SPACE</span> hoặc <span class="text-white font-bold">ENTER</span> để đặt bom
      </p>

    </div>
  </div>
</template>