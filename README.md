const BOARD_SIZE = 8;
const MAX_PIECES = 3;
const PIECE_LIBRARY = [
  { color: '#ffb703', cells: [[0, 0]] },
  { color: '#fb8500', cells: [[0, 0], [1, 0]] },
  { color: '#ff5d8f', cells: [[0, 0], [1, 0], [2, 0]] },
  { color: '#8ecae6', cells: [[0, 0], [0, 1]] },
  { color: '#219ebc', cells: [[0, 0], [1, 0], [0, 1]] },
  { color: '#90be6d', cells: [[0, 0], [1, 0], [1, 1]] },
  { color: '#a78bfa', cells: [[0, 0], [1, 0], [0, 1], [1, 1]] },
  { color: '#f46036', cells: [[0, 0], [1, 0], [2, 0], [2, 1]] },
  { color: '#f72585', cells: [[0, 0], [0, 1], [1, 1], [2, 1]] },
  { color: '#06d6a0', cells: [[0, 0], [1, 0], [1, 1], [2, 1]] },
  { color: '#ffd166', cells: [[0, 0], [1, 0], [2, 0], [0, 1]] },
  { color: '#4cc9f0', cells: [[0, 0], [1, 0], [2, 0], [1, 1]] }
];

const boardEl = document.getElementById('board');
const trayEl = document.getElementById('piece-tray');
const scoreEl = document.getElementById('score');
const bestScoreEl = document.getElementById('best-score');
const statusEl = document.getElementById('status');
const newGameBtn = document.getElementById('new-game');
const rotateBtn = document.getElementById('rotate-piece');

let board = [];
let tray = [];
let score = 0;
let selectedPieceIndex = null;
let hoverAnchor = null;
let previewCells = [];
let gameOver = false;
let rotationStep = 0;

const bestScore = Number(localStorage.getItem('blockBlastBest') || 0);
bestScoreEl.textContent = bestScore;

function cloneBoard() {
  return Array.from({ length: BOARD_SIZE }, (_, row) =>
    Array.from({ length: BOARD_SIZE }, (_, col) => board[row][col])
  );
}

function randomPieceTemplate() {
  const template = PIECE_LIBRARY[Math.floor(Math.random() * PIECE_LIBRARY.length)];
  return {
    color: template.color,
    cells: template.cells.map(([row, col]) => [row, col]),
    rotation: 0,
  };
}

function rotatePieceCells(cells) {
  const maxRow = Math.max(...cells.map(([row]) => row));
  const maxCol = Math.max(...cells.map(([, col]) => col));
  const rotated = cells.map(([row, col]) => [maxCol - col, row]);
  const minRow = Math.min(...rotated.map(([row]) => row));
  const minCol = Math.min(...rotated.map(([, col]) => col));

  return rotated.map(([row, col]) => [row - minRow, col - minCol]);
}

function getPieceCells(piece) {
  let cells = piece.cells.map(([row, col]) => [row, col]);
  for (let i = 0; i < piece.rotation; i += 1) {
    cells = rotatePieceCells(cells);
  }
  return cells;
}

function resetBoard() {
  board = Array.from({ length: BOARD_SIZE }, () => Array(BOARD_SIZE).fill(null));
}

function refillTray() {
  while (tray.length < MAX_PIECES) {
    tray.push(randomPieceTemplate());
  }
}

function ensurePlayablePieces() {
  const anyPlayable = tray.some((piece) => canPlaceAny(piece));
  if (!anyPlayable && !gameOver) {
    gameOver = true;
    statusEl.textContent = 'No moves left. Tap New Game to play again.';
  }
}

function createBoardCells() {
  boardEl.innerHTML = '';
  for (let row = 0; row < BOARD_SIZE; row += 1) {
    for (let col = 0; col < BOARD_SIZE; col += 1) {
      const cell = document.createElement('button');
      cell.type = 'button';
      cell.className = 'board-cell';
      cell.dataset.row = row;
      cell.dataset.col = col;
      cell.setAttribute('aria-label', `Board cell ${row + 1}, ${col + 1}`);
      cell.addEventListener('pointermove', handleBoardHover);
      cell.addEventListener('pointerleave', clearPreview);
      cell.addEventListener('click', handleBoardClick);
      boardEl.appendChild(cell);
    }
  }
}

function renderBoard() {
  const cells = [...boardEl.children];
  cells.forEach((cell) => {
    const row = Number(cell.dataset.row);
    const col = Number(cell.dataset.col);
    const filledColor = board[row][col];
    cell.classList.toggle('filled', !!filledColor);
    cell.classList.toggle('preview', false);
    cell.style.background = filledColor || '';
    cell.style.boxShadow = filledColor ? 'inset 0 0 0 1px rgba(255,255,255,0.18)' : '';
  });

  previewCells.forEach(([row, col]) => {
    const cell = cells.find((item) => Number(item.dataset.row) === row && Number(item.dataset.col) === col);
    if (cell) {
      cell.classList.add('preview');
      cell.style.background = '#dbeafe';
      cell.style.opacity = '0.7';
    }
  });
}

function renderTray() {
  trayEl.innerHTML = '';

  tray.forEach((piece, index) => {
    const card = document.createElement('div');
    card.className = `piece-card ${selectedPieceIndex === index ? 'selected' : ''}`;
    card.addEventListener('click', () => selectPiece(index));

    const preview = document.createElement('div');
    preview.className = 'piece-preview';
    const cells = getPieceCells(piece);
    const maxX = Math.max(...cells.map(([, col]) => col));
    const maxY = Math.max(...cells.map(([row]) => row));
    const gridSize = Math.max(maxX + 1, maxY + 1, 1);

    preview.style.gridTemplateColumns = `repeat(${Math.min(gridSize, 3)}, 16px)`;
    for (let row = 0; row < 3; row += 1) {
      for (let col = 0; col < 3; col += 1) {
        const cell = document.createElement('span');
        cell.className = 'preview-cell';
        const hasPiece = cells.some(([r, c]) => r === row && c === col);
        if (hasPiece) {
          cell.classList.add('filled');
          cell.style.background = piece.color;
        }
        preview.appendChild(cell);
      }
    }

    card.appendChild(preview);
    trayEl.appendChild(card);
  });
}

function updateScore() {
  scoreEl.textContent = score;
  if (score > bestScore) {
    localStorage.setItem('blockBlastBest', score);
    bestScoreEl.textContent = score;
  }
}

function selectPiece(index) {
  if (gameOver) return;
  selectedPieceIndex = selectedPieceIndex === index ? null : index;
  hoverAnchor = null;
  previewCells = [];
  statusEl.textContent = selectedPieceIndex !== null
    ? 'Placement active: click a board cell to drop the piece.'
    : 'Select a piece to begin.';
  renderTray();
  renderBoard();
}

function clearPreview() {
  hoverAnchor = null;
  previewCells = [];
  renderBoard();
}

function handleBoardHover(event) {
  if (selectedPieceIndex === null || gameOver) return;
  const row = Number(event.currentTarget.dataset.row);
  const col = Number(event.currentTarget.dataset.col);
  hoverAnchor = { row, col };
  const piece = tray[selectedPieceIndex];
  previewCells = getPreviewCells(piece, row, col);
  renderBoard();
}

function getPreviewCells(piece, row, col) {
  const cells = getPieceCells(piece);
  const preview = [];
  for (const [offsetRow, offsetCol] of cells) {
    const nextRow = row + offsetRow;
    const nextCol = col + offsetCol;
    if (nextRow >= 0 && nextRow < BOARD_SIZE && nextCol >= 0 && nextCol < BOARD_SIZE) {
      preview.push([nextRow, nextCol]);
    }
  }
  return preview;
}

function canPlacePiece(piece, anchorRow, anchorCol) {
  const cells = getPieceCells(piece);
  for (const [offsetRow, offsetCol] of cells) {
    const row = anchorRow + offsetRow;
    const col = anchorCol + offsetCol;
    if (row < 0 || row >= BOARD_SIZE || col < 0 || col >= BOARD_SIZE) {
      return false;
    }
    if (board[row][col] !== null) {
      return false;
    }
  }
  return true;
}

function canPlaceAny(piece) {
  for (let row = 0; row < BOARD_SIZE; row += 1) {
    for (let col = 0; col < BOARD_SIZE; col += 1) {
      if (canPlacePiece(piece, row, col)) return true;
    }
  }
  return false;
}

function clearCompletedLines() {
  let cleared = 0;

  for (let row = 0; row < BOARD_SIZE; row += 1) {
    if (board[row].every(Boolean)) {
      board[row].fill(null);
      cleared += 1;
    }
  }

  for (let col = 0; col < BOARD_SIZE; col += 1) {
    if (board.every((rowCells) => !!rowCells[col])) {
      for (const rowCells of board) {
        rowCells[col] = null;
      }
      cleared += 1;
    }
  }

  if (cleared > 0) {
    score += cleared * 75;
    updateScore();
    statusEl.textContent = cleared > 1 ? `Nice! You cleared ${cleared} lines.` : 'Nice! You cleared a line.';
  }
}

function handleBoardClick(event) {
  if (selectedPieceIndex === null || gameOver) return;

  const row = Number(event.currentTarget.dataset.row);
  const col = Number(event.currentTarget.dataset.col);
  const piece = tray[selectedPieceIndex];

  if (!canPlacePiece(piece, row, col)) {
    statusEl.textContent = 'That placement does not fit. Try a different spot.';
    return;
  }

  const cells = getPieceCells(piece);
  for (const [offsetRow, offsetCol] of cells) {
    const boardRow = row + offsetRow;
    const boardCol = col + offsetCol;
    board[boardRow][boardCol] = piece.color;
  }

  tray.splice(selectedPieceIndex, 1);
  selectedPieceIndex = null;
  hoverAnchor = null;
  previewCells = [];
  refillTray();
  clearCompletedLines();
  ensurePlayablePieces();

  if (!gameOver) {
    statusEl.textContent = 'Great move! Pick another piece.';
  }

  renderBoard();
  renderTray();
}

function startNewGame() {
  gameOver = false;
  score = 0;
  tray = [];
  selectedPieceIndex = null;
  hoverAnchor = null;
  previewCells = [];
  rotationStep = 0;
  resetBoard();
  refillTray();
  updateScore();
  statusEl.textContent = 'Select a piece to begin.';
  renderBoard();
  renderTray();
}

rotateBtn.addEventListener('click', () => {
  if (selectedPieceIndex === null || gameOver) return;
  const piece = tray[selectedPieceIndex];
  piece.rotation = (piece.rotation + 1) % 4;
  hoverAnchor = null;
  previewCells = [];
  renderTray();
  renderBoard();
});

newGameBtn.addEventListener('click', startNewGame);

createBoardCells();
startNewGame();
