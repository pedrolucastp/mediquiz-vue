<template>
  <GameContainer
    title="Palavras Cruzadas"
    :gameInstructions="gameInstructions"
    :loading="loading"
    :score="score"
    :availablePerks="['hint', 'extra_time', 'skip']"
    @specialty-change="startGame(true)"
    @difficulty-change="startGame(true)"
    @use-perk="handlePerk"
  >
    <template #game-settings>
      <BaseButton @click="startGame(true)">Novo Jogo</BaseButton>
      <BaseButton @click="revealAnswers" variant="accent">Revelar Respostas</BaseButton>
    </template>

    <div class="crosswords-game">
      <div id="crossword-container">

        <div id="crossword" :style="gridStyle">
          <template v-for="row in gridRows" :key="row">
            <div v-for="col in gridCols" :key="`${row}-${col}`" class="cell" :class="{ empty: !getCell(row, col) }"
              :style="getCell(row, col) ? getHighlightedCellBorder(row, col) : {}">
              <template v-if="getCell(row, col)">
                <span v-if="isStartingCell(row, col)" class="clue-number">
                  {{ getWordNumber(row, col) }}
                </span>
                <input :id="`cell-${row}-${col}`" :ref="el => cellRefs[`${row}-${col}`] = el" maxLength="1"
                  :data-row="row" :data-col="col" :data-correct="getCell(row, col)" :data-words="getCellWords(row, col)"
                  :style="{ backgroundColor: getCellColor(row, col) }" @input="handleInput"
                  @click="handleCellClick(row, col)" />
              </template>
            </div>
          </template>
        </div>

        <div id="clues">
          <ul id="clue-list" :style="{ display: isLoading ? 'none' : 'block' }">
            <li v-for="word in placedWords" :key="word.number" :class="{ 'highlighted': isClueHighlighted(word.number) }"
              @click="highlightWord(word)" :style="{ borderBottom: `2px solid ${word.color}` }" :title="word.word">
              {{ word.number }}. {{ word.clue }}
              ({{ word.direction === 'across' ? 'Horizontal' : 'Vertical' }})
            </li>
          </ul>
        </div>
      </div>
      
      <div v-if="isLoading" class="loading-overlay">
        <div class="loading-message">
          <p>Gerando palavras cruzadas...</p>
          <p>Tentativa {{ currentAttempt }} de {{ GAME_ATTEMPTS }}</p>
        </div>
      </div>
    </div>
  </GameContainer>
</template>

<script setup>
import { ref, computed, onMounted, reactive } from 'vue'
import { useVocabularyStore } from '@/store/vocabulary'
import GameContainer from "@/components/game/GameContainer.vue"
import BaseButton from "@/components/base/BaseButton.vue"
import { useGamePoints } from '@/composables/useGamePoints'
import { useGameState } from '@/composables/useGameState';

const gameInstructions = `Complete as palavras cruzadas usando as dicas fornecidas.
- Clique em uma célula para selecionar uma palavra
- Use as dicas para descobrir as palavras
- Ganhe 2 pontos por cada palavra completada
- Ganhe 10 pontos por completar o jogo
- Ganhe 15 pontos de bônus por terminar sem erros
- Use a dica (5 pts) para revelar uma letra aleatória
- Todas as palavras estão relacionadas à especialidade médica selecionada`

const { usePerk, POINTS_CONFIG, awardPoints } = useGamePoints();
const { startGame: initGameState, endGame, resetGame } = useGameState();

// Constants
const PLACEMENT_DELAY = 20; // 500ms delay between attempts
const GAME_ATTEMPTS = 5;
const WORD_COUNT = 5
const GRID_ROWS = 25
const GRID_COLS = 30
const WORD_COLORS = [
  '#FF6B6B', // coral red
  '#4ECDC4', // turquoise
  '#45B7D1', // sky blue
  '#96CEB4', // sage green
  '#ff24c0', // pink
  '#D4A5A5', // dusty rose
  '#9B97B2', // muted purple
  '#FFB347', // pastel orange
  '#87CEEB', // light blue
  '#98FB98'  // pale green
]
const GRID_CENTER_ROW = Math.floor(GRID_ROWS / 2)
const GRID_CENTER_COL = Math.floor(GRID_COLS / 2)

// Refs
const grid = ref([])
const placedWords = ref([])
const currentWord = ref(null)
const currentDirection = ref('across')
const highlightedCells = ref(new Set())
const highlightedIntersections = ref(new Set())
const highlightedClue = ref(null)
const cellRefs = reactive({})
const isLoading = ref(false)
const currentAttempt = ref(1)
const loading = ref(false)
const score = ref(0)
const gameStarted = ref(false)
const pointsEarned = ref(0)

// Stores
const vocabularyStore = useVocabularyStore()

// Add a flag to track initial load
const isInitialLoad = ref(true)

// Add to the refs section
const lastClickedCell = ref({ row: null, col: null, wordIndex: 0 })

// Computed
const gridStyle = computed(() => ({
  gridTemplateColumns: `repeat(${GRID_COLS}, 40px)`,
  gridTemplateRows: `repeat(${GRID_ROWS}, 40px)`
}))

const gridRows = computed(() => Array.from({ length: GRID_ROWS }, (_, i) => i))
const gridCols = computed(() => Array.from({ length: GRID_COLS }, (_, i) => i))

// Methods
function getCell(row, col) {
  return grid.value[row]?.[col] || null
}

function getCellWords(row, col) {
  const cellWords = placedWords.value.filter(word => {
    return word.positions.some(pos => pos.row === row && pos.col === col)
  })
  return JSON.stringify(cellWords.map(word => ({
    wordNumber: word.number,
    direction: word.direction
  })))
}

function getCellColor(row, col) {
  const cellWords = placedWords.value.filter(word => {
    return word.positions.some(pos => pos.row === row && pos.col === col)
  })
  return cellWords.length > 0 ? getCell(row, col) === ' ' ? 'white' : cellWords[0].color + ((isHighlighted(row, col) || isIntersectionHighlighted(row, col)) ? '' : '70') : 'transparent';
}

//Function to help to determine the border of a highlited cell when a word is selected, apply a 3px solid darkgrey;
// when the word is in horizontal direction, 
// all cells of the current selected word should have border on top and bottom;
// the first cell should also have border on left;
// the last  cell should also have border on right; 

// when the word is in vertical direction,
// all cells of the current selected word should have border on left and right;
// the first cell should also have border on top;
// the last cell should also have border on bottom;

function getHighlightedCellBorder(row, col) {
  const cellWords = placedWords.value.filter(word => {
    return word.positions.some(pos => pos.row === row && pos.col === col)
  })

  if (cellWords.length === 0) return {}

  // Initialize styles with default border
  const styles = {
    border: '1px solid var(--dark-border-color)',
    backgroundColor: 'transparent'
  }

  // Find which word is currently highlighted (if any)
  const highlightedWord = cellWords.find(word => 
    word.positions.some(pos => highlightedCells.value.has(`${pos.row}-${pos.col}`))
  )

  if (highlightedWord) {
    styles.backgroundColor = 'rgba(255, 255, 0, 0.1)'

    // Get the cell's position within the highlighted word
    const cellIndex = highlightedWord.positions.findIndex(pos => pos.row === row && pos.col === col)
    const isFirstCell = cellIndex === 0
    const isLastCell = cellIndex === highlightedWord.positions.length - 1

    if (highlightedWord.direction === 'across') {
      styles.borderTop = '3px solid darkgrey'
      styles.borderBottom = '3px solid darkgrey'
      if (isFirstCell) styles.borderLeft = '3px solid darkgrey'
      if (isLastCell) styles.borderRight = '3px solid darkgrey'
    } else { // direction is 'down'
      styles.borderLeft = '3px solid darkgrey'
      styles.borderRight = '3px solid darkgrey'
      if (isFirstCell) styles.borderTop = '3px solid darkgrey'
      if (isLastCell) styles.borderBottom = '3px solid darkgrey'
    }
  }

  return styles
}

function isStartingCell(row, col) {
  return placedWords.value.some(word => {
    if (word.row === row && word.col === col) {
      return true;
    }
    return false;
  })
}

function getWordNumber(row, col) {
  const word = placedWords.value.find(w => w.row === row && w.col === col)
  return word?.number
}

function isHighlighted(row, col) {
  return highlightedCells.value.has(`${row}-${col}`)
}

function isIntersectionHighlighted(row, col) {
  return highlightedIntersections.value.has(`${row}-${col}`)
}

function isClueHighlighted(number) {
  return highlightedClue.value === number
}

function highlightWord(wordObj) {
  clearHighlights()

  wordObj.positions.forEach(pos => {
    highlightedCells.value.add(`${pos.row}-${pos.col}`)
  })

  highlightedClue.value = wordObj.number
}

function handleCellClick(row, col) {
  const cellWords = JSON.parse(getCellWords(row, col))
  if (cellWords.length === 0) return

  // Reset word index if clicking a different cell
  if (lastClickedCell.value.row !== row || lastClickedCell.value.col !== col) {
    lastClickedCell.value = { row, col, wordIndex: 0 }
  } else {
    // Cycle through words at the same cell
    lastClickedCell.value.wordIndex = (lastClickedCell.value.wordIndex + 1) % cellWords.length
  }

  const selectedWord = placedWords.value.find(w =>
    w.number === cellWords[lastClickedCell.value.wordIndex].wordNumber
  )

  if (selectedWord) {
    highlightWord(selectedWord)
    const input = cellRefs[`${row}-${col}`]
    if (input) {
      input.select()
    }
  }
}

function clearHighlights() {
  highlightedCells.value.clear()
  highlightedIntersections.value.clear()
  highlightedClue.value = null
}

function normalizeString(str) {
  return str.normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .toUpperCase()
    .trim()
}

function handleInput(event) {
  const input = event.target
  const cellWords = JSON.parse(input.dataset.words || '[]')

  // Set direction and highlight word if needed
  if (cellWords && cellWords.length > 0 && (!currentWord.value || !isInputPartOfCurrentWord(input))) {
    currentWord.value = cellWords[0]
    currentDirection.value = cellWords[0].direction
    const word = placedWords.value.find(w => w.number === cellWords[0].wordNumber)
    if (word) {
      highlightWord(word)
    }
  }

  // Handle backspace (when input becomes empty or no data event)
  if (!event.data || input.value === '') {
    input.value = '' // Ensure the current cell is empty
    moveToPreviousInput(input)
    return
  }

  // Handle normal typing (overwrite mode)

  input.value = event.data.toUpperCase()
  input.select()
  moveToNextInput(input)
}

function moveToNextInput(currentInput) {
  const currentRow = parseInt(currentInput.dataset.row)
  const currentCol = parseInt(currentInput.dataset.col)

  if (!currentWord.value || !currentDirection.value) return

  let nextRow = currentRow
  let nextCol = currentCol

  if (currentDirection.value === 'across') {
    nextCol++
  } else {
    nextRow++
  }

  const nextInput = cellRefs[`${nextRow}-${nextCol}`]
  if (nextInput && isInputPartOfCurrentWord(nextInput)) {
    nextInput.focus()
  }
}

function moveToPreviousInput(currentInput) {
  const currentRow = parseInt(currentInput.dataset.row)
  const currentCol = parseInt(currentInput.dataset.col)

  if (!currentWord.value || !currentDirection.value) return

  let prevRow = currentRow
  let prevCol = currentCol

  if (currentDirection.value === 'across') {
    prevCol--
  } else {
    prevRow--
  }

  const prevInput = cellRefs[`${prevRow}-${prevCol}`]
  if (prevInput && isInputPartOfCurrentWord(prevInput)) {
    prevInput.focus()
    prevInput.select() // Select text in previous cell
  }
}

function isInputPartOfCurrentWord(input) {
  if (!currentWord.value || !currentDirection.value) return false;

  const cellWords = JSON.parse(input.dataset.words || '[]')
  if (!cellWords || cellWords.length === 0) return false;

  // Check if any word in this cell matches our current direction and is part of our path
  const currentWordObj = placedWords.value.find(w => w.number === currentWord.value.wordNumber);
  if (!currentWordObj) return false;

  // Get the input's position
  const row = parseInt(input.dataset.row);
  const col = parseInt(input.dataset.col);

  // Check if this position is part of our current word's path
  if (currentDirection.value === 'across') {
    return row === currentWordObj.row &&
      col >= currentWordObj.col &&
      col < currentWordObj.col + currentWordObj.word.length;
  } else {
    return col === currentWordObj.col &&
      row >= currentWordObj.row &&
      row < currentWordObj.row + currentWordObj.word.length;
  }
}

function calculateGamePoints(mistakes) {
  const basePoints = WORD_COUNT * POINTS_CONFIG.WORD_FOUND;
  const isPerfect = mistakes === 0;
  return basePoints + POINTS_CONFIG.GAME_COMPLETION + (isPerfect ? POINTS_CONFIG.PERFECT_SCORE : 0);
}

async function handleGameCompletion(mistakes = 0) {
  endGame();
  const points = calculateGamePoints(mistakes);
  await awardPoints(points);
  pointsEarned.value = points;
  alert(`Parabéns! Você completou o jogo${mistakes === 0 ? ' sem erros' : ''}!\nPontos ganhos: ${points}`);
}

function initializeGrid() {
  // Clear the grid array
  grid.value = []
  for (let i = 0; i < GRID_ROWS; i++) {
    grid.value.push(new Array(GRID_COLS).fill(null))
  }

  // Clear all input values and styling
  for (const key in cellRefs) {
    const input = cellRefs[key]
    if (input) {
      input.value = ''
      input.style.backgroundColor = 'transparent'
      input.style.color = 'var(--text-color)'
    }
  }

  // Clear placed words and highlights
  placedWords.value = []
  clearHighlights()

  // Reset any game state
  currentWord.value = null
  currentDirection.value = 'across'

  grid.value.forEach(row => row.fill(null));
}


function canPlaceWord(word, row, col, direction) {
  if (direction === 'across') {
    if (col < 0 || col + word.length > GRID_COLS || row < 0 || row >= GRID_ROWS) return false;

    let hasIntersection = false;
    for (let i = 0; i < word.length; i++) {
      const currentCell = grid.value[row][col + i];
      if (currentCell) {
        if (currentCell !== word[i]) return false;
        hasIntersection = true;
      }

      const above = row > 0 ? grid.value[row - 1][col + i] : null;
      const below = row < GRID_ROWS - 1 ? grid.value[row + 1][col + i] : null;

      if ((above && !isPartOfWord(row - 1, col + i)) ||
        (below && !isPartOfWord(row + 1, col + i))) {
        return false;
      }
    }

    const before = col > 0 ? grid.value[row][col - 1] : null;
    const after = col + word.length < GRID_COLS ? grid.value[row][col + word.length] : null;
    if (before || after) return false;

    return placedWords.value.length === 0 || hasIntersection;

  } else if (direction === 'down') {
    if (row < 0 || row + word.length > GRID_ROWS || col < 0 || col >= GRID_COLS) return false;

    let hasIntersection = false;
    for (let i = 0; i < word.length; i++) {
      const currentCell = grid.value[row + i][col];
      if (currentCell) {
        if (currentCell !== word[i]) return false;
        hasIntersection = true;
      }

      const left = col > 0 ? grid.value[row + i][col - 1] : null;
      const right = col < GRID_COLS - 1 ? grid.value[row + i][col + 1] : null;

      if ((left && !isPartOfWord(row + i, col - 1)) ||
        (right && !isPartOfWord(row + i, col + 1))) {
        return false;
      }
    }

    const above = row > 0 ? grid.value[row - 1][col] : null;
    const below = row + word.length < GRID_ROWS ? grid.value[row + word.length][col] : null;
    if (above || below) return false;

    return placedWords.value.length === 0 || hasIntersection;
  }
  return false;
}

function isPartOfWord(row, col) {
  return placedWords.value.some(wordObj => {
    if (wordObj.direction === 'across') {
      return row === wordObj.row && col >= wordObj.col && col < wordObj.col + wordObj.word.length
    } else if (wordObj.direction === 'down') {
      return col === wordObj.col && row >= wordObj.row && row < wordObj.row + wordObj.word.length
    }
    return false
  })
}

async function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function visualizePlacement(word, row, col, direction, temp = true) {
  const positions = [];
  if (direction === 'across') {
    for (let i = 0; i < word.length; i++) {
      if (grid.value[row][col + i] === null) {
        grid.value[row][col + i] = temp ? '?' : word[i];
        positions.push({ row, col: col + i });
      }
    }
  } else {
    for (let i = 0; i < word.length; i++) {
      if (grid.value[row + i][col] === null) {
        grid.value[row + i][col] = temp ? '?' : word[i];
        positions.push({ row: row + i, col });
      }
    }
  }

  if (temp) {
    await sleep(PLACEMENT_DELAY);
    positions.forEach(pos => {
      if (grid.value[pos.row][pos.col] === '?') {
        grid.value[pos.row][pos.col] = null;
      }
    });
  }
  return positions;
}

function countIntersections(word, row, col, direction) {
  let intersections = 0;

  if (direction === 'across') {
    for (let i = 0; i < word.length; i++) {
      if (grid.value[row][col + i] !== null && grid.value[row][col + i] === word[i]) {
        intersections++;
      }
    }
  } else {
    for (let i = 0; i < word.length; i++) {
      if (grid.value[row + i][col] !== null && grid.value[row + i][col] === word[i]) {
        intersections++;
      }
    }
  }

  return intersections;
}

function findAllPossiblePositions(word, direction) {
  const positions = [];

  for (let row = 0; row < GRID_ROWS; row++) {
    for (let col = 0; col < GRID_COLS; col++) {
      if (canPlaceWord(word, row, col, direction)) {
        const intersections = countIntersections(word, row, col, direction);
        positions.push({
          row,
          col,
          direction,
          intersections
        });
      }
    }
  }

  // Sort by number of intersections in descending order
  positions.sort((a, b) => b.intersections - a.intersections);

  return positions;
}

async function placeRemainingWords(availableWords) {

  while (placedWords.value.length < WORD_COUNT) {
    let placed = false;
    const triedIndices = new Set();

    // Keep trying random words until we place one or exhaust all options
    while (!placed && triedIndices.size < availableWords.length) {
      let randomIndex;
      do {
        randomIndex = Math.floor(Math.random() * availableWords.length);
      } while (triedIndices.has(randomIndex));

      triedIndices.add(randomIndex);
      const word = availableWords[randomIndex];
      const allPossiblePositions = [];

      // Find all possible positions in both directions
      for (const direction of ['across', 'down']) {
        const positionsInDirection = await findAllPossiblePositions(word.word, direction);
        allPossiblePositions.push(...positionsInDirection);
      }

      // If we found valid positions
      if (allPossiblePositions.length > 0) {
        // Sort by intersections
        allPossiblePositions.sort((a, b) => b.intersections - a.intersections);

        // Take top 3 positions when available
        const topPositions = allPossiblePositions.slice(0, Math.min(3, allPossiblePositions.length));

        // Randomly select from top positions
        const selectedPosition = topPositions[Math.floor(Math.random() * topPositions.length)];

        // Visualize the attempt
        await visualizePlacement(word.word, selectedPosition.row, selectedPosition.col, selectedPosition.direction, true);

        // Place the word
        const positions = await visualizePlacement(word.word, selectedPosition.row, selectedPosition.col, selectedPosition.direction, false);

        if (positions.length > 0) {
          placedWords.value.push({
            word: word.word,
            clue: word.clue,
            row: selectedPosition.row,
            col: selectedPosition.col,
            direction: selectedPosition.direction,
            positions: positions,
            number: placedWords.value.length + 1,

            // Assign a color based on the amount of available colors.
            // color: WORD_COLORS[placedWords.value.length]
            color: WORD_COLORS[placedWords.value.length % WORD_COLORS.length]
          });
          availableWords.splice(randomIndex, 1);
          placed = true;
          break;
        }
      }
    }

    if (!placed) {
      console.log('No more words can be placed with current configuration');
      return false;
    }
  }

  return placedWords.value.length === WORD_COUNT;
}

async function generateCrossword(words) {
  if (currentAttempt.value > GAME_ATTEMPTS) {
    console.log('Maximum attempts reached');
    return false;
  }

  console.log(`Attempt ${currentAttempt.value} of 10`);
  initializeGrid();
  setTimeout(() => scrollToGridCenter(), 100)


  // Create a copy of words array to work with
  const availableWords = [...words];
  const triedIndices = new Set();

  // Try to place first word
  while (triedIndices.size < availableWords.length) {
    let randomIndex;
    do {
      randomIndex = Math.floor(Math.random() * availableWords.length);

    } while (triedIndices.has(randomIndex));

    triedIndices.add(randomIndex);
    const firstWord = availableWords[randomIndex];

    // Try to place the first word horizontally in the center
    const positions = await visualizePlacement(
      firstWord.word,
      GRID_CENTER_ROW,
      Math.floor(GRID_CENTER_COL - firstWord.word.length / 2),
      'across',
      false
    );

    if (positions.length > 0) {
      placedWords.value.push({
        word: firstWord.word,
        clue: firstWord.clue,
        row: GRID_CENTER_ROW,
        col: Math.floor(GRID_CENTER_COL - firstWord.word.length / 2),
        direction: 'across',
        positions: positions,
        number: 1,
        color: WORD_COLORS[0]
      });

      availableWords.splice(randomIndex, 1);

      // Try to place remaining words with the new strategy
      const success = await placeRemainingWords(availableWords);
      if (success) {
        return true;
      }
    }
  }

  currentAttempt.value++;
  return false;
}

function scrollToGridCenter() {
  const crosswordDiv = document.getElementById('crossword')
  if (crosswordDiv) {
    const scrollLeft = (crosswordDiv.scrollWidth - crosswordDiv.clientWidth) / 2
    const scrollTop = (crosswordDiv.scrollHeight - crosswordDiv.clientHeight) / 2
    crosswordDiv.scrollTo({
      left: scrollLeft,
      top: scrollTop,
      behavior: 'smooth'
    })
  }
}

async function startGame(force = false) {
  resetGame();
  loading.value = true;
  isLoading.value = true;
  
  try {
    // Initialize game state
    if (!initGameState()) {
      throw new Error('Failed to initialize game state');
    }
    
    const words = vocabularyStore.filteredWords
    const success = await generateCrossword(words)

    if (!success && currentAttempt.value < GAME_ATTEMPTS) {
      await startGame(true)
    }
  } catch (error) {
    console.error('Error generating crossword:', error)
  } finally {
    isLoading.value = false
    loading.value = false
    clearHighlights()
    isInitialLoad.value = false
  }
}

/**
 * @function handlePerk
 * @param {string} perkId - The perk to activate
 * @returns {Promise<void>}
 * @description Deducts points and applies the perk effect. 'hint' reveals a random letter in the grid.
 */
async function handlePerk(perkId) {
  const success = await usePerk(perkId);
  if (!success) return;

  if (perkId === 'hint') {
    // Reveal a random letter in the crossword grid
    const allInputs = Object.values(cellRefs).filter(input => 
      input && 
      input.value === '' && 
      input.dataset.letter
    );
    
    if (allInputs.length > 0) {
      const randomInput = allInputs[Math.floor(Math.random() * allInputs.length)];
      const letter = randomInput.dataset.letter;
      randomInput.value = letter;
      randomInput.classList.add('revealed');
      checkWord(randomInput);
    }
  } else if (perkId === 'skip') {
    // Skip current grid and generate a new one
    await startGame(true);
  } else if (perkId === 'extra_time') {
    // Add 30 seconds to the timer if it exists
    if (typeof timeLeft !== 'undefined' && timeLeft) {
      timeLeft.value += 30;
    }
  }
}

/**
 * @function revealAnswers
 * @description Reveals all answers in the crossword grid and updates the score to 0.
 */
function revealAnswers() {
  // Reveal all answers and mark them in red
  for (const key in cellRefs) {
    const input = cellRefs[key]
    if (input && input.dataset.correct) {
      input.value = input.dataset.correct
      input.style.color = '#FF0000'
    }
  }
  // Update the score to 0 since answers were revealed
  score.value = 0
}

// Initialization
onMounted(() => {
  startGame()
})
</script>

<style scoped>
#crossword-container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  gap: var(--spacing-lg, 2rem);
  padding: var(--spacing-md, 1rem);
  max-width: 1400px;
  margin: 0 auto;
  width: 100%;
  min-height: calc(100vh - 200px);
  position: relative;
}

.crosswords-game {
  touch-action: none;
  /* Prevent zooming on the page */
  -ms-content-zooming: none;
  -ms-touch-action: none;
}

#clues {
  position: absolute;
  top: var(--spacing-md, 1rem);
  left: var(--spacing-md, 1rem);
  width: 350px;
  flex-shrink: 0;
  padding: var(--spacing-md, 1rem);
  border-radius: var(--radius-md, 8px);
  box-shadow: var(--shadow-sm);
  height: 100%;
  display: flex;
  flex-direction: column;
  max-height: 80vh;
}


#crossword {
  display: grid;
  gap: 0;
  margin: 0;
  padding: 2rem 2rem 2rem 350px;
  /* border-radius: var(--radius-md); */
  box-shadow: var(--shadow-md);
  overflow: auto;
  max-height: 80vh;
  border: 2px solid var(--dark-border-color);
  touch-action: manipulation;
  -webkit-overflow-scrolling: touch;
  -webkit-user-select: none;
  user-select: none;
  transform: scale(1);
  transform-origin: center;
  touch-action: pan-x pan-y pinch-zoom;
  width: 100%;
  position: relative;
}

/* Scrollbar styling */
::-webkit-scrollbar {
  width: 3px;
  height: 3px;
}

::-webkit-scrollbar-track {
  background: transparent;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb {
  background: var(--primary-color);

  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: var(--accent-color);
}

@media (max-width: 768px) {
  #crossword-container {
    flex-direction: column;
    align-items: center;
    padding: 20px 0;
  }

  #crossword {
    width: 90%;
    order: 2;
    padding: 10px;
  }

  #clues {
    width: 100%;
    position: relative;
    top: 0;
    left: 0;
    order: 1;
    max-height: none;
  }
}

.cell {
  width: 40px;
  height: 40px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: center;
  /* Remove the default border since we handle it in getHighlightedCellBorder */
  /* border: 1px solid var(--dark-border-color); */
}

.empty {
  border: none;
  background-color: transparent;
}

.cell input {
  width: 100%;
  height: 100%;
  text-align: center;
  font-size: 16px;
  padding: 0;
  margin: 0;
  border: none;
  outline: none;
  background: transparent;
  text-transform: uppercase;
  box-sizing: border-box;
  border-radius: 0;
}

.cell:hover {
  background-color: var(--hover-color, rgba(0, 0, 0, 0.05));
}

.cell input:focus {
  background-color: var(--focus-color, rgba(var(--primary-rgb, 46, 204, 113), 0.1));
}

.cell input.highlighted {
  background-color: var(--highlight-color, rgba(255, 255, 0, 0.3));
}

.cell input.highlighted-intersection {
  background-color: var(--intersection-highlight-color, rgba(0, 255, 255, 0.3));
}

.cell .clue-number {
  position: relative;
  top: -8px;
  left: 2px;
  font-size: var(--clue-number-size, 0.75rem);
  font-weight: bold;
  color: var(--text-color);
  width: 0;
}

/* #clues h2 {
  font-size: 24px;
  margin-bottom: 10px;
  color: var(--text-color);
} */

#clue-list {
  list-style: none;
  margin-top: 20px;
  overflow-y: auto;
  list-style-position: outside;
}

#clue-list li {
  margin-bottom: 0px;
  font-size: 1rem;
  text-align: left;
  /* color: #eee; */
  word-wrap: break-word;
  cursor: pointer;
  padding: 4px;
}

#clue-list li.highlighted {
  background-color: var(--highlight-color, rgba(255, 255, 0, 0.3));
  font-size: 1.1rem;
}

.button-container {
  display: none;
}

@media (max-width: 480px) {
  .cell {
    width: 100%;
    height: 100%;
  }

  .cell input {
    font-size: 14px;
  }
}

.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.4);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.loading-message {
  text-align: center;
  padding: 20px;
  background-color: var(--surface-color, #fff);
  border-radius: 8px;
  box-shadow: var(--shadow-sm);
  border: 1px solid var(--border-color);
}

:deep(.dark) .loading-overlay {
  background: rgba(0, 0, 0, 0.4);
}

:deep(.dark) .cell {
  border-color: var(--dark-border-color);
}

:deep(.dark) .cell:hover {
  background-color: var(--dark-hover-color, rgba(255, 255, 255, 0.1));
}

:deep(.dark) .cell input {
  color: var(--dark-text-color);
}

:deep(.dark) .cell input:focus {
  background-color: var(--dark-focus-color, rgba(var(--dark-primary-rgb, 46, 204, 113), 0.2));
}

:deep(.dark) .cell input.highlighted {
  background-color: var(--dark-highlight-color, rgba(255, 255, 0, 0.3));
}

:deep(.dark) .cell input.highlighted-intersection {
  background-color: var(--dark-intersection-highlight-color, rgba(0, 255, 255, 0.3));
}

:deep(.dark) .cell .clue-number {
  color: var(--dark-text-color);
}

/* :deep(.dark) #clues h2 {
  color: var(--dark-text-color);
} */

:deep(.dark) #clue-list li {
  color: var(--dark-text-color);
}

:deep(.dark) #clue-list li.highlighted {
  background-color: var(--dark-highlight-color, rgba(255, 255, 0, 0.3));
}
</style>