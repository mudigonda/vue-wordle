<script setup lang="ts">
import { onUnmounted } from 'vue';
import { getWordOfTheDay, allWords } from './words';
import Keyboard from './Keyboard.vue';
import { LetterState } from './types';

// Get word of the day
const answer = getWordOfTheDay();

// Board state. Each tile is represented as { letter, state }
const board = $ref(
  Array.from({ length: 6 }, () =>
    Array.from({ length: 5 }, () => ({
      letter: '',
      state: LetterState.INITIAL,
    }))
  )
);

// Current active row.
let currentRowIndex = $ref(0);
const currentRow = $computed(() => board[currentRowIndex]);

// Feedback state: message and shake
let message = $ref('');
let grid = $ref('');
let shakeRowIndex = $ref(-1);
let success = $ref(false);
let info = $ref(false);
let showModal = $ref(false);
let wordOfTheDay = $ref('');

// Keep track of revealed letters for the virtual keyboard
const letterStates: Record<string, LetterState> = $ref({});

// Handle keyboard input.
let allowInput = true;

const onKeyup = (e: KeyboardEvent) => onKey(e.key);

window.addEventListener('keyup', onKeyup);

onUnmounted(() => {
  window.removeEventListener('keyup', onKeyup);
});

function onKey(key: string) {
  if (!allowInput) return;
  if (/^[a-zA-Z]$/.test(key)) {
    fillTile(key.toLowerCase());
  } else if (key === 'Backspace') {
    clearTile();
  } else if (key === 'Enter') {
    completeRow();
  }
}

function fillTile(letter: string) {
  for (const tile of currentRow) {
    if (!tile.letter) {
      tile.letter = letter;
      break;
    }
  }
}

function clearTile() {
  for (const tile of [...currentRow].reverse()) {
    if (tile.letter) {
      tile.letter = '';
      break;
    }
  }
}

function completeRow() {
  if (currentRow.every((tile) => tile.letter)) {
    const guess = currentRow.map((tile) => tile.letter).join('');
    if (!allWords.includes(guess) && guess !== answer) {
      shake();
      showMessage(`Not in word list`);
      return;
    }

    const answerLetters: (string | null)[] = answer.split('');
    // first pass: mark correct ones
    currentRow.forEach((tile, i) => {
      if (answerLetters[i] === tile.letter) {
        tile.state = letterStates[tile.letter] = LetterState.CORRECT;
        answerLetters[i] = null;
      }
    });
    // second pass: mark the present
    currentRow.forEach((tile) => {
      if (!tile.state && answerLetters.includes(tile.letter)) {
        tile.state = LetterState.PRESENT;
        answerLetters[answerLetters.indexOf(tile.letter)] = null;
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.PRESENT;
        }
      }
    });
    // 3rd pass: mark absent
    currentRow.forEach((tile) => {
      if (!tile.state) {
        tile.state = LetterState.ABSENT;
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.ABSENT;
        }
      }
    });

    allowInput = false;
    if (currentRow.every((tile) => tile.state === LetterState.CORRECT)) {
      // yay!
      setTimeout(() => {
        grid = genResultGrid();
        showMessage(
          ['Genius', 'Magnificent', 'Impressive', 'Splendid', 'Great', 'Phew'][
            currentRowIndex
          ],
          500
        );
        success = true;
        setTimeout(() => {
          showModal = true;
        }, 1000);
      }, 1600);
    } else if (currentRowIndex < board.length - 1) {
      // go the next row
      currentRowIndex++;
      setTimeout(() => {
        allowInput = true;
      }, 1600);
    } else {
      // game over :(
      setTimeout(() => {
        showMessage(answer.toUpperCase(), -1);
      }, 1600);
    }
  } else {
    shake();
    showMessage('Not enough letters');
  }
}

function showMessage(msg: string, time = 1000) {
  message = msg;
  if (time > 0) {
    setTimeout(() => {
      message = '';
    }, time);
  }
}

function shake() {
  shakeRowIndex = currentRowIndex;
  setTimeout(() => {
    shakeRowIndex = -1;
  }, 1000);
}

const icons = {
  [LetterState.CORRECT]: '🟩',
  [LetterState.PRESENT]: '🟨',
  [LetterState.ABSENT]: '⬜',
  [LetterState.INITIAL]: null,
};

function genResultGrid() {
  return board
    .slice(0, currentRowIndex + 1)
    .map((row) => {
      return row.map((tile) => icons[tile.state]).join('');
    })
    .join('\n');
}

function getResultsText() {
  let attempts = currentRowIndex + 1;
  return `#savesoil wordle ${attempts}/6\n${grid}`;
}

function share() {
  const el = document.createElement('textarea');
  el.setAttribute('readonly', '');
  el.style.position = 'absolute';
  el.style.left = '-9999px';
  el.value = getResultsText();
  document.body.appendChild(el);
  el.select();
  document.execCommand('copy');
  info = true;
  document.body.removeChild(el);
  setTimeout(function () {
    info = false;
  }, 1000);
}
</script>

<template>
  <Transition>
    <div class="message" v-if="message">
      {{ message }}
    </div>
  </Transition>
  <Transition name="fade" appear>
    <div class="modal-overlay" v-if="showModal">
      <div class="modal">
        <div class="close-icon" @click="showModal = false">
          <i class="fa fa-times"></i>
        </div>
        <a
          href="https://consciousplanet.org/"
          target="_blank"
          style="cursor: pointer"
        >
          <img
            class="savesoil-img"
            v-if="grid"
            src="http://savesoil.me/wp-content/uploads/2022/03/conciousplanet-savesoil.jpg"
            alt="savesoil"
          />
        </a>
        <div class="results-wrapper">
          <div v-if="grid" class="results-grid">
            <pre id="gridRef" ref="gridRef">{{ grid }}</pre>
          </div>
          <div v-if="grid" class="share-wrapper">
            <button class="share-icon" @click="share">
              Share &nbsp;&nbsp;&nbsp; <i class="fa fa-share-alt"></i>
            </button>
            <p v-if="info" class="share-text">Copied results to clipboard!</p>
          </div>
        </div>
      </div>
    </div>
  </Transition>
  <header>
    <h1>SAVESOIL WORDLE</h1>
    <!-- <a
      id="source-link"
      href="https://github.com/yyx990803/vue-wordle"
      target="_blank"
      >Source</a
    > -->
  </header>
  <div id="board">
    <div
      v-for="(row, index) in board"
      :key="index"
      :class="[
        'row',
        shakeRowIndex === index && 'shake',
        success && currentRowIndex === index && 'jump',
      ]"
    >
      <div
        v-for="(tile, index) in row"
        :key="index"
        :class="['tile', tile.letter && 'filled', tile.state && 'revealed']"
      >
        <div class="front" :style="{ transitionDelay: `${index * 300}ms` }">
          {{ tile.letter }}
        </div>
        <div
          :class="['back', tile.state]"
          :style="{
            transitionDelay: `${index * 300}ms`,
            animationDelay: `${index * 100}ms`,
          }"
        >
          {{ tile.letter }}
        </div>
      </div>
    </div>
  </div>
  <Keyboard @key="onKey" :letter-states="letterStates" />
</template>

<style scoped>
#board {
  display: grid;
  grid-template-rows: repeat(6, 1fr);
  grid-gap: 5px;
  padding: 10px;
  box-sizing: border-box;
  --height: min(420px, calc(var(--vh, 100vh) - 310px));
  height: var(--height);
  width: min(350px, calc(var(--height) / 6 * 5));
  margin: 0px auto;
}
.message {
  position: absolute;
  left: 50%;
  top: 80px;
  color: #fff;
  background-color: rgba(0, 0, 0, 0.85);
  padding: 16px 20px;
  z-index: 2;
  border-radius: 4px;
  transform: translateX(-50%);
  transition: opacity 0.3s ease-out;
  font-weight: 600;
}
.message.v-leave-to {
  opacity: 0;
}
.results-wrapper {
  display: flex;
  margin: 10px 0px 20px 0px;
  align-items: center;
}
.results-grid {
  flex: 3;
}
.share-wrapper {
  position: relative;
  text-align: left;
  flex: 2;
}
.share-icon {
  background-color: #65a135;
  color: white;
  padding: 10px 30px;
  border: 1px solid #65a135;
  outline: none;
  cursor: pointer;
  margin: 0 10px;
}
.share-text {
  position: absolute;
  font-size: 12px;
  opacity: 0.5;
}
.row {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  grid-gap: 5px;
}
.tile {
  width: 100%;
  font-size: 2rem;
  line-height: 2rem;
  font-weight: bold;
  vertical-align: middle;
  text-transform: uppercase;
  user-select: none;
  position: relative;
}
.tile.filled {
  animation: zoom 0.2s;
}
.tile .front,
.tile .back {
  box-sizing: border-box;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: transform 0.6s;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}
.tile .front {
  border: 2px solid #d3d6da;
}
.tile.filled .front {
  border-color: #999;
}
.tile .back {
  transform: rotateX(180deg);
}
.tile.revealed .front {
  transform: rotateX(180deg);
}
.tile.revealed .back {
  transform: rotateX(0deg);
}

.modal-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 98;
  background-color: rgba(0, 0, 0, 0.3);
}
.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  transition: opacity 1s ease-in;
  z-index: 99;
  width: 100%;
  max-width: 45%;
  background-color: #fff;
  border-radius: 5px;
  padding: 10px 0;
}
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
.close-icon {
  cursor: pointer;
  opacity: 0.5;
  cursor: pointer;
  opacity: 0.5;
  text-align: right;
  margin-right: 15px;
}
.savesoil-img {
  width: 450px;
}

@keyframes zoom {
  0% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
  }
}

.shake {
  animation: shake 0.5s;
}

@keyframes shake {
  0% {
    transform: translate(1px);
  }
  10% {
    transform: translate(-2px);
  }
  20% {
    transform: translate(2px);
  }
  30% {
    transform: translate(-2px);
  }
  40% {
    transform: translate(2px);
  }
  50% {
    transform: translate(-2px);
  }
  60% {
    transform: translate(2px);
  }
  70% {
    transform: translate(-2px);
  }
  80% {
    transform: translate(2px);
  }
  90% {
    transform: translate(-2px);
  }
  100% {
    transform: translate(1px);
  }
}

.jump .tile .back {
  animation: jump 0.5s;
}

@keyframes jump {
  0% {
    transform: translateY(0px);
  }
  20% {
    transform: translateY(5px);
  }
  60% {
    transform: translateY(-25px);
  }
  90% {
    transform: translateY(3px);
  }
  100% {
    transform: translateY(0px);
  }
}

@media (max-height: 680px) {
  .tile {
    font-size: 3vh;
  }
}
@media only screen and (max-width: 600px) {
  .modal {
    max-width: unset;
  }
  .savesoil-img {
    width: 350px;
  }
}
</style>
