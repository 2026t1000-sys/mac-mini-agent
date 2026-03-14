# Multi-Agent Game Pipeline Design
## For mac-mini-agent system — Complex Creative Tasks

**Version:** 1.0
**Created:** 2026-03-14
**Scope:** Research + concrete implementation recommendations for high-quality game creation from user prompts
**Status:** Design document — ready for implementation

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Decomposition Strategies](#2-decomposition-strategies)
3. [The Optimal 4-Phase Pipeline](#3-the-optimal-4-phase-pipeline)
4. [Quality Gates](#4-quality-gates)
5. [Context Passing Between Agents](#5-context-passing-between-agents)
6. [Game Validation Agent Design](#6-game-validation-agent-design)
7. [Timeout Prevention Strategies](#7-timeout-prevention-strategies)
8. [Industry Reference: Studio Pipelines](#8-industry-reference-studio-pipelines)
9. [Implementation Recommendations](#9-implementation-recommendations)
10. [Integration with Existing Code](#10-integration-with-existing-code)

---

## 1. Problem Statement

### Root Cause of Current Failures

The system has a `pipeline.py` running a sequential 4-stage flow:
`architect (300s) → builder (600s) → reviewer (180s) → validator (120s)`

For a HoMM3-style game, the builder stage alone needs ~4000+ lines of code.
A single Claude agent in a 600-second window cannot produce:
- 21 programmatic sprite generators
- Hex grid + pathfinding + BFS
- Full combat system (12 special abilities, morale, counters)
- AI decision tree with target scoring
- 8 spells with scaling
- Campaign mode (5 battles, rewards, persistence)
- Procedural audio (9 sound effects via Web Audio API)
- Mobile touch controls + resize handling

**Result:** Either timeout (1800s hit) or massive code omissions and colored-rectangle fallbacks.

### What "Too Big for One Agent" Looks Like

An agent task is too big when it:
- Requires writing more than ~600 lines of interdependent code
- Has more than 5 distinct subsystems that each have their own internal logic
- Cannot produce a testable deliverable in 20-40 minutes
- Requires making sequential design decisions that block all other work

For the HoMM3 clone specifically: the GDD defines 21 units, 8 spells, 3 factions, 5 campaign
battles, a morale system, 11 special abilities, and real programmatic sprites. This is
approximately 5000-6000 lines of working JavaScript — at least 10x what a single agent
reliably delivers in one shot.

---

## 2. Decomposition Strategies

### 2.1 What Can Run in Parallel

The key insight is to identify which subsystems have NO shared mutable state during creation.

```
PARALLEL-SAFE SUBSYSTEMS:

  Data Layer (pure definitions — no runtime state)
  ├─ Unit stats tables (attack, defense, hp, speed, initiative)
  ├─ Spell definitions (cost, effect formula, scaling)
  ├─ Faction rosters (which units belong to which faction)
  └─ Game constants (grid size, timing values, modifiers)

  Visual Layer (pure drawing functions — no game state needed)
  ├─ Programmatic sprite generators (drawPikeman, drawGriffin, etc.)
  ├─ Hex grid renderer (drawHex, hexToPixel coordinate math)
  ├─ Terrain texture generator (Perlin noise or pattern fill)
  └─ Animation system (move, lunge, flash, float number)

  Logic Layer (pure functions — takes state in, returns new state)
  ├─ Combat resolution (calculateDamage, applyDamage, handleCounter)
  ├─ Pathfinding (BFS for reachable hexes, movement validation)
  ├─ Special ability handlers (jousting, life drain, breath attack)
  └─ Spell effect system (apply buff/debuff, AoE damage)

  AI Layer (strategy functions — reads state, decides action)
  ├─ Target scoring formula
  ├─ Movement decision (best hex to advance toward)
  ├─ Spell casting decision tree
  └─ Wait/defend logic

  UI Layer (event handlers and DOM updates)
  ├─ Touch event handling (tap, long press, swipe)
  ├─ Button handlers (spell buttons, wait, defend)
  ├─ Initiative bar renderer
  └─ Unit info panel updater
```

**Why these are safe to parallelize:** Each subsystem is a collection of pure functions or
constant definitions. They share NO mutable variables during creation. Agent 4 writing
`drawGriffin()` doesn't need to know the contents of `calculateDamage()`. They only need
to agree on shared data contracts (unit object shape, hex coordinate format).

### 2.2 What Must Be Sequential

Sequential ordering is enforced by real data dependencies:

```
1. Constants & Data Structures  (defines the shared types all others use)
         ↓
2. Parallel zone: Visual + Logic + AI + UI  (each uses types from step 1)
         ↓
3. Integration  (assembles all parallel outputs into one file)
         ↓
4. Validation  (tests the assembled file)
         ↓
5. Polish/Fix  (addresses validation failures)
```

Anything that reads from a shared output must happen after that output is written.
The integration agent (step 3) MUST wait for all parallel agents in step 2 to finish
before it can assemble the final HTML.

The architecture agent (step 1) MUST finish before any parallel agents start, because it
defines the interfaces — specifically the shape of the `Unit` object, the `GameState`
object, and the coordinate system used for hex math. If the visual agent assumes
`unit.icon` is a string (emoji) and the logic agent assumes `unit.spriteIndex` is a number,
integration will fail.

### 2.3 Optimal Task Sizes for AI Agents

Based on empirical observation from this system's existing runs and the HoMM3 post-mortem:

| Task Size | Code Lines | Time Budget | Success Rate | Verdict |
|-----------|-----------|-------------|--------------|---------|
| Micro | 50-150 | 5-10 min | ~95% | Too granular — orchestration overhead not worth it |
| Small | 150-400 | 10-20 min | ~90% | Good for pure subsystems (sprite set, AI logic) |
| Medium | 400-800 | 20-40 min | ~75% | Sweet spot — one complete subsystem |
| Large | 800-1500 | 40-60 min | ~45% | Risky — consider splitting further |
| Oversized | 1500+ | 60+ min | ~15% | Always split this further |

**Target task size: 400-800 lines per agent, 20-40 minutes per task.**

The existing `orchestrator.py` uses a 30-minute per-subtask timeout (1800s). This is
appropriate for medium tasks. The pipeline.py builder stage has a 600-second (10-minute)
timeout — this is too short for any non-trivial game subsystem and should be raised to
at least 1800s for game builder agents.

### 2.4 Game-Specific Decomposition Rules

For HTML canvas games specifically:

**Split along render vs. logic, not along feature:**
- BAD: "Build the combat system including combat animations"
- GOOD: "Build combat logic functions (no rendering)" + "Build combat animations (no logic)"

**Split data from behavior:**
- BAD: "Build the unit system with all stats and drawing code"
- GOOD: "Define all unit data (UNITS constant)" + "Build unit drawing functions"

**Never split the state machine:**
- The main game state machine (which `gameState` variable belongs to, turn transitions,
  win/loss detection) must be written by ONE agent. It is the glue. Split its consumers
  (AI, UI, renderer) but never the state machine itself.

---

## 3. The Optimal 4-Phase Pipeline

This maps to the existing system's terminology (orchestrator.py phases + pipeline.py stages).

```
PHASE 1: ARCHITECTURE (sequential, 1 agent)
  ↓ outputs: GAME_SPEC.json (shared types + constants + function signatures)

PHASE 2: PARALLEL BUILD (up to 5 parallel agents)
  ├─ Agent A: Visual System (sprites + grid + terrain)
  ├─ Agent B: Combat Engine (damage + pathfinding + special abilities)
  ├─ Agent C: Game State Machine (turn system + state transitions + win/loss)
  ├─ Agent D: AI System (decision tree + target scoring + spell logic)
  └─ Agent E: UI System (HTML structure + touch handlers + panel updates)
  All read GAME_SPEC.json, write to separate JS module files

PHASE 3: INTEGRATION (sequential, 1 agent)
  Reads all Phase 2 outputs + GAME_SPEC.json
  Produces: game_draft.html (single file, all modules inlined)

PHASE 4: VALIDATION + POLISH (sequential, up to 2 agents)
  ├─ Validator: runs game_validator.py + manual checks → VALIDATION_REPORT.json
  └─ Polish: reads VALIDATION_REPORT.json + game_draft.html → game_final.html
```

### Phase 1: Architecture Agent

**Inputs:**
- User prompt (raw natural language)
- HEROES_GDD.md (if available — game design reference)
- Existing sprite library reference (`games/sprite-reference.txt`)

**Outputs:** A file named `GAME_SPEC.json` written to `apps/listen/jobs/{JOB_ID}-spec.json`

The spec must contain:

```json
{
  "game_type": "turn-based-tactical",
  "canvas_model": "canvas2d",
  "coordinate_system": {
    "type": "hex-odd-r",
    "cols": 15,
    "rows": 11,
    "hex_size_formula": "auto-scale to fit screen with 8px padding"
  },
  "shared_types": {
    "Unit": {
      "fields": ["id", "name", "side", "attack", "defense", "minDmg", "maxDmg",
                 "hp", "maxHp", "speed", "initiative", "type", "count", "maxCount",
                 "col", "row", "effects", "waited", "defended", "counterUsed"],
      "type_notes": "type is 'melee'|'ranged'|'flying'. effects is array of {type, duration}"
    },
    "Hero": {
      "fields": ["name", "attack", "defense", "spellPower", "knowledge", "mana", "maxMana", "level"]
    },
    "GameState": {
      "fields": ["phase", "units", "heroes", "turnOrder", "turnIndex", "round",
                 "selectedUnit", "reachableHexes", "attackableUnits", "spellMode",
                 "pendingSpell", "winner", "messages"]
    },
    "HexCoord": {
      "fields": ["col", "row"],
      "pixel_formula": "provided in spec — axial math for odd-r offset"
    }
  },
  "constants": {
    "COLS": 15,
    "ROWS": 11,
    "ATK_DEF_MODIFIER": 0.05,
    "COUNTER_DAMAGE_MULT": 0.5,
    "...": "all constants from GDD section 16"
  },
  "function_signatures": {
    "hexToPixel": "(col, row, hexSize) => {x, y}",
    "pixelToHex": "(px, py, hexSize) => {col, row}",
    "hexDistance": "(a, b) => number",
    "getReachableHexes": "(unit, gs) => [{col, row}]",
    "calculateDamage": "(attacker, defender, gs) => number",
    "applyDamage": "(unit, damage) => {killed, remainingHp}",
    "checkWin": "(gs) => 'player'|'enemy'|null",
    "buildTurnOrder": "(units) => Unit[]",
    "doAITurn": "(gs) => GameState"
  },
  "module_assignments": {
    "visual.js": ["hexToPixel", "pixelToHex", "drawHex", "drawUnit", "drawAllSprites", "animateMove"],
    "combat.js": ["calculateDamage", "applyDamage", "getReachableHexes", "hexDistance", "handleCounter"],
    "state.js": ["buildTurnOrder", "nextTurn", "checkWin", "applySpell", "startBattle"],
    "ai.js": ["doAITurn", "scoreTarget", "findBestMove"],
    "ui.js": ["updateInitBar", "updateUnitPanel", "onHexTap", "onSpellButton", "renderMessages"]
  },
  "units": "[ full unit data array from GDD ]",
  "spells": "[ full spell data array from GDD ]",
  "required_features": ["pegasus unit", "programmatic sprites", "hex grid", "initiative bar",
                        "hero stats display", "all 8 spells", "AI opponent", "win/loss detection"]
}
```

**Timeout:** 300 seconds (5 minutes — this is a planning task, no code writing)

**Success Criteria:**
- GAME_SPEC.json is valid JSON parseable without error
- `shared_types` section defines all fields used by all modules
- `function_signatures` covers the complete cross-module interface
- `module_assignments` is complete — every required function assigned to exactly one module
- `required_features` explicitly lists all user-requested features (prevents spec drift)
- `constants` section has every numeric constant (prevents agents inventing different values)

**Quality Gate (FAIL conditions):**
- Any function called across module boundaries is not listed in `function_signatures`
- Any field referenced in `shared_types` is missing a description
- `module_assignments` has functions assigned to multiple modules (conflict)
- File size < 500 bytes (empty/stub spec)

### Phase 2: Parallel Build Agents

All 5 agents start simultaneously after Phase 1 writes GAME_SPEC.json.
Each agent is spawned via the existing `run_agent_visible()` in `pipeline.py` with
its own tmux session.

#### Agent A: Visual System (`visual.js`)

**Reads:** `GAME_SPEC.json`

**Task prompt structure:**
```
You are a game visual agent. Build the rendering subsystem for a HoMM3-style HTML canvas game.

Reference spec: [GAME_SPEC.json contents]

Write a JavaScript module (visual.js) that implements:
1. hexToPixel(col, row, hexSize) → {x, y}  — convert hex grid coords to canvas pixels
2. pixelToHex(px, py, hexSize) → {col, row}  — inverse transform
3. drawHexGrid(ctx, gs, hexSize)  — draw all hexes with terrain colors and overlays
4. drawAllSprites(ctx, gs, hexSize)  — draw all units using their programmatic sprites
5. A drawUnit(ctx, unit, x, y, hexSize, selected) function
6. Programmatic sprite generators for all [N] units (one function per unit, 64x64 canvas)
7. animateMove(unit, fromHex, toHex, hexSize, onDone)  — 300ms ease-out movement
8. animateMeleeAttack(attacker, target, hexSize, onDone)  — lunge + return
9. animateFloatNumber(ctx, text, x, y, color)  — floating damage numbers, 800ms
10. drawBattleBackground(ctx, canvas)  — grass terrain with Perlin-style variation

Constraints:
- Output ONLY JavaScript code — no HTML, no CSS, no game logic
- Do not reference gameState or game logic functions — only draw what you are given
- Do not define Unit, Hero, or GameState — only use them as parameters
- All drawing goes through the 2d canvas context (ctx) parameter
- Use requestAnimationFrame for animations
- Every function must be callable in isolation for testing
```

**Output:** `jobs/{JOB_ID}-visual.js` — approximately 600-900 lines

**Timeout:** 1800 seconds (30 minutes)

**Success criteria (pre-integration checks):**
- `hexToPixel` function is defined
- `drawHexGrid` function is defined
- At least 10 sprite generator functions defined (one per unit type)
- No references to undefined globals (`gs`, `gameState`) without receiving them as parameters
- No game logic: no `calculateDamage`, no `applyDamage`, no turn transitions

#### Agent B: Combat Engine (`combat.js`)

**Reads:** `GAME_SPEC.json`

**Task prompt structure:**
```
You are a game combat agent. Build the combat logic subsystem for a HoMM3-style game.

Reference spec: [GAME_SPEC.json contents]

Write a JavaScript module (combat.js) that implements:
1. hexDistance(a, b) — axial distance between two hex coords
2. getNeighbors(col, row) — returns 6 adjacent hex coords
3. getReachableHexes(unit, units, obstacles) — BFS to find all hexes unit can reach
4. getAttackableUnits(attacker, units) — list of enemies unit can attack
5. calculateDamage(attacker, defender, modifiers) — full damage formula from spec
6. applyDamage(unit, rawDamage) — reduces hp, recalculates count, returns {killed, newCount, remainingHp}
7. executeAttack(attacker, defender, gs) — full attack resolution including counter-attack
8. All special ability handlers: jousting, lifeBleed, breathSplash, deathCloud, petrify,
   strikeAndReturn, griffinUnlimitedCounter, wightRegenerate, curse, moraleAura
9. applySpellEffect(spell, target, caster, gs) — apply any spell effect to target
10. checkWin(units) — return 'player'|'enemy'|null
11. buildTurnOrder(units) — sort by initiative, ties broken by player-first then tier

Constraints:
- Output ONLY JavaScript code — no rendering, no HTML, no UI
- Do not draw anything — no ctx, no canvas operations
- All functions are pure or near-pure: take state as parameters, return new state
- Use the exact damage formula from the spec (ATK_DEF_MODIFIER = 0.05, etc.)
- All special abilities must be implemented (not stubbed out)
```

**Output:** `jobs/{JOB_ID}-combat.js` — approximately 500-800 lines

**Timeout:** 1800 seconds

**Success criteria:**
- `calculateDamage` function defined
- `applyDamage` function defined
- `executeAttack` function defined
- `checkWin` function defined
- `getReachableHexes` defined
- At least 8 special ability handler functions defined
- No canvas/drawing operations (`ctx.`, `fillRect`, etc.)

#### Agent C: Game State Machine (`state.js`)

**Reads:** `GAME_SPEC.json`

**Task prompt structure:**
```
You are a game state machine agent. Build the state management and game flow for a
HoMM3-style tactical game.

Reference spec: [GAME_SPEC.json contents]

Write a JavaScript module (state.js) that implements:
1. The UNIT_DATA constant — all unit definitions with stats (from spec)
2. The SPELL_DATA constant — all spell definitions
3. initBattle(playerFaction, enemyFaction, difficulty) → GameState — creates fresh game state
4. nextTurn(gs) → GameState — advance to next unit in turn order, handle round-start effects
5. handleUnitAction(gs, action) → GameState — process: move, attack, defend, wait, castSpell
6. startEnemyTurn(gs) → GameState — flag that AI should take over
7. endBattle(gs, winner) → GameState — set winner, calculate stats
8. applySaveGame(slot) → void — save to localStorage
9. loadSaveGame() → GameState|null — load from localStorage
10. calculateMorale(unit, gs) — compute morale modifier for a unit
11. applyRoundStartEffects(gs) → GameState — wight regen, morale rolls, status duration decrement

IMPORTANT: This module owns the game state. All state changes go through these functions.
The state machine uses these phases:
- 'PLAYER_TURN' — waiting for player input
- 'ANIMATING' — playing a visual animation (no input)
- 'ENEMY_TURN' — AI is making decisions
- 'GAME_OVER' — battle ended

Constraints:
- No drawing code — state.js contains zero canvas operations
- No input handling — state.js doesn't handle click/touch events
- All functions are pure: take GameState in, return new GameState (immutable-style)
- UNIT_DATA and SPELL_DATA are defined as constants in this file
```

**Output:** `jobs/{JOB_ID}-state.js` — approximately 400-600 lines

**Timeout:** 1800 seconds

**Success criteria:**
- `initBattle` defined
- `nextTurn` defined
- `UNIT_DATA` constant defined with at least 10 unit entries
- `SPELL_DATA` constant defined with at least 4 spells
- Phase names are string literals matching the spec
- No canvas drawing code

#### Agent D: AI System (`ai.js`)

**Reads:** `GAME_SPEC.json`

**Task prompt structure:**
```
You are a game AI agent. Build the enemy AI decision-making system for a HoMM3-style game.

Reference spec: [GAME_SPEC.json contents]

Write a JavaScript module (ai.js) that implements:
1. doAITurn(gs, callbacks) → void — execute a complete AI turn for the current unit
2. scoreTarget(attacker, target, gs) — numeric score for how valuable this attack is
3. findBestMove(unit, targets, gs) — find best hex to move to before attacking
4. selectSpell(hero, gs) — decide which spell (if any) the AI hero should cast
5. selectSpellTarget(spell, gs) — select best target for a spell

The AI decision priority (per unit turn):
1. If ranged and has target in range → attack highest-scoring target
2. If can reach enemy this turn → move + attack highest-scoring reachable target
3. If cannot reach any enemy → move toward highest-value target
4. If surrounded by 2+ enemies and low HP → defend

Target scoring formula:
  score = (expectedDamage / target.currentHP) × target.tier × positionBonus
  positionBonus = 1.5 if target is ranged
  positionBonus = 2.0 if target has count == 1 (finish it off)
  positionBonus = 0.8 if target is defended

Spell casting (once per round, not per unit):
  - Low HP enemy unit exists → Magic Arrow on it
  - Own fastest unit not hasted → Haste
  - Player's highest-value unit not slowed → Slow
  - 30% random chance → Magic Arrow on any player unit

Difficulty scaling:
  - Easy: 15% spell chance, random targeting
  - Normal: 30% spell chance, optimal targeting
  - Hard: 50% spell chance, optimal targeting + Wait tactic

Constraints:
- No drawing code
- No state mutation — call the callbacks (moveUnit, attackUnit, castSpell) instead
- The callbacks parameter is an object: {moveUnit, attackUnit, castSpell, endTurn}
- AI must call endTurn() at the end of its action sequence
```

**Output:** `jobs/{JOB_ID}-ai.js` — approximately 300-500 lines

**Timeout:** 1200 seconds (20 minutes)

**Success criteria:**
- `doAITurn` defined
- `scoreTarget` defined
- `findBestMove` defined
- No canvas drawing code
- AI calls `endTurn()` — unit must act and yield

#### Agent E: UI System (`ui.js`)

**Reads:** `GAME_SPEC.json`

**Task prompt structure:**
```
You are a game UI agent. Build the user interface and input handling layer for a
HoMM3-style tactical game.

Reference spec: [GAME_SPEC.json contents]

Write a JavaScript module that implements the HTML structure and JavaScript UI handlers:
1. buildHTML() → string — the complete HTML body structure (top bar, canvas, initiative bar,
   bottom panel with unit info + spell buttons + action buttons)
2. initTouchHandlers(canvas, callbacks) — attach touch/click handlers to canvas
3. updateInitiativeBar(turnOrder, currentIndex) — redraw the initiative strip
4. updateUnitInfo(unit) — update bottom panel with unit stats
5. updateHeroInfo(hero, side) — update hero mana/stats in top bar
6. showMessage(text, duration) — floating message above battlefield
7. updateSpellButtons(hero, activeSpell) — show spell buttons, highlight active, grey out if no mana
8. setPhaseDisplay(phase) — update any phase indicator in the UI

Touch input interpretation:
- Tap on hex → onHexTap(col, row) callback
- Tap on unit → onUnitTap(unit) callback
- Tap on spell button → onSpellSelect(spellId) callback
- Tap Wait button → onWait() callback
- Tap Defend button → onDefend() callback
- Long press (300ms) on unit → onLongPress(unit) callback (show tooltip)

Mobile requirements:
- All buttons minimum 44×44px touch targets
- Canvas uses touch-action: none
- viewport meta: width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover
- No 300ms click delay (use touchstart)
- Layout works at 375px width (iPhone SE)

CSS requirements:
- Dark theme: background #1a1a2e
- Gold accents for player, red for enemy
- All CSS inlined (no external stylesheets)

Constraints:
- No game logic — ui.js does not calculate damage or decide AI actions
- All game events go through callbacks parameter — never call game functions directly
- The canvas element ID is 'game-canvas'
```

**Output:** `jobs/{JOB_ID}-ui.js` — approximately 300-500 lines

**Timeout:** 1200 seconds (20 minutes)

**Success criteria:**
- `buildHTML` defined
- `initTouchHandlers` defined
- `updateInitiativeBar` defined
- `updateHeroInfo` defined
- No game logic (no `calculateDamage` calls, no `applyDamage` calls)
- CSS includes dark theme variables

### Phase 3: Integration Agent

**Reads:**
- `GAME_SPEC.json`
- `{JOB_ID}-visual.js`
- `{JOB_ID}-combat.js`
- `{JOB_ID}-state.js`
- `{JOB_ID}-ai.js`
- `{JOB_ID}-ui.js`

**Task prompt structure:**
```
You are a game integration agent. Merge 5 JavaScript modules into a single working HTML file.

Your inputs are the following files:
[GAME_SPEC.json — paste contents]
[visual.js — paste contents]
[combat.js — paste contents]
[state.js — paste contents]
[ai.js — paste contents]
[ui.js — paste contents]

Produce a single HTML file (game.html) that:
1. Sets up the HTML structure from ui.js's buildHTML()
2. Inlines all CSS from ui.js
3. Inlines all JavaScript in this order: state.js, combat.js, visual.js, ai.js, ui.js
4. Adds a glue/init script that:
   a. Creates a canvas element with id="game-canvas"
   b. Calls buildHTML() to set up the DOM
   c. Initializes battle: const gs = initBattle('castle', 'dungeon', 'normal')
   d. Sets up the requestAnimationFrame render loop: calls drawHexGrid + drawAllSprites each frame
   e. Wires AI callbacks: when gs.phase === 'ENEMY_TURN', call doAITurn(gs, callbacks)
   f. Wires input callbacks: hex taps → handleUnitAction, spell buttons → spell mode
   g. Handles resize: recalculate hexSize on window resize
5. Ensures all cross-module calls use the exact function names from GAME_SPEC.json
6. Removes any duplicate constant definitions (e.g. if both state.js and combat.js define COLS)
7. Adds an audio system stub (AudioContext, 5 procedural sounds: sword, arrow, spell, death, fanfare)

CRITICAL requirements:
- The final file must start the game automatically on page load (no separate "start" button required
  for the battle to initialize — though a title screen is fine)
- requestAnimationFrame must be called
- doAITurn MUST be called when it is the enemy's turn
- checkWin MUST be called after each attack
- All onclick handlers must reference functions that are defined in the file
- Output ONLY the complete HTML file — no commentary before or after the code block
```

**Output:** `jobs/{JOB_ID}-game-draft.html`

**Timeout:** 2400 seconds (40 minutes)

**Success criteria (run `game_validator.py` against output):**
- validator score >= 70
- No critical issues in validator output (critical = score deduction of 15+)
- File is valid UTF-8 HTML parseable by browser
- File size > 50KB (indicates substantial content) and < 300KB

### Phase 4: Validation + Polish

**Validation Agent (automated, not Claude):**
Uses existing `game_validator.py` which checks:
- onclick handler → function definition cross-check
- State machine completeness (all states that are set are also checked)
- AI turn handler presence
- Win/lose condition presence
- Canvas element presence
- requestAnimationFrame presence
- Hex grid renderer presence (Heroes-like detection)
- Brace balance
- setTimeout gap detection

If validator score >= 85: skip Polish agent, deliver draft directly.
If validator score 60-84: run Polish agent with specific feedback.
If validator score < 60: escalate — run a targeted Repair agent with access to all
  Phase 2 module files (not just the draft HTML).

**Polish Agent:**

**Reads:** `VALIDATION_REPORT.json` + `game-draft.html`

```
You are a game polish agent. Fix specific issues found by the automated validator.

Validator report:
[VALIDATION_REPORT.json contents]

Game file:
[game-draft.html contents]

Fix ONLY the issues listed as CRITICAL in the validator report. For each issue:
- State which issue you are fixing
- Make a surgical fix (minimum code change)
- Do not refactor working code
- Do not change feature behavior while fixing

After fixing all critical issues, also:
- Verify all spell buttons have onclick handlers that are defined
- Verify the AI turn is triggered when gs.phase === 'ENEMY_TURN'
- Ensure the game has a title screen before battle starts
- Ensure there is a victory/defeat screen when checkWin() returns non-null

Output ONLY the complete fixed HTML file.
```

**Output:** `jobs/{JOB_ID}-game-final.html`

---

## 4. Quality Gates

### Gate 1: After Architecture (Phase 1 → Phase 2)

**Checks run before spawning Phase 2 agents:**

```python
def gate_1_architecture(spec_path: Path) -> tuple[bool, list[str]]:
    errors = []
    try:
        spec = json.loads(spec_path.read_text())
    except json.JSONDecodeError as e:
        return False, [f"GAME_SPEC.json is not valid JSON: {e}"]

    required_top_keys = ["shared_types", "constants", "function_signatures",
                         "module_assignments", "units", "spells", "required_features"]
    for key in required_top_keys:
        if key not in spec:
            errors.append(f"Missing required key in spec: {key}")

    # All module_assignments must cover all function_signatures
    assigned_funcs = set()
    for funcs in spec.get("module_assignments", {}).values():
        assigned_funcs.update(funcs)
    for func in spec.get("function_signatures", {}):
        if func not in assigned_funcs:
            errors.append(f"Function '{func}' in signatures but not assigned to any module")

    # units array must have at least 10 entries
    if len(spec.get("units", [])) < 10:
        errors.append(f"Spec has only {len(spec.get('units',[]))} units — expected 10+")

    return len(errors) == 0, errors
```

**On failure:** Re-run architecture agent with feedback, max 1 retry.
If retry also fails: fall back to single-agent pipeline (existing `pipeline.py` behavior).

### Gate 2: After Each Parallel Agent (Phase 2 individual)

Run immediately when each agent completes, before integration:

```python
AGENT_CHECKS = {
    "visual": [
        "hexToPixel",
        "drawHexGrid",
        "drawUnit",
    ],
    "combat": [
        "calculateDamage",
        "applyDamage",
        "executeAttack",
        "checkWin",
        "getReachableHexes",
    ],
    "state": [
        "initBattle",
        "nextTurn",
        "UNIT_DATA",
        "SPELL_DATA",
    ],
    "ai": [
        "doAITurn",
        "scoreTarget",
    ],
    "ui": [
        "buildHTML",
        "initTouchHandlers",
        "updateInitiativeBar",
    ],
}

def gate_2_module_check(module_name: str, js_content: str) -> tuple[bool, list[str]]:
    errors = []
    required = AGENT_CHECKS.get(module_name, [])
    for func in required:
        # Check function is defined (not just called)
        pattern = rf"\b(?:function\s+{func}|const\s+{func}\s*=|{func}\s*=\s*function)"
        if not re.search(pattern, js_content):
            errors.append(f"Required function '{func}' not defined in {module_name}.js")

    # Check for "colored rectangles" smell: agent wrote minimal stub
    if len(js_content.strip()) < 500:
        errors.append(f"{module_name}.js is suspiciously short ({len(js_content)} chars)")

    # Check for no-op canvas stubs: "ctx.fillRect(0,0,100,100)" with nothing else
    if module_name == "visual":
        sprite_funcs = len(re.findall(r"function\s+draw\w+", js_content))
        if sprite_funcs < 5:
            errors.append(f"Visual agent only defined {sprite_funcs} sprite functions — expected 10+")

    return len(errors) == 0, errors
```

**On failure of an individual agent:** Retry that agent with specific feedback.
Integration waits for all agents to pass their individual gate before starting.

**"Colored rectangles" early detection:**
This is the pattern where an agent produces a stub game that fills hexes with solid
colors instead of real sprites. Catch it early:
- `visual.js` has fewer than 5 unique `function draw*` definitions → rework
- `combat.js` has `calculateDamage` as a 1-liner returning a constant → rework
- `state.js` has `UNIT_DATA` with fewer than 5 entries → rework
- Any module file is under 300 characters → rework

### Gate 3: After Integration (Phase 3 → Phase 4)

Run `game_validator.py` automatically:

```python
def gate_3_integration(html_path: Path) -> tuple[bool, dict]:
    from game_validator import validate_game
    html = html_path.read_text(encoding="utf-8", errors="replace")
    report = validate_game(html)

    # Hard failures (must not ship):
    blockers = []
    for issue in report["critical"]:
        blockers.append(issue["message"])

    passed = len(blockers) == 0 and report["score"] >= 60
    return passed, report
```

Additionally check for non-validator issues:
- File is a complete HTML document (`<!DOCTYPE html>` present, `</html>` present)
- File size is between 50KB and 400KB
- No `TODO` or `FIXME` comments in the JS section
- The string `requestAnimationFrame` appears at least once
- The string `doAITurn` or equivalent AI function is called (not just defined)
- At least one spell is referenced from a button handler

### Gate 4: After Polish (Phase 4 → Delivery)

Run `game_validator.py` again on the polished file.
Threshold for delivery: score >= 75.

If score still below 75 after one polish cycle: deliver the draft with a note to the
user explaining known issues (partial delivery is better than no delivery).

---

## 5. Context Passing Between Agents

### The Core Problem

When 5 agents write code in parallel, each agent invents its own variable names,
function names, and data structures. The integration agent then discovers they are
incompatible. Example failure modes seen in practice:

- visual.js expects `unit.emoji` but state.js stores `unit.icon`
- combat.js calls `getHexNeighbors(col, row)` but ai.js calls `getNeighbors({col, row})`
- state.js uses `gs.currentUnit` but ui.js reads `gs.activeUnit`

### Solution: Explicit Shared Type Contract (in GAME_SPEC.json)

The architecture agent writes a contract that every parallel agent MUST read and MUST
honor. The contract defines:

**1. Canonical field names (one source of truth):**
```
unit.id        — not unit.unitId, not unit.uid
unit.icon      — not unit.emoji, not unit.sprite
unit.col       — not unit.x, not unit.gridX, not unit.q
unit.row       — not unit.y, not unit.gridY, not unit.r
unit.hp        — not unit.health, not unit.hitPoints
unit.side      — 'player' | 'enemy'  (not 'player1', not 'blue', not 0/1)
gs.phase       — 'PLAYER_TURN' | 'ANIMATING' | 'ENEMY_TURN' | 'GAME_OVER'
```

**2. Function call signatures (not just names):**
```
hexToPixel(col, row, hexSize) → {x, y}
  — NOT hexToPixel({col, row}, hexSize)
  — NOT hexToPixel(col, row)  (hexSize is required parameter)

getReachableHexes(unit, allUnits, obstacles) → [{col, row}]
  — obstacles is array of {col, row} occupied hexes
  — NOT getReachableHexes(unit, gs) — avoid passing entire gs to pure functions
```

**3. Module boundary contract (what calls what):**
```
state.js → MAY call functions from: combat.js
visual.js → MAY call nothing from other modules (pure drawing)
ai.js → MAY call: scoreTarget (own), findBestMove (own), combat functions
ui.js → CALLS CALLBACKS ONLY — never imports game functions directly
integration glue → wires everything together by calling each module's public interface
```

**4. How to inject context into each agent:**

Each Phase 2 agent prompt begins with the complete GAME_SPEC.json injected as a code block,
plus explicit instructions:

```
IMPORTANT: Use EXACTLY the field names defined in the spec.
- Units use 'col' and 'row' for grid position, NOT x/y, NOT q/r
- The game state object is called 'gs' everywhere
- The canvas context parameter is always called 'ctx'
- Function 'hexToPixel' signature is exactly: hexToPixel(col, row, hexSize) → {x, y}
  Do not change the signature.
```

This prevents agents from inventing their own conventions.

### Conflict Prevention at Integration Time

The integration agent gets ALL five modules and has explicit instructions to resolve conflicts:

```
CONFLICTS TO WATCH FOR:
1. Duplicate constant declarations — if both state.js and combat.js define COLS,
   keep the state.js version and remove the duplicate
2. Different spellings of the same thing — normalize to spec names
3. Functions defined in multiple modules — keep the version from the module
   the spec assigned it to, remove duplicates
4. Cross-module calls that don't match signatures — fix the call site to match
   the defined signature, not the other way around
```

### Shared Constants Strategy

All numeric constants (grid dimensions, timing values, damage modifiers, AI thresholds)
must be defined ONCE in `state.js` as top-level constants, and the spec must list them all.

Agents are instructed: "Do not define COLS, ROWS, or any combat modifier as a magic number.
Reference the constants section of the spec and use those names."

At integration time, the integration agent verifies:
- No inline magic numbers for known constants (e.g., `0.05` instead of `ATK_DEF_MODIFIER`)
- All constant names match the spec

This single change prevents the most common integration failure: two modules using
different values for the same game parameter.

---

## 6. Game Validation Agent Design

### Existing Infrastructure

`apps/listen/game_validator.py` already implements a solid static analysis engine:

**What it already checks:**
- onclick handler → function definition cross-check
- State machine completeness (assign/check pairs)
- AI turn handler presence
- Win/lose condition presence
- Canvas element presence
- requestAnimationFrame presence
- AudioContext presence
- Hex grid renderer presence
- Brace balance
- setTimeout references to undefined functions
- Heroes-like game detection + extended checks (initiative, combat resolution, spell casting)

**Score: 0-100 based on:**
- -15 per critical issue
- -5 per warning

### What to Add

The current `game_validator.py` does text/regex analysis but cannot test functional behavior.
Here are concrete additions ordered by implementation cost vs. value:

#### Addition 1: Headless Browser Screenshot (High Value, Medium Cost)

Use Playwright (or Puppeteer) to load the game in a headless browser and:

```python
async def screenshot_validate(html_path: Path) -> dict:
    """Launch game in headless browser, take screenshot, analyze it."""
    from playwright.async_api import async_playwright
    import base64

    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page(viewport={"width": 375, "height": 812})

        # Load the file
        await page.goto(f"file://{html_path}")
        await page.wait_for_timeout(2000)  # Let game initialize

        # Take screenshot
        screenshot_bytes = await page.screenshot()

        # Check console errors
        errors = []
        page.on("console", lambda msg: errors.append(msg.text) if msg.type == "error" else None)

        # Check if canvas has been drawn to (not just black)
        canvas_data = await page.evaluate("""
            () => {
                const canvas = document.getElementById('game-canvas');
                if (!canvas) return null;
                const ctx = canvas.getContext('2d');
                const data = ctx.getImageData(0, 0, 100, 100).data;
                const nonBlack = Array.from(data).filter((v, i) => i % 4 !== 3 && v > 20).length;
                return { nonBlack, total: data.length };
            }
        """)

        # Check if UI elements exist
        ui_checks = await page.evaluate("""
            () => ({
                hasCanvas: !!document.getElementById('game-canvas'),
                hasInitBar: !!document.querySelector('#initiative-bar, .initiative-bar'),
                hasSpellButtons: document.querySelectorAll('.spell, .spell-btn, [data-spell]').length,
                hasUnitInfo: !!document.querySelector('#unit-info, .unit-info'),
            })
        """)

        await browser.close()

        return {
            "console_errors": errors,
            "canvas_drawn": canvas_data and canvas_data["nonBlack"] > 500,
            "ui_checks": ui_checks,
            "screenshot_bytes": base64.b64encode(screenshot_bytes).decode(),
        }
```

**What "colored rectangles" looks like in screenshot analysis:**
- Canvas is drawn (nonBlack > 500) but all pixels are uniform rectangles
- No variation in hex cell colors (all the same shade)
- No unit sprites (uniform-colored regions at unit positions)
- Run a simple entropy check: low pixel variance = stub rendering

```python
def detect_colored_rectangles(image_bytes: bytes) -> bool:
    """Return True if game appears to be colored rectangles (stub rendering)."""
    from PIL import Image
    import io, numpy as np
    img = Image.open(io.BytesIO(image_bytes)).convert('RGB')
    arr = np.array(img)
    # Check color variance — real sprites have high variance
    variance = arr.var()
    # Check if grid-like structure (many straight edges)
    # Low variance + simple structure = colored rectangles
    return variance < 1000  # empirical threshold
```

**Cost:** Requires `playwright` and `pillow` packages. These are not currently installed.
Plan: add as optional extras in `apps/listen/pyproject.toml`, skip if not available.

#### Addition 2: JavaScript Execution Test (Medium Value, Low Cost)

Use Node.js (already available on macOS) to execute the game's JavaScript in isolation
and run basic sanity checks:

```python
def js_execution_test(html_path: Path) -> dict:
    """Extract and run game JS with Node.js, check for runtime errors."""
    html = html_path.read_text(encoding="utf-8", errors="replace")

    # Extract JS sections
    scripts = re.findall(r"<script(?:[^>]*)>(.*?)</script>", html, re.DOTALL | re.IGNORECASE)
    js_content = "\n".join(scripts)

    # Build a test harness
    test_harness = f"""
// Mock browser globals
const document = {{
    getElementById: () => ({{ getContext: () => ({{ fillRect: ()=>{{}}, drawImage: ()=>{{}},
        beginPath: ()=>{{}}, arc: ()=>{{}}, fill: ()=>{{}}, stroke: ()=>{{}},
        fillText: ()=>{{}}, save: ()=>{{}}, restore: ()=>{{}}, translate: ()=>{{}},
        rotate: ()=>{{}}, scale: ()=>{{}}, createRadialGradient: ()=>({{}}) }}) }}),
    createElement: () => ({{ getContext: () => ({{ }) }}),
    querySelectorAll: () => [],
    querySelector: () => null,
    body: {{ innerHTML: '', appendChild: ()=>{{}} }},
}};
const window = {{ addEventListener: ()=>{{}}, requestAnimationFrame: ()=>{{}} }};
const localStorage = {{ getItem: ()=>null, setItem: ()=>{{}} }};
const AudioContext = class {{}};

// --- GAME CODE START ---
{js_content}
// --- GAME CODE END ---

// Test: can we call key functions?
const tests = {{}};
try {{
    if (typeof initBattle !== 'undefined') {{
        const gs = initBattle('castle', 'dungeon', 'normal');
        tests.initBattle = gs && gs.units && gs.units.length > 0 ? 'PASS' : 'FAIL';
    }} else {{
        tests.initBattle = 'MISSING';
    }}
    tests.checkWin = typeof checkWin !== 'undefined' ? 'PASS' : 'MISSING';
    tests.doAITurn = typeof doAITurn !== 'undefined' ? 'PASS' : 'MISSING';
    tests.hexToPixel = typeof hexToPixel !== 'undefined' ? 'PASS' : 'MISSING';
}} catch(e) {{
    tests.error = e.toString();
}}
console.log(JSON.stringify(tests));
"""

    result = subprocess.run(
        ["node", "-e", test_harness],
        capture_output=True, text=True, timeout=10,
    )

    if result.returncode != 0:
        return {"passed": False, "error": result.stderr[:500]}

    try:
        tests = json.loads(result.stdout.strip())
        passed = all(v == "PASS" for k, v in tests.items() if k != "error")
        return {"passed": passed, "tests": tests}
    except Exception:
        return {"passed": False, "raw_output": result.stdout[:500]}
```

**Cost:** Node.js is already installed on macOS. This test requires only stdlib tools.

#### Addition 3: Feature Presence Regex Checks (Low Value, Zero Cost)

Beyond what `game_validator.py` already does, add game-type-specific checks:

```python
HOMM3_REQUIRED_PATTERNS = {
    "pegasus_unit": [r"\bpegasus\b", r"Pegasus"],
    "campaign_mode": [r"\binitiative\b", r"turnOrder"],
    "save_system": [r"localStorage\.(setItem|getItem)"],
    "title_screen": [r"title.{0,30}screen|menu.{0,30}screen|MENU", re.IGNORECASE],
    "victory_screen": [r"victory|победа|game.?over", re.IGNORECASE],
    "float_numbers": [r"float.{0,20}number|damage.{0,20}float|ANIM_FLOAT"],
}

def check_homm3_features(html: str) -> dict:
    results = {}
    for feature, patterns in HOMM3_REQUIRED_PATTERNS.items():
        found = any(re.search(p, html) for p in patterns)
        results[feature] = "PASS" if found else "MISSING"
    return results
```

---

## 7. Timeout Prevention Strategies

### 7.1 Why the Current System Times Out

The existing `pipeline.py` sets:
- architect: 300s (5 min)
- builder: 600s (10 min)
- reviewer: 180s (3 min)
- validator: 120s (2 min)

For a HoMM3 clone, the builder needs to write ~5000 lines. Claude generates code at
roughly 500-800 lines per minute in `--print` mode. That means minimum 6-10 minutes
just for typing — plus all the thinking time. A 10-minute timeout is structurally
impossible for complex games.

The observed failure mode: timeout at 1800 seconds (30 min) in the worker's tmux
session, before the pipeline.py limit even fires. The worker has a `JOB_TIMEOUT_SECONDS = 7200`
(2 hour) limit but the game building started and Claude ran until the Claude CLI's own
session limit was hit.

### 7.2 The Chunk Delivery Strategy

**Core idea:** Never ask for the complete game in one agent call. Instead:

**Step 1: Minimal Playable Core (MPC)**
Ask Phase 2 agents to produce a stripped-down version first:
- state.js: just UNIT_DATA, initBattle, nextTurn, checkWin
- combat.js: just hexDistance, calculateDamage, applyDamage, executeAttack (no special abilities)
- visual.js: just hexToPixel, drawHexGrid with flat colors (no sprites yet)
- ai.js: just doAITurn with simple "attack nearest enemy" logic
- ui.js: just the HTML structure + canvas tap handler

This MPC delivers a working (ugly) game in ~15 minutes per agent.

**Step 2: Enhancement Pass (parallel)**
Once MPC integration passes gate 3 with score >= 60, run enhancement agents:
- sprite-agent: adds programmatic sprite generators for all units
- special-ability-agent: adds all 11 special abilities to combat.js
- polish-agent: adds animations, floating numbers, audio, title screen

**Step 3: Final integration**
Merge MPC + enhancements into the final HTML.

This prevents the "all or nothing" failure mode. The user gets a working game in 30-40
minutes total instead of a timeout at 30 minutes.

### 7.3 Progressive Enhancement Prompting

Instead of "write the complete combat system," split into:

**Prompt 1 (core — 10 min):**
```
Write the minimal combat system: calculateDamage, applyDamage, executeAttack.
No special abilities yet. Just the base formula from the spec.
```

**Prompt 2 (enhancements — 10 min, only if core passed):**
```
Add these special abilities to the combat system you just wrote:
jousting, lifeBleed, petrify, breathSplash. Each as a standalone function.
```

**Prompt 3 (remaining — 10 min, only if enhancements passed):**
```
Add the remaining special abilities: deathCloud, strikeAndReturn,
griffinUnlimitedCounter, wightRegenerate, curse, moraleAura.
```

Each pass is independently checkable and produces a deliverable, instead of needing
all 11 special abilities in one shot.

### 7.4 File Size as Quality Proxy

For game code, a short file almost always means incomplete implementation.
Use file size as a quality proxy at gate checks:

| Module | Minimum acceptable size |
|--------|------------------------|
| visual.js | 8,000 characters |
| combat.js | 6,000 characters |
| state.js | 5,000 characters |
| ai.js | 3,000 characters |
| ui.js | 3,000 characters |
| game-draft.html | 60,000 characters |
| game-final.html | 60,000 characters |

Below these thresholds: automatically flag for rework before integration.

### 7.5 Checkpoint Save System

For very complex games (GDD complexity score > 7), implement checkpoint saves:

1. Phase 2 agents write their modules to `jobs/{JOB_ID}-{module}.js`
2. These files persist even if the integration agent fails
3. If integration times out: create a partial `game-v1.html` with only modules
   that passed gate 2, and deliver that with a note
4. On retry: skip Phase 2 agents that already produced passing files,
   only re-run failed/missing modules

This is essentially a build system with dependency caching. Implementation:

```python
def get_completed_modules(job_id: str) -> dict[str, bool]:
    """Check which Phase 2 modules have already been built and passed gate 2."""
    completed = {}
    for module in ["visual", "combat", "state", "ai", "ui"]:
        path = JOBS_DIR / f"{job_id}-{module}.js"
        if path.exists():
            content = path.read_text()
            passed, _ = gate_2_module_check(module, content)
            completed[module] = passed
    return completed

def run_phase2_with_caching(job_id: str, spec: dict) -> dict[str, str]:
    """Run only modules that haven't been built yet."""
    completed = get_completed_modules(job_id)
    modules_to_build = [m for m, done in completed.items() if not done]
    # Only spawn agents for incomplete modules
    results = {}
    with ThreadPoolExecutor(max_workers=len(modules_to_build)) as executor:
        futures = {executor.submit(run_module_agent, job_id, m, spec): m
                   for m in modules_to_build}
        for future in as_completed(futures):
            module, content = future.result()
            results[module] = content
    return results
```

---

## 8. Industry Reference: Studio Pipelines

### 8.1 Pre-production → Production → QA

Game studios structure work into three phases:

**Pre-production (what we call Phase 1: Architecture)**
- Define the game's core loop and win condition
- Prototype the highest-risk features (does the hex grid render correctly?)
- Create all data structures and type definitions
- Write the design document (GDD)
- Duration: 10-25% of total project time

**Production (what we call Phase 2: Parallel Build)**
- Art team, programming team, audio team work simultaneously
- Communication is through the design document (equivalent to GAME_SPEC.json)
- Milestones are intermediate deliverables: alpha (playable core), beta (feature complete)
- Each team has a lead who is responsible for the interface contract with other teams

**QA (what we call Phase 3-4: Integration + Validation)**
- Integration builds (often called "daily builds" or CI)
- Bug reports go to the correct team based on module ownership
- Polish passes address cosmetic issues after feature-complete milestone

**Key lesson for AI agents:** Studios discovered that the most expensive bugs are those
found late. A character model that is the wrong scale (visual team mistake) discovered
in QA requires fixing in the visual module and re-integrating. The AI equivalent: a
`drawUnit()` that uses `unit.emoji` discovered during integration requires re-running
the visual agent. The fix is the same: shared type contracts reviewed and approved
before parallel work begins.

### 8.2 The Interface Contract Pattern (From Software Engineering)

The game studio model maps to the classic "contract-first" API design pattern:

1. Define all interfaces (function signatures, data shapes) before implementation
2. Different teams implement against the same interface specification
3. Integration only begins when all implementations are complete
4. Integration failures trace back to contract violations, not "bad code"

For AI agents: GAME_SPEC.json is the interface contract. Every parallel agent is
building against the same spec. Integration failures are traceable to specific
contract violations (wrong field name, wrong function signature).

### 8.3 What AI Agents Learn from Game Studios

| Studio Practice | AI Agent Equivalent | Current Gap |
|----------------|--------------------|-----------|
| Pre-production prototype | Architecture agent writes function stubs | Not done — architecture agent currently writes plans but not stubs |
| Daily builds catch integration issues | Integration agent runs frequently | Only runs once, at end |
| Art/programming interface specs | GAME_SPEC.json module_assignments | Exists in spec template but not enforced |
| Alpha milestone (playable core) | MPC delivery before enhancement | Not implemented |
| Bug triage by module | Error traces back to module owner | Integration errors are untraced |
| QA test plans | `game_validator.py` + js_execution_test | Validator exists, execution test missing |

### 8.4 The "Vertical Slice" Pattern

Rather than building all systems to completion sequentially, game studios often do a
"vertical slice" first: build ONE feature end-to-end across all layers.

Applied to game agents: before running the full 5-agent parallel build, run a
"vertical slice" agent that builds a minimal but playable version:
- One hex cell on screen
- One player unit (pikeman) and one enemy unit
- Click to attack, see damage number
- Win when enemy dies

This takes one agent ~10 minutes and proves the architecture is sound before
committing to the full 5-agent parallel build. If the vertical slice fails
(canvas doesn't render, touch doesn't work), the architecture has a fundamental
flaw that the parallel agents would all hit.

---

## 9. Implementation Recommendations

### 9.1 New File: `game_pipeline.py`

Create `apps/listen/game_pipeline.py` as a specialized pipeline for game tasks,
wrapping the existing `orchestrator.py` infrastructure.

Key differences from the general-purpose `pipeline.py`:
- Uses 5 specialized phase-2 agent prompts (not generic build prompt)
- Injects GAME_SPEC.json into every parallel agent
- Runs `game_validator.py` gates automatically
- Has checkpoint save/resume logic
- Uses module-specific timeout values (visual: 1800s, combat: 1800s, ai: 1200s)
- Delivers MPC first, then enhancement

**Detection logic (when to use game_pipeline vs. general pipeline):**

```python
GAME_INDICATORS = [
    "игр", "game", "homm", "heroes", "tactical", "battle",
    "hex", "canvas", "sprite", "rpg", "strategy"
]

def is_game_task(prompt: str) -> bool:
    lower = prompt.lower()
    return any(indicator in lower for indicator in GAME_INDICATORS)
```

### 9.2 Prompt Template Injection

Every Phase 2 agent prompt should use this preamble:

```python
PHASE2_PREAMBLE = """
You are a specialized game development agent. You are building ONE module of a larger game.

CRITICAL INSTRUCTIONS:
1. Read the GAME_SPEC.json section carefully — it defines ALL field names and function signatures
2. Use EXACTLY the field names from the spec (unit.col not unit.x, unit.hp not unit.health)
3. Use EXACTLY the function signatures from the spec (don't change parameter names or order)
4. Do NOT implement logic that belongs to other modules (see module_assignments in spec)
5. Write COMPLETE, WORKING code — no stubs, no TODOs, no placeholder comments
6. Code must be at least {min_lines} lines — if you finish early, add error handling and edge cases
7. Output ONLY JavaScript code — no explanation before or after the code block

GAME_SPEC.json:
{spec_json}

YOUR MODULE: {module_name}
YOUR ASSIGNED FUNCTIONS: {assigned_functions}
"""
```

### 9.3 Integration with Existing Orchestrator

The game pipeline should use the existing `run_subtask_with_self_heal()` from
`orchestrator_agents.py` for each Phase 2 agent. This gives:
- Automatic retry on failure
- Self-healing with error classification
- Health checks via `check_agent_health()`
- Session management via tmux

The main change is: don't use `decompose_task()` (which uses Claude to split the task).
Instead, use the fixed 5-agent decomposition described above. For game tasks, the
decomposition is known in advance.

```python
# In game_pipeline.py
from orchestrator import OrchestrationPlan, Subtask, run_orchestration

def build_game_plan(job_id: str, spec: dict, prompt: str) -> OrchestrationPlan:
    """Build a fixed 5-agent plan for game Phase 2, using the spec as context."""
    plan = OrchestrationPlan(master_job_id=job_id, original_prompt=prompt)

    spec_json = json.dumps(spec, indent=2)

    modules = [
        ("visual", VISUAL_AGENT_PROMPT.format(spec_json=spec_json), 1800),
        ("combat", COMBAT_AGENT_PROMPT.format(spec_json=spec_json), 1800),
        ("state",  STATE_AGENT_PROMPT.format(spec_json=spec_json),  1800),
        ("ai",     AI_AGENT_PROMPT.format(spec_json=spec_json),     1200),
        ("ui",     UI_AGENT_PROMPT.format(spec_json=spec_json),     1200),
    ]

    for i, (name, prompt_text, timeout) in enumerate(modules):
        plan.subtasks.append(Subtask(
            id=f"{job_id[:6]}-{name}",
            prompt=prompt_text,
            index=i,
        ))

    return plan
```

### 9.4 Quality Thresholds to Use

| Gate | Threshold | Action on Failure |
|------|-----------|-------------------|
| Architecture spec validity | All required keys present | Retry architecture agent once, then fallback |
| Per-module minimum size | See table in §7.4 | Retry that module agent once |
| Per-module required functions | All defined | Retry that module agent once |
| Integration validator score | >= 60 | Run polish agent |
| Final validator score | >= 75 | Deliver with notes if still failing |
| JS execution test | initBattle returns valid gs | Log warning, continue |

### 9.5 Deliverable Format

When the game pipeline completes, deliver:
1. The final HTML file as a Tailscale URL link (existing `jobs/` serving infrastructure)
2. A validation report summary in the Telegram message
3. If any critical issues remain: list them explicitly so user can request targeted fixes

Example summary format:
```
Game complete: Heroes of Strategy
Score: 82/100 — 2 warnings, 0 critical issues
Link: http://100.71.137.124:7600/files/{JOB_ID}-game-final.html

Warnings:
- No AudioContext detected (game has no sound)
- No window resize handler (fixed layout)
```

---

## 10. Integration with Existing Code

### Files to Create

| New File | Purpose | Based On |
|----------|---------|---------|
| `apps/listen/game_pipeline.py` | Main entry point for game tasks | `pipeline.py` (add game-specific routing) |
| `apps/listen/game_spec_builder.py` | Architecture agent helper + gate 1 logic | New |
| `apps/listen/game_modules.py` | Phase 2 agent prompt templates + gate 2 logic | New |

### Files to Modify

| Existing File | Change | Priority |
|--------------|--------|---------|
| `apps/listen/pipeline.py` | Add `is_game_task()` detection, route to `game_pipeline.py` | High |
| `apps/listen/game_validator.py` | Add `js_execution_test()` and `check_homm3_features()` | Medium |
| `apps/listen/orchestrator.py` | Add `build_game_plan()` that uses fixed decomposition | Medium |
| `apps/listen/pyproject.toml` | Add `playwright` and `pillow` as optional extras | Low |

### Existing Code That Stays Unchanged

- `orchestrator_agents.py` — `run_subtask_with_self_heal()`, `check_agent_health()`, `kill_agent()` all used as-is
- `worker.py` — no changes needed
- `main.py` / `channels/telegram.py` — routing logic unchanged
- `.claude/agents/` — architect.md, builder.md, reviewer.md, validator.md used as-is for non-game tasks

### Estimated Implementation Time

| Component | Time | Complexity |
|-----------|------|-----------|
| game_pipeline.py (orchestration logic) | 4-6 hours | Medium |
| game_spec_builder.py (gate 1 + architecture prompt) | 2-3 hours | Low |
| game_modules.py (5 agent prompts + gate 2) | 3-4 hours | Low |
| game_validator.py additions (js_execution_test) | 2-3 hours | Low |
| Integration into pipeline.py routing | 1-2 hours | Low |
| Testing on HoMM3 clone | 4-8 hours | High |
| **Total** | **16-26 hours** | — |

---

## Appendix A: Quick Reference — What Each Phase Produces

```
Phase 1 (Architecture, 5 min):
  → jobs/{JOB_ID}-spec.json
     Contains: shared types, function signatures, module assignments, all constants

Phase 2 (Parallel Build, 20-30 min wall-clock):
  → jobs/{JOB_ID}-visual.js   (hex grid, sprites, animations)
  → jobs/{JOB_ID}-combat.js   (damage, pathfinding, special abilities)
  → jobs/{JOB_ID}-state.js    (game state machine, UNIT_DATA, SPELL_DATA)
  → jobs/{JOB_ID}-ai.js       (enemy AI decision tree)
  → jobs/{JOB_ID}-ui.js       (HTML structure, touch handlers, panels)

Phase 3 (Integration, 40 min):
  → jobs/{JOB_ID}-game-draft.html  (all modules merged into single file)

Phase 4 (Validation + Polish, 20-30 min):
  → jobs/{JOB_ID}-validation.json  (validator report)
  → jobs/{JOB_ID}-game-final.html  (polished, deliverable)
```

## Appendix B: Failure Modes and Recovery

| Failure Mode | Detection | Recovery |
|-------------|-----------|---------|
| Architecture produces empty spec | gate_1 file size check | Retry once with explicit example |
| Parallel agent times out | worker timeout | Retry with simplified scope (no specials, fewer units) |
| Parallel agent produces stub | gate_2 size + function count | Retry with "REWORK: your output is too short" |
| Integration creates conflicting variable names | gate_3 JS execution error | Run conflict-resolution patch agent |
| Validator score < 60 after polish | gate_4 score | Deliver draft + detailed issue list to user |
| All phases complete but game is all black canvas | screenshot entropy check | Run targeted visual-debug agent |
| Game crashes on unit click | JS execution test | Run targeted state machine debug agent |

---

*Document created by research session 2026-03-14.*
*Target system: mac-mini-agent at apps/listen/*
