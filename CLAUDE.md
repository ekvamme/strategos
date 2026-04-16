# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Strategos is a tactical 5x5 grid board game rendered on an HTML5 Canvas with isometric projection. The entire game lives in a single `strategos.html` file (~8,700 lines). There is no build system, bundler, or package manager — just open the HTML file in a browser.

## Running

- **Play the game:** Open `strategos.html` in any modern browser.
- **AI regression harness:** `node ai_vs_ai.js [numMatches] [budget]` — runs headless Hard-AI-vs-Hard-AI matches and reports win/loss/draw rates. The harness evals the `<script>` from `strategos.html` with DOM mocks, so the path constant `HTML_PATH` inside `ai_vs_ai.js` must point to the actual location of `strategos.html` on disk.
- **Legacy Python versions:** `game.py` (PvP), `game_ai.py` (vs AI), `game_ai_vs_ai.py` (headless sim) are older Pygame implementations with outdated stats/mechanics. The HTML version is canonical.

## Architecture (strategos.html)

Everything is in one `<script>` block. Key sections in order:

1. **Constants & STATS table** (lines ~54–97) — grid dimensions, zone definitions, the `STATS` object defining every unit type's HP/mov/cost/weight/range/atk/dfn ratings, upgrade paths, and token costs.
2. **SVG Sprite system** (~100–370) — inline SVG templates per unit type, parameterized by `{TEAM}` color. Built into cached `Image` objects at startup.
3. **Unit class** (~374–410) — per-instance state: position, planned orders, facing, stealth/channel/archmage cycle state, synergy trackers.
4. **Game class** (~413–2638) — all game logic:
   - `nextPhase()` — the phase state machine. Local modes cycle: `BUDGET → SHOP_P1 → HANDOFF → SHOP_P2 → PLACE_P1 → HANDOFF → PLACE_P2 → PLAN_P1 → HANDOFF → PLAN_P2 → READY_ACTION → ACTION_RESOLVE → loop`. AI/Campaign skip HANDOFFs. Remote PvP uses `remoteNextPhase()` with concurrent client progression.
   - `getValidMoves()` — BFS movement with diagonal support, charge detection (2-tile straight for weight-3 mov-2+ units), path-through vs terminal occupation rules.
   - `getValidAttacks()` — melee (adjacent orthogonal + optional diagonal), tile-target ranged (Chebyshev radius), directional firebolt.
   - `resolveAction()` (~871–2000) — the simultaneous resolution engine. Order: phalanx detection → phalanx charge halt → charge defense → trample → collision/tile-superiority (momentum > weight > HP) → defender-holds → bump/squeeze → Archmage blink → movement commit → trap detonation → beacon mark → shadow strike → arrow/firebolt/tile-target fire → cleave → lend aid → inspire → culling attrition → dead removal → animation build.
   - `computeSynergies()` — detects High Ground, Steady Aim, Last Stand, Phalanx per planning tick.
   - `hit()` — damage formula: `max(1, atk[type] - (dfn[type] + aura + lastStand + phalanx)) * multiplier`, with halving for AOE splash.
   - AI methods: `aiShop()`, `aiPlace()`, `aiPlan()` with Easy/Medium/Hard variants. Hard AI uses role-based tactics (melee advance + charge preference, ranged stay-at-range with firing-lane scoring, focus-fire on lowest-HP enemy).
5. **Renderer class** (~2640–7294) — all Canvas drawing: isometric grid, unit sprites, plan previews, attack highlights, action animations, full UI for every phase (menu, tutorial with 12 pages, shop, placement, planning, budget, remote setup, game-over).
6. **Input handling & game loop** (~7296–end) — `pixelToTileSmart()` for click→tile with sprite hit-testing, `handleButtonClick()` dispatch table for all UI buttons, keyboard shortcuts, `gameLoop()` with `requestAnimationFrame`.

## Key Design Patterns

- **Plan→Resolve simultaneous turns:** During PLAN phases, each unit receives `plannedR/C` (destination) and `plannedAtk` (attack order). `resolveAction()` then processes all plans at once — no unit acts before another.
- **Tile Superiority tiebreakers:** momentum (charging) > weight > remaining HP. Only full ties cause mutual bounce.
- **AI is hardcoded to P2.** The `ai_vs_ai.js` harness plays P1 by flipping all player labels (`flipAndSwap`), calling `aiPlan()`, then flipping back. Both sides plan against pre-seeded current positions for fairness.
- **Remote PvP** uses PeerJS/WebRTC. Host generates a 6-char code. Both clients run their own game loop; they sync at `REMOTE_WAIT` boundaries with a 3-2-1 countdown, then resolve independently using the same seeded PRNG (`Mulberry32`).
- **All rendering is imperative Canvas 2D** — no DOM elements inside the game area. UI buttons are drawn to canvas and tracked via `game.clickBounds` hit regions.

## Unit System Quick Reference

Three class trees, each L1 (1 token) → L2 (3 tokens) → L3 (6 tokens):
- **Melee:** Squire → Knight → Paladin (weight 3, blocks arrows, charge, cleave, aura)
- **Ranged:** Archer → Rogue → Assassin (weight 1, tile-target, traps, stealth/shadow-strike)
- **Magic:** Adept → Mage → Archmage (weight 2, firebolt/fire-rain, diagonal move+atk, blink)

See `glossary.txt` for full mechanics documentation.

## When Editing

- All game logic, rendering, and UI are in the single `<script>` in `strategos.html`. Search by function name.
- After any change to game logic or stats, run `node ai_vs_ai.js 1000` to check for regressions (crashes, extreme win-rate skew). Update `HTML_PATH` in `ai_vs_ai.js` if needed.
- The `STATS` object is the single source of truth for all unit balance numbers. The glossary and any comments may lag behind.
- When adding new unit abilities, wire them into: (1) the PLAN step UI in `handleButtonClick` + tile click handler, (2) `clearPlans()` in `ai_vs_ai.js`, (3) `resolveAction()`, (4) the AI planner if the ability should be used by AI, (5) the Renderer for visual feedback.
