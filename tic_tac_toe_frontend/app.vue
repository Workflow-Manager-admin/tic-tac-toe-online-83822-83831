<template>
  <div class="ttt__wrapper">
    <header class="ttt__header">
      <h1>Tic Tac Toe</h1>
      <div class="ttt__mode-chooser">
        <button
          class="ttt__mode-btn"
          :class="{ active: mode === 'pvp' }"
          @click="setMode('pvp')"
          :aria-pressed="mode === 'pvp'"
        >
          Player vs Player
        </button>
        <button
          class="ttt__mode-btn"
          :class="{ active: mode === 'cpu' }"
          @click="setMode('cpu')"
          :aria-pressed="mode === 'cpu'"
        >
          Player vs Computer
        </button>
      </div>
    </header>
    <div class="ttt__scores">
      <div class="ttt__player-score" :class="{ 'ttt__player-active': currentPlayer === 'X' }">
        <span class="ttt__icon" :style="{ color: colors.primary }">X</span>
        Player 1 Score: <strong>{{ scores.X }}</strong>
      </div>
      <div class="ttt__player-score" :class="{ 'ttt__player-active': currentPlayer === 'O' }">
        <span class="ttt__icon" :style="{ color: colors.accent }">O</span>
        {{ mode === 'cpu' ? "CPU Score" : "Player 2 Score" }}: <strong>{{ scores.O }}</strong>
      </div>
    </div>
    <main>
      <div class="ttt__gameboard-container">
        <div class="ttt__gameboard">
          <div
            v-for="(cell, i) in board"
            :key="i"
            class="ttt__cell"
            :aria-label="'Cell ' + (i + 1)"
            :tabindex="isActive() && !cell ? 0 : -1"
            @click="makeMove(i)"
            @keyup.enter.space="makeMove(i)"
            :class="{
              played: cell,
              winner: winnerLine && winnerLine.includes(i),
              clickable: !cell && !gameOver
            }"
          >
            <span v-if="cell === 'X'" :style="{ color: colors.primary }">X</span>
            <span v-if="cell === 'O'" :style="{ color: colors.accent }">O</span>
          </div>
        </div>
        <div class="ttt__winner-message" v-if="gameOver">
          <span v-if="winner === 'draw'" class="ttt__draw-text">It's a draw!</span>
          <span v-else>
            <span class="ttt__icon" :style="{ color: winner === 'X' ? colors.primary : colors.accent }">{{ winner }}</span>
            <span class="ttt__win-text">
              {{ winner === 'X' ? "Player 1" : mode === 'cpu' ? "CPU" : "Player 2" }} wins!
            </span>
            <span class="ttt__confetti">🎉</span>
          </span>
        </div>
      </div>
      <div class="ttt__actions">
        <button class="ttt__restart-btn" @click="restartGame">
          Restart Game
        </button>
      </div>
    </main>
    <footer class="ttt__footer">
      <small>Minimalistic Tic Tac Toe &copy; 2024</small>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from "vue";

// Styling colors as per requirements
const colors = {
  primary: "#1E88E5",
  secondary: "#F5F5F5",
  accent: "#C62828"
};

type Player = "X" | "O";
type Cell = Player | "";

// State and Logic
const mode = ref<"pvp" | "cpu">("pvp");
const board = ref<Cell[]>(Array(9).fill(""));
const currentPlayer = ref<Player>("X");
const scores = ref<Record<Player, number>>({ X: 0, O: 0 });
const winner = ref<Player | "draw" | null>(null);
const gameOver = ref(false);
const winnerLine = ref<number[] | null>(null);

function setMode(newMode: "pvp" | "cpu") {
  if (mode.value !== newMode) {
    mode.value = newMode;
    restartGame(true);
  }
}

function isActive() {
  // Always active if PvP; if CPU, player acts when it's X's turn and not gameOver
  return mode.value === "pvp" || (mode.value === "cpu" && currentPlayer.value === "X");
}

// PUBLIC_INTERFACE
function makeMove(idx: number) {
  if (gameOver.value || board.value[idx]) return;
  if (mode.value === "cpu" && currentPlayer.value === "O") return; // Human acts as X

  board.value[idx] = currentPlayer.value;
  checkGameStatus();
  if (!gameOver.value) {
    currentPlayer.value = currentPlayer.value === "X" ? "O" : "X";
  }
}

// PUBLIC_INTERFACE
function restartGame(soft = false) {
  board.value = Array(9).fill("");
  currentPlayer.value = "X";
  winner.value = null;
  winnerLine.value = null;
  gameOver.value = false;
  if (!soft && mode.value === "cpu") {
    // Reset scores only if user not changing mode.
    scores.value = { X: 0, O: 0 };
  }
}

// PUBLIC_INTERFACE
function checkGameStatus() {
  const lines = [
    [0,1,2], [3,4,5], [6,7,8],
    [0,3,6], [1,4,7], [2,5,8],
    [0,4,8], [2,4,6]
  ];
  for (const line of lines) {
    const [a, b, c] = line;
    if (board.value[a] && board.value[a] === board.value[b] && board.value[a] === board.value[c]) {
      winner.value = board.value[a] as Player;
      winnerLine.value = line;
      scores.value[winner.value]++;
      gameOver.value = true;
      return;
    }
  }
  if (board.value.every(cell => cell)) {
    winner.value = "draw";
    gameOver.value = true;
  }
}

// CPU MOVE (Simple random, can be improved)
// PUBLIC_INTERFACE
function cpuMove() {
  // Only act if it's CPU's turn
  if (mode.value !== "cpu" || currentPlayer.value !== "O" || gameOver.value) return;
  // Simple: random open cell
  const emptyCells = board.value.map((cell, idx) => cell ? null : idx).filter(i => i !== null) as number[];
  if (emptyCells.length === 0) return;
  // Try to win/block else random
  let pick = cpuPickSmart(board.value, "O") ?? cpuPickSmart(board.value, "X");
  if (pick === null || pick === undefined) pick = emptyCells[Math.floor(Math.random() * emptyCells.length)];
  board.value[pick] = "O";
  checkGameStatus();
  if (!gameOver.value) {
    currentPlayer.value = "X";
  }
}

// Try to win/block
function cpuPickSmart(b: Cell[], checkFor: Player): number | null {
  const lines = [
    [0,1,2], [3,4,5], [6,7,8],
    [0,3,6], [1,4,7], [2,5,8],
    [0,4,8], [2,4,6]
  ];
  for (const line of lines) {
    const cells = line.map(i => b[i]);
    if (cells.filter(c => c === checkFor).length === 2 && cells.includes("")) {
      return line[cells.indexOf("")];
    }
  }
  return null;
}

// Watch for CPU turn
watch(
  () => [mode.value, currentPlayer.value, gameOver.value],
  ([m, p, over]) => {
    if (m === "cpu" && p === "O" && !over) {
      setTimeout(cpuMove, 400); // little delay for realism
    }
  }
);

// Reset board if mode changes (not scores)
watch(
  () => mode.value,
  () => { restartGame(true); }
);

</script>

<style scoped>
body {
  background: #F5F5F5;
}
.ttt__wrapper {
  min-height: 100vh;
  font-family: 'Inter', 'Roboto', Arial, sans-serif;
  background: #F5F5F5;
  color: #222;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
}
.ttt__header {
  text-align: center;
  margin-top: 32px;
  margin-bottom: 16px;
}
.ttt__header h1 {
  font-size: 2.2rem;
  margin-bottom: 12px;
  font-weight: 700;
  letter-spacing: -1px;
  color: #222;
}
.ttt__mode-chooser {
  display: flex;
  gap: 8px;
  justify-content: center;
  margin-top: 8px;
}
.ttt__mode-btn {
  background: #F5F5F5;
  color: #1E88E5;
  border: 2px solid #1E88E5;
  border-radius: 18px;
  padding: 8px 16px;
  font-size: 1rem;
  outline: none;
  font-weight: 500;
  transition: all 0.18s;
  cursor: pointer;
}
.ttt__mode-btn.active, .ttt__mode-btn:focus {
  background: #1E88E5;
  color: #fff;
}
.ttt__scores {
  display: flex;
  gap: 24px;
  justify-content: center;
  align-items: center;
  margin-bottom: 8px;
  margin-top: 15px;
}
.ttt__player-score {
  background: #fff;
  border: 1px solid #ededed;
  border-radius: 14px;
  padding: 8px 18px;
  display: flex;
  align-items: center;
  gap: 7px;
  font-size: 1.08rem;
  min-width: 170px;
  justify-content: center;
  transition: box-shadow .16s;
}
.ttt__player-active {
  box-shadow: 0 0 0 2px #1E88E5 inset;
  font-weight: 700;
  color: #1E88E5;
}
.ttt__icon {
  font-size: 1.19em;
  font-weight: 700;
}
main {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1 1 0;
}
.ttt__gameboard-container {
  margin: 26px 0 10px 0;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.ttt__gameboard {
  display: grid;
  grid-template-columns: repeat(3, 70px);
  grid-template-rows: repeat(3, 70px);
  gap: 7px;
  background: #eee;
  border-radius: 12px;
  box-shadow: 0 1px 8px #dbe6fd73;
  padding: 10px;
}
.ttt__cell {
  background: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2.15rem;
  font-weight: 700;
  cursor: pointer;
  border-radius: 10px;
  border: 2px solid #f0f6ff;
  outline: none;
  transition: background 0.16s, border 0.16s;
  min-width: 70px;
  min-height: 70px;
  user-select: none;
}
.ttt__cell.clickable:hover, .ttt__cell:focus {
  background: #e3f2fd;
  border-color: #90caf9;
}
.ttt__cell.played {
  cursor: default;
  color: #222;
}
.ttt__cell.winner {
  background: #C62828 !important;
  color: #fff !important;
  box-shadow: 0 0 5px #C62828;
  animation: pop 0.42s;
}
@keyframes pop {
  0% { transform: scale(1.0);}
  40% { transform: scale(1.12);}
  60% { transform: scale(0.98);}
  100% { transform: scale(1);}
}
.ttt__winner-message {
  margin: 20px 0 10px 0;
  text-align: center;
  font-size: 1.13rem;
  font-weight: 600;
  color: #333;
}
.ttt__draw-text {
  color: #1E88E5;
}
.ttt__win-text {
  color: #C62828;
  margin-left: 7px;
  font-weight: 700;
}
.ttt__confetti {
  margin-left: 8px;
  font-size: 1.2em;
}
.ttt__actions {
  display: flex;
  justify-content: center;
  margin: 18px 0 0 0;
}
.ttt__restart-btn {
  background: #C62828;
  color: #fff;
  border: none;
  border-radius: 29px;
  padding: 10px 28px;
  font-size: 1rem;
  font-weight: 600;
  letter-spacing: 0.5px;
  margin-bottom: 6px;
  cursor: pointer;
  transition: background 0.18s, color 0.18s;
  outline: none;
  box-shadow: 0 1px 3px #cccccc35;
}
.ttt__restart-btn:hover, .ttt__restart-btn:focus {
  background: #a61c1c;
  color: #fff;
}
.ttt__footer {
  margin-top: 50px;
  padding-bottom: 16px;
  font-size: 0.99em;
  opacity: 0.8;
  text-align: center;
  color: #777;
  width: 100%;
}

@media (max-width: 620px) {
  .ttt__gameboard {
    grid-template-columns: repeat(3, 18vw);
    grid-template-rows: repeat(3, 18vw);
    min-width: 0;
    min-height: 0;
    padding: 3vw;
  }
  .ttt__cell {
    min-width: 18vw;
    min-height: 18vw;
    font-size: 1.45rem;
  }
  .ttt__scores {
    flex-direction: column;
    gap: 7px;
    min-width: 0;
  }
  .ttt__mode-chooser {
    flex-direction: column;
    gap: 2px;
  }
  main {
    width: 96vw;
  }
}
</style>
