# Fitness Assessment Feature - Implementation Plan

## Overview
Complete fitness assessment capture flow with 6 metric tiles, live progress tracking, and immediate data persistence to player profiles.

---

## Flow Diagram

```
1. Select "Fitness Assessment" session type
2. Mark attendance (only present players)
   ↓
3. Six metric tiles appear (grid):
   - Beep Test | Burpees | Sit-Ups | J.Squats | Speed/Agility | Laps
   ↓
4. Tap a tile → it goes DARK (active state)
   ↓
5. Player list appears below with input fields (filtered to present players only)
   ↓
6. Type each player's score → counter updates live "2/6 done"
   ↓
7. Tap 💾 Save → tile turns GREEN with ✓ → panel closes → back to metric grid
   ↓
8. Tap next metric → repeat
   ↓
9. Header counter updates: "3/6" as metrics completed
   ↓
10. Each metric saves:
    - localStorage (for resilience)
    - Player's fitness profile (immediate backend sync)
    ↓
11. Session complete → "Player of Session" selector at bottom
    ↓
12. Tap "Save Session" → full reset, data locked in history

---

## Data Structure

### Current Player Object (Enhanced)
```javascript
{
  id: "legacy-002",
  num: 2,
  name: "Ethan Pietersen",
  dob: "2014-01-29",
  pos: "",
  parent: "Neil / Miss B",
  phone: "0609728226",
  notes: "",
  fitness: [
    {
      date: "2026-02-07",
      beep: 3.3,
      burpees: 12,
      situps: 13,
      jsquats: 33,
      agility: "00:37:02",  // HH:MM:SS format
      laps: "03:37",         // MM:SS format
      notes: "Feb 2026 Assessment"
    }
  ]
}
```

### Session Header (Current)
```javascript
{
  date: "2026-05-24",
  type: "Fitness Assessment",
  notes: "...",
  fitnessMetrics: {  // NEW
    beep: { done: true, startTime: ... },
    burpees: { done: false, startTime: ... },
    situps: { done: false, startTime: ... },
    jsquats: { done: false, startTime: ... },
    agility: { done: false, startTime: ... },
    laps: { done: false, startTime: ... }
  }
}
```

### Active Fitness Session (localStorage)
```javascript
// Key: "gsc2_fa" (FA = Fitness Assessment)
{
  date: "2026-05-24",
  presentPlayers: [
    { id: "legacy-002", num: 2, name: "Ethan Pietersen" },
    { id: "legacy-003", num: 3, name: "Hlumelo" }
  ],
  metrics: {
    beep: {
      done: false,
      scores: {
        "legacy-002": 3.3,
        "legacy-003": null
      }
    },
    burpees: {
      done: true,
      scores: {
        "legacy-002": 12,
        "legacy-003": 15
      }
    },
    // ... other metrics
  },
  playerOfSession: null  // {id, name} when set
}
```

---

## UI Components

### 1. Fitness Assessment Section (Replaces existing fa-section)

**HTML Structure:**
```html
<div class="fa-section" id="faSection">
  <!-- Header with overall progress -->
  <div class="fa-hdr">
    <div>
      <div class="fa-title">💪 Fitness Assessment</div>
      <div class="fa-subtitle">Select metric tiles below</div>
    </div>
    <div id="faProgress" class="fa-progress-badge">0/6</div>
  </div>

  <!-- 6 Metric Tiles Grid -->
  <div class="fa-metrics-grid">
    <div class="fa-metric-tile" data-metric="beep" onclick="activateMetric('beep')">
      <div class="fa-tile-icon">🔊</div>
      <div class="fa-tile-label">Beep Test</div>
      <div class="fa-tile-status"></div>
    </div>
    <!-- ... repeat for burpees, situps, jsquats, agility, laps -->
  </div>

  <!-- Active Metric Panel (hidden initially) -->
  <div class="fa-active-panel" id="faActivePanel" style="display:none">
    <div class="fa-panel-hdr">
      <div class="fa-metric-title" id="faPanelTitle">Beep Test</div>
      <div class="fa-panel-progress" id="faPanelProgress">0/6</div>
    </div>
    <div class="fa-player-scores" id="faPlayerScores"></div>
    <div class="fa-panel-actions">
      <button class="fa-save-btn" onclick="saveMetricScores()">💾 Save</button>
      <button class="fa-cancel-btn" onclick="cancelMetric()">✕ Cancel</button>
    </div>
  </div>
</div>
```

### 2. Metric Tile States

**CSS:**
```css
.fa-metric-tile {
  /* Default: light, clickable */
  background: var(--grey);
  border: 2px solid var(--border);
  cursor: pointer;
  transition: all 0.2s;
}

.fa-metric-tile.active {
  /* When selected: dark blue */
  background: var(--navy);
  color: #fff;
  border-color: var(--navy);
}

.fa-metric-tile.done {
  /* When completed: green with checkmark */
  background: var(--green);
  border-color: var(--green);
  color: #fff;
}

.fa-metric-tile.done::after {
  content: "✓";
}
```

### 3. Player Score Input (per Metric)

```html
<div class="fa-score-row">
  <div class="fa-score-player">
    <span class="fa-pnum">2</span>
    <span class="fa-pname">Ethan Pietersen</span>
  </div>
  <input
    type="number"
    class="fa-score-input"
    data-player-id="legacy-002"
    placeholder="Score"
    oninput="updateMetricProgress()"
  />
</div>
```

---

## JavaScript Implementation

### 1. Initialize Fitness Assessment

```javascript
// Constants
const FA_METRICS = ['beep', 'burpees', 'situps', 'jsquats', 'agility', 'laps'];
const FA_KEY = 'gsc2_fa'; // localStorage key for active fitness session

// Init on session type change
function initFitnessAssessment() {
  const type = document.getElementById('sType').value;
  if (type !== 'Fitness Assessment') return;

  const presentIds = Object.keys(att).filter(id => att[id] === 'x');
  const presentPlayers = players.filter(p => presentIds.includes(p.id));

  const faSession = {
    date: document.getElementById('sDate').value,
    presentPlayers: presentPlayers.map(p => ({id: p.id, num: p.num, name: p.name})),
    metrics: {}
  };

  // Initialize each metric
  FA_METRICS.forEach(metric => {
    faSession.metrics[metric] = {
      done: false,
      scores: {}
    };
    presentPlayers.forEach(p => {
      faSession.metrics[metric].scores[p.id] = null;
    });
  });

  faSession.playerOfSession = null;
  save(FA_KEY, faSession);
  renderFitnessMetrics();
}

// Render 6-tile grid
function renderFitnessMetrics() {
  const container = document.getElementById('faMetricsGrid');
  container.innerHTML = '';

  FA_METRICS.forEach(metric => {
    const tile = document.createElement('div');
    tile.className = 'fa-metric-tile';
    tile.dataset.metric = metric;
    tile.onclick = () => activateMetric(metric);

    const icons = {beep: '🔊', burpees: '⬆️', situps: '📤', jsquats: '🦵', agility: '⚡', laps: '🏃'};
    const labels = {beep: 'Beep Test', burpees: 'Burpees', situps: 'Sit-Ups', jsquats: 'Jump Squats', agility: 'Speed/Agility', laps: 'Laps'};

    const fa = load(FA_KEY, {});
    if (fa.metrics && fa.metrics[metric]?.done) {
      tile.classList.add('done');
    }

    tile.innerHTML = `
      <div class="fa-tile-icon">${icons[metric]}</div>
      <div class="fa-tile-label">${labels[metric]}</div>
      <div class="fa-tile-status">${fa.metrics?.[metric]?.done ? '✓' : ''}</div>
    `;

    container.appendChild(tile);
  });

  updateFitnessProgress();
}

// Activate metric panel
let currentMetric = null;

function activateMetric(metric) {
  currentMetric = metric;
  const fa = load(FA_KEY, {});

  // Show panel
  const panel = document.getElementById('faActivePanel');
  panel.style.display = 'block';

  const labels = {beep: 'Beep Test', burpees: 'Burpees', situps: 'Sit-Ups', jsquats: 'Jump Squats', agility: 'Speed/Agility', laps: 'Laps'};
  document.getElementById('faPanelTitle').textContent = labels[metric];

  // Render player inputs
  const playerScoresDiv = document.getElementById('faPlayerScores');
  playerScoresDiv.innerHTML = '';

  fa.presentPlayers?.forEach(player => {
    const currentScore = fa.metrics?.[metric]?.scores?.[player.id] ?? null;
    const row = document.createElement('div');
    row.className = 'fa-score-row';
    row.innerHTML = `
      <div class="fa-score-player">
        <span class="fa-pnum">${player.num}</span>
        <span class="fa-pname">${player.name}</span>
      </div>
      <input
        type="number"
        class="fa-score-input"
        data-player-id="${player.id}"
        value="${currentScore ?? ''}"
        placeholder="0"
        step="0.1"
        oninput="updateMetricProgress()"
      />
    `;
    playerScoresDiv.appendChild(row);
  });

  updateMetricProgress();
}

// Update progress counter for current metric
function updateMetricProgress() {
  const fa = load(FA_KEY, {});
  const inputs = document.querySelectorAll('.fa-score-input');
  const filled = Array.from(inputs).filter(i => i.value.trim() !== '').length;
  const total = inputs.length;

  document.getElementById('faPanelProgress').textContent = `${filled}/${total}`;
}

// Save metric scores to localStorage & player profiles
function saveMetricScores() {
  const fa = load(FA_KEY, {});
  const inputs = document.querySelectorAll('.fa-score-input');

  // Collect scores
  inputs.forEach(input => {
    const playerId = input.dataset.playerId;
    const score = input.value ? parseFloat(input.value) : null;
    fa.metrics[currentMetric].scores[playerId] = score;
  });

  fa.metrics[currentMetric].done = true;
  save(FA_KEY, fa);

  // Save to player profiles immediately
  fa.presentPlayers?.forEach(player => {
    const playerObj = players.find(p => p.id === player.id);
    if (playerObj) {
      // Find or create today's fitness entry
      const today = document.getElementById('sDate').value;
      let entry = playerObj.fitness.find(f => f.date === today);
      if (!entry) {
        entry = {
          date: today,
          beep: null, burpees: null, situps: null, jsquats: null, agility: null, laps: null,
          notes: 'Fitness Assessment'
        };
        playerObj.fitness.push(entry);
      }

      // Update metric
      entry[currentMetric] = fa.metrics[currentMetric].scores[player.id];
    }
  });

  save(K.PL, players);

  // Update UI
  document.getElementById('faActivePanel').style.display = 'none';
  currentMetric = null;
  renderFitnessMetrics();
  showToast(`✓ ${currentMetric} scores saved`);
}

// Update overall progress in header
function updateFitnessProgress() {
  const fa = load(FA_KEY, {});
  const completed = FA_METRICS.filter(m => fa.metrics?.[m]?.done).length;
  document.getElementById('faProgress').textContent = `${completed}/6`;
}

function cancelMetric() {
  document.getElementById('faActivePanel').style.display = 'none';
  currentMetric = null;
}
```

### 4. Time Input Handler (for Agility/Laps)

```javascript
// For MM:SS format fields (agility, laps)
function initTimeInputs() {
  const timeFields = document.querySelectorAll('[data-metric="agility"], [data-metric="laps"]');
  timeFields.forEach(field => {
    field.type = 'text'; // Use text to accept MM:SS format
    field.placeholder = 'MM:SS';
    field.oninput = (e) => validateTimeFormat(e.target);
  });
}

function validateTimeFormat(input) {
  let val = input.value;
  // Remove non-digits
  val = val.replace(/\D/g, '');
  // Format as MM:SS
  if (val.length > 4) val = val.slice(0, 4);
  if (val.length > 2) val = val.slice(0, 2) + ':' + val.slice(2);
  input.value = val;
}
```

### 5. Session Completion

```javascript
// After all 6 metrics are done, show Player of Session selector
function checkFitnessComplete() {
  const fa = load(FA_KEY, {});
  const allDone = FA_METRICS.every(m => fa.metrics?.[m]?.done);

  if (allDone) {
    document.getElementById('motmSection').style.display = 'block';
    // Highlight the button
    showToast('🎉 All metrics complete! Select Player of Session');
  }
}

// On final "Save Session", lock data and reset
function saveFitnessSession() {
  const fa = load(FA_KEY, {});

  // Verify all metrics done
  if (!FA_METRICS.every(m => fa.metrics?.[m]?.done)) {
    showToast('⚠️ Complete all metrics first');
    return;
  }

  // Create history entry
  const historyEntry = {
    id: `${Date.now()}-FA`,
    date: fa.date,
    type: 'Fitness Assessment',
    opp: '',
    present: fa.presentPlayers,
    absent: [],
    late: [],
    injured: [],
    motm: motm || null,
    fitness: fa.metrics, // Store all metric scores
    tags: tags,
    notes: document.getElementById('attNotes').value,
    timestamp: new Date().toISOString()
  };

  history.push(historyEntry);
  save(K.HI, history);

  // Clear session
  localStorage.removeItem(FA_KEY);
  att = {};
  scorers = [];
  motm = null;
  tags = [];

  showToast('✓ Fitness Assessment saved!');
  showHome();
}
```

---

## CSS Additions

```css
/* Fitness Assessment Grid */
.fa-metrics-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  padding: 10px 14px;
  background: var(--grey);
}

.fa-metric-tile {
  background: #fff;
  border: 2px solid var(--border);
  border-radius: 10px;
  padding: 14px 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  transition: all 0.2s;
  text-align: center;
  min-height: 90px;
  justify-content: center;
}

.fa-metric-tile:active {
  transform: scale(0.97);
}

.fa-metric-tile.active {
  background: var(--navy);
  border-color: var(--navy);
  color: #fff;
}

.fa-metric-tile.done {
  background: var(--green);
  border-color: var(--green);
  color: #fff;
}

.fa-tile-icon {
  font-size: 28px;
}

.fa-tile-label {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.fa-tile-status {
  font-size: 16px;
  min-height: 16px;
}

/* Active Panel */
.fa-active-panel {
  background: #fff;
  border-top: 1px solid var(--border);
  padding: 12px 14px;
}

.fa-panel-hdr {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 2px solid var(--border);
}

.fa-metric-title {
  font-family: 'Bebas Neue', sans-serif;
  font-size: 16px;
  color: var(--navy);
  letter-spacing: 1px;
}

.fa-panel-progress {
  font-family: 'Bebas Neue', sans-serif;
  font-size: 16px;
  color: var(--red);
}

.fa-player-scores {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 12px;
  max-height: 300px;
  overflow-y: auto;
}

.fa-score-row {
  display: flex;
  align-items: center;
  gap: 8px;
  background: var(--grey);
  padding: 10px;
  border-radius: 8px;
}

.fa-score-player {
  display: flex;
  align-items: center;
  gap: 6px;
  flex: 1;
}

.fa-pnum {
  font-family: 'Bebas Neue', sans-serif;
  font-size: 14px;
  color: var(--navy);
  min-width: 18px;
}

.fa-pname {
  font-size: 12px;
  font-weight: 600;
  color: var(--dgrey);
}

.fa-score-input {
  width: 60px;
  padding: 7px 8px;
  border: 1.5px solid var(--border);
  border-radius: 6px;
  font-size: 13px;
  font-weight: 700;
  text-align: center;
  background: #fff;
  outline: none;
}

.fa-score-input:focus {
  border-color: var(--navy);
  background: #fff;
}

.fa-panel-actions {
  display: flex;
  gap: 8px;
}

.fa-save-btn, .fa-cancel-btn {
  flex: 1;
  padding: 10px;
  border: none;
  border-radius: 8px;
  font-family: 'DM Sans', sans-serif;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
}

.fa-save-btn {
  background: var(--green);
  color: #fff;
}

.fa-cancel-btn {
  background: #FEE2E2;
  color: #DC2626;
}
```

---

## Integration Checklist

- [ ] Add `FA_KEY` constant at top of script
- [ ] Add `currentMetric` state variable
- [ ] Call `initFitnessAssessment()` when fitness type selected
- [ ] Render `renderFitnessMetrics()` on type change
- [ ] Wire up tile click → `activateMetric()`
- [ ] Implement score input → `updateMetricProgress()`
- [ ] Save button → `saveMetricScores()`
- [ ] Update player profiles immediately
- [ ] Update header progress counter live
- [ ] Show Player of Session selector on completion
- [ ] Final save → lock history entry

---

## Data Flow Summary

1. **Attendance** → Present players identified
2. **Fitness Assessment selected** → 6 tiles shown, FA session initialized
3. **Tile clicked** → Player inputs shown, localStorage ready
4. **Scores entered** → Live counter updates
5. **Save → Scores saved to localStorage + player.fitness array**
6. **All 6 done** → Player of Session selector active
7. **Final Save Session** → History entry created, app reset

✅ **Dual persistence**: localStorage (offline resilience) + player profiles (permanent)
