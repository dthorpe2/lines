# LINES 1.1 — System Specification
## Logographic Integrated Notation Expression System

---

## Plain Language Rules

### 1. The Grid

A 5×5 grid of 25 cells holds the alphabet. Letters are arranged in reading order, left to right, top to bottom:

```
A  B  C  D  E
F  G  H  I  J
K  L  M  N  O
P  R  S  T  U
V  W  X  Y  Z
```

Q is absent. Q and X share the same cell (row 4, column 2). When encoding a word containing Q, use X's position.

Each cell has a fixed coordinate: row 0–4 (top to bottom), column 0–4 (left to right). A = (0,0). Z = (4,4). X/Q = (4,2).

---

### 2. Encoding a Word

Each word becomes a single connected path across the grid.

**Step 1 — Map letters to nodes**
Convert each letter to its grid coordinate. Consecutive duplicate letters (e.g. the two L's in "BELL") collapse to one node — they occupy the same position.

**Step 2 — Draw the path**
Connect each node in sequence. The path traces through every letter's position in order.

**Step 3 — Mark the start**
Place a filled dot at the first letter's node.

**Step 4 — Mark the end**
Place a filled arrowhead over the final letter's node. The arrowhead points in the direction of the incoming path. Its base midpoint — not its tip — sits exactly on the node. The tip extends beyond the node.

**Step 5 — Mark double letters**
If any letter repeats consecutively, place a small open circle (white dot with outline) directly on that letter's node. This visually distinguishes "TO" from "TOO", "OF" from "OFF", etc.

---

### 3. Reading a Symbol

1. Locate the filled dot — that is the first letter.
2. Follow the line through each node in order.
3. The arrowhead's base midpoint marks the last letter — the tip extends slightly beyond it.
4. Any white dot on the path indicates a doubled letter at that node.
5. Look up each node's grid coordinate to recover the letter sequence.

---

### 4. Writing a Sentence

Words are placed in a horizontal row, left to right, with consistent spacing between each symbol. Punctuation marks are inserted inline between symbols, rendered in standard type at the same height as the symbols. No spaces are encoded — word boundaries are implied by the discrete symbols.

---

### 5. Disambiguation Rules

- **Consecutive duplicate letters** collapse to one node AND receive a white dot marker. "LETTER" has two T's: the T node appears once in the path, with a white dot on it.
- **Q always uses X's node.** Context disambiguates in practice.
- **Single-letter words** are represented by a filled dot alone at that letter's node, with no line or arrowhead.
- **Two-letter words where both letters share the same node** (e.g. "XX") are treated as a single-node word with a white dot.

---

### 6. Proportionality Rules

All visual elements scale proportionally with the cell size (CELL = grid drawing area ÷ 5):

All values are relative to SIZE (the canvas pixel width). CELL = (SIZE − 2×PAD) ÷ 5, where PAD = SIZE × 0.06.

| Element | Formula | Notes |
|---|---|---|
| Line weight (LW) | SIZE × 0.05 | Scales with canvas |
| Arrowhead length (AL) | SIZE × 0.18 | Tip-to-base distance |
| Start dot radius | SIZE × 0.09 | Filled circle at first node |
| White dot radius | SIZE × 0.08 | Open circle at doubled node |
| White dot stroke | LW × 0.4 | Outline of white dot |

The arrowhead base midpoint sits exactly on the final node. Tip extends AL/2 past the node; the line terminates AL × 0.309 before the node. The constant 0.309 = cos(36°) − 0.5, derived from the arrowhead half-angle of 36°.

---

### 7. What LINES Is Not

- It is not a cipher. The encoding rule is public and self-evident from the grid.
- It is not a phonetic system. Letters map to positions; pronunciation is not encoded.
- It is not a lookup system. Any word can be encoded from the grid alone with no dictionary.

---

---

## Specification as Commented Code

```javascript
// ─────────────────────────────────────────────────────────────────────────────
// LINES 1.1 — Logographic Integrated Notation Expression System
// Core encoding and rendering specification
// ─────────────────────────────────────────────────────────────────────────────


// ── 1. THE GRID ───────────────────────────────────────────────────────────────

// 25 cells, A–Z, arranged in reading order (left to right, top to bottom).
// Row index 0 = top row (A B C D E).
// Column index 0 = leftmost column.
// Q is absent — it shares X's position at row 4, column 2.

const LAYOUT = [
  ['A','B','C','D','E'],   // row 0
  ['F','G','H','I','J'],   // row 1
  ['K','L','M','N','O'],   // row 2
  ['P','R','S','T','U'],   // row 3  — note: no Q
  ['V','W','X','Y','Z'],   // row 4
];

// Build a position lookup: letter → {row, col}
const POS = {};
for (let r = 0; r < 5; r++)
  for (let c = 0; c < 5; c++)
    POS[LAYOUT[r][c]] = { r, c };

// Q shares X's position
POS['Q'] = POS['X'];


// ── 2. ENCODING A WORD TO A NODE LIST ────────────────────────────────────────

// A word is encoded as an ordered list of grid nodes.
// Rules:
//   - Each letter maps to its grid position.
//   - Consecutive duplicate letters collapse: only one node is added.
//   - We also track which nodes carry a "double letter" marker.

function encodeWord(word) {
  const letters = word.toUpperCase().replace(/[^A-Z]/g, '');

  const nodes = [];        // collapsed list of [row, col] pairs
  const letterToNode = []; // maps letter index → node index

  for (const ch of letters) {
    const pos = POS[ch];
    if (!pos) { letterToNode.push(-1); continue; }

    const node = [pos.r, pos.c];

    // Collapse consecutive duplicates: if this letter's node is the same
    // as the last node added, don't add a new one.
    const lastNode = nodes[nodes.length - 1];
    const isNewNode = !lastNode
      || lastNode[0] !== node[0]
      || lastNode[1] !== node[1];

    if (isNewNode) nodes.push(node);
    letterToNode.push(nodes.length - 1);
  }

  // Find which nodes carry a double-letter marker.
  // A node is marked if its letter appears consecutively in the source word.
  const doubleNodes = new Set();
  for (let i = 0; i < letters.length - 1; i++) {
    if (letters[i] === letters[i + 1] && letterToNode[i] >= 0) {
      doubleNodes.add(letterToNode[i]);
    }
  }

  return { nodes, doubleNodes };
}


// ── 3. COORDINATE HELPERS ────────────────────────────────────────────────────

// Given the drawing origin (ox, oy) and cell size, return the pixel centre
// of a grid node. Canvas y-axis points DOWN, so row 0 is at the top.
function nodeCentre(row, col, ox, oy, cell) {
  return {
    x: ox + col * cell + cell / 2,
    y: oy + row * cell + cell / 2,
  };
}


// ── 4. RENDERING ─────────────────────────────────────────────────────────────

// All proportions are derived from SIZE (canvas pixel width) and CELL (SIZE/5).
// This ensures the symbol scales correctly at any resolution.

function drawWord(canvas, word, options = {}) {
  const SIZE = canvas.width / (window.devicePixelRatio || 1);
  const dpr  = window.devicePixelRatio || 1;
  const ctx  = canvas.getContext('2d');
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  ctx.clearRect(0, 0, SIZE, SIZE);

  // Layout: small padding so strokes don't clip at the canvas edge
  const PAD  = SIZE * 0.06;
  const DRAW = SIZE - PAD * 2;  // usable drawing area
  const CELL = DRAW / 5;        // one grid cell
  const ox   = PAD;
  const oy   = PAD;

  // ── Optional grid rendering ──────────────────────────────────────────────
  // (shown in early learning levels; hidden in later levels)
  if (options.showGrid) {
    ctx.strokeStyle = '#e0e0e0';
    ctx.lineWidth   = SIZE * 0.008;
    for (let i = 1; i < 5; i++) {
      // vertical lines
      ctx.beginPath();
      ctx.moveTo(ox + i * CELL, oy);
      ctx.lineTo(ox + i * CELL, oy + DRAW);
      ctx.stroke();
      // horizontal lines
      ctx.beginPath();
      ctx.moveTo(ox,         oy + i * CELL);
      ctx.lineTo(ox + DRAW,  oy + i * CELL);
      ctx.stroke();
    }
  }

  // ── Optional letter labels ───────────────────────────────────────────────
  // (shown in level 1; progressively removed in later levels)
  if (options.showLetters) {
    ctx.textAlign    = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillStyle    = '#cccccc';
    for (let r = 0; r < 5; r++) {
      for (let c = 0; c < 5; c++) {
        // X/Q share a cell — label shows both
        const label = (r === 4 && c === 2) ? 'X/Q' : LAYOUT[r][c];
        ctx.font = `${CELL * 0.46}px monospace`;
        const { x, y } = nodeCentre(r, c, ox, oy, CELL);
        ctx.fillText(label, x, y);
      }
    }
  }

  // ── Encode the word ──────────────────────────────────────────────────────
  const { nodes, doubleNodes } = encodeWord(word);
  if (!nodes.length) return;

  // Convert node list to pixel coordinates
  const pts = nodes.map(([r, c]) => nodeCentre(r, c, ox, oy, CELL));

  // ── Proportional sizing ──────────────────────────────────────────────────
  const LW     = SIZE * 0.05;   // line weight
  const AL     = SIZE * 0.18;   // arrowhead length (tip-to-base)
  const DOT_R  = SIZE * 0.09;   // start dot radius
  const WD_R   = SIZE * 0.08;   // white (double-letter) dot radius
  const AA     = Math.PI / 5;   // arrowhead half-angle (36°)

  ctx.strokeStyle = '#000000';
  ctx.fillStyle   = '#000000';
  ctx.lineWidth   = LW;
  ctx.lineCap     = 'butt';     // butt cap: line ends exactly at the endpoint
  ctx.lineJoin    = 'round';

  // ── Single-node words (one letter, or all letters at same position) ──────
  // Represented by a filled dot alone — no line, no arrowhead.
  if (pts.length === 1) {
    ctx.beginPath();
    ctx.arc(pts[0].x, pts[0].y, DOT_R, 0, Math.PI * 2);
    ctx.fill();
    return;
  }

  // ── Path line ────────────────────────────────────────────────────────────
  // The line runs from the first node through all intermediate nodes,
  // then stops where the arrowhead base begins.
  //
  // Arrowhead geometry:
  //   - The arrowhead is CENTRED on the final node.
  //   - Tip is at: last + (AL/2) in the direction of travel.
  //   - Base midpoint is at: last − (AL/2) in the direction of travel.
  //   - The two base corners are at: tip − AL*(cos(ang±AA), sin(ang±AA))
  //   - The visible base of the triangle sits at: last − cos(AA)*(AL/2 ... )
  //   - Line terminates at: last − (AL × 0.309) so it meets the arrowhead base
  //     without gap or overlap.
  //     Derivation: base_midpoint = last − AL*(cos(AA) − 0.5) = last − AL*0.309
  //     where cos(36°) = 0.809, so cos(36°) − 0.5 = 0.309.

  const last = pts[pts.length - 1];
  const prev = pts[pts.length - 2];
  const ang  = Math.atan2(last.y - prev.y, last.x - prev.x);

  // Line endpoint: pulled back to meet the arrowhead base exactly
  const lineEnd = {
    x: last.x - AL * 0.309 * Math.cos(ang),
    y: last.y - AL * 0.309 * Math.sin(ang),
  };

  ctx.beginPath();
  ctx.moveTo(pts[0].x, pts[0].y);
  for (let i = 1; i < pts.length - 1; i++) {
    ctx.lineTo(pts[i].x, pts[i].y);
  }
  ctx.lineTo(lineEnd.x, lineEnd.y);
  ctx.stroke();

  // ── Arrowhead ────────────────────────────────────────────────────────────
  // Filled triangle, centred on the final node.
  // Tip extends AL/2 past the node in the direction of travel.
  const tipX = last.x + AL * 0.5 * Math.cos(ang);
  const tipY = last.y + AL * 0.5 * Math.sin(ang);

  ctx.beginPath();
  ctx.moveTo(tipX, tipY);
  ctx.lineTo(tipX - AL * Math.cos(ang - AA), tipY - AL * Math.sin(ang - AA));
  ctx.lineTo(tipX - AL * Math.cos(ang + AA), tipY - AL * Math.sin(ang + AA));
  ctx.closePath();
  ctx.fill();

  // ── Start dot ────────────────────────────────────────────────────────────
  // Filled circle centred on the first letter's node.
  ctx.beginPath();
  ctx.arc(pts[0].x, pts[0].y, DOT_R, 0, Math.PI * 2);
  ctx.fill();

  // ── Double-letter markers ────────────────────────────────────────────────
  // A white open circle placed directly on the node of any doubled letter.
  // This distinguishes e.g. "TO" (no marker) from "TOO" (white dot on T/O node).
  for (const nodeIndex of doubleNodes) {
    const pt = pts[nodeIndex];
    ctx.save();
    ctx.fillStyle   = '#ffffff';        // white fill
    ctx.strokeStyle = '#000000';        // black outline
    ctx.lineWidth   = LW * 0.4;
    ctx.beginPath();
    ctx.arc(pt.x, pt.y, WD_R, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
    ctx.restore();
  }
}


// ── 5. ENCODING AS A BIT STREAM ──────────────────────────────────────────────

// For data storage or transmission, each word is encoded as a sequence of
// 6-bit node descriptors: 3 bits for row (0–4), 3 bits for column (0–4).
// No header, no length prefix — the word boundary is implicit in the document
// structure. A single word averages ~37 bits vs ~46 bits for UTF-8.

function wordToBits(word) {
  const { nodes } = encodeWord(word);
  let bits = '';
  for (const [r, c] of nodes) {
    bits += r.toString(2).padStart(3, '0');  // 3 bits: row
    bits += c.toString(2).padStart(3, '0');  // 3 bits: column
  }
  return bits;  // length = nodes.length × 6 bits
}

function bitsToNodes(bits) {
  const nodes = [];
  for (let i = 0; i + 5 < bits.length; i += 6) {
    const r = parseInt(bits.slice(i,     i + 3), 2);
    const c = parseInt(bits.slice(i + 3, i + 6), 2);
    nodes.push([r, c]);
  }
  return nodes;
}

// To decode a bit stream back to letters:
// Each node [r, c] → LAYOUT[r][c]
// Consecutive identical nodes → single letter (was a double)
// Any node that appears twice consecutively in the ORIGINAL word
// is flagged by its white dot marker in the visual form.
function nodesToLetters(nodes) {
  return nodes.map(([r, c]) => LAYOUT[r][c]).join('');
}
```
