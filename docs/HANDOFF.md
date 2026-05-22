# Claude Code Handoff — Texas Hold'em Project

## How to use this file
Paste this entire document as your first message to Claude Code, then say:
"Please read all the docs in the docs/ folder and begin implementation starting with the Deck module."

---

## Project Handoff

You are taking over implementation of a 1v1 Texas Hold'em game from a design session. All major design decisions have been made and approved. Do not re-ask design questions — implement according to the spec. If you encounter a genuine ambiguity not covered by the design docs, flag it before proceeding.

---

## Project Location

All files live in: `texas-holdem/`

```
texas-holdem/
  docs/
    DESIGN.md              ← Master design document (read this first)
    ADMIN_PARAMETERS.md    ← All admin-configurable parameters with meanings
    HANDOFF.md             ← This file
  src/                     ← All source code goes here
  tests/                   ← Test output and test runner hooks
```

---

## Critical Constraints

1. **Single file deliverable.** The final game is one HTML file: `src/index.html`. All CSS and JS are embedded in it. No build step, no npm, no bundler.

2. **Module architecture must be respected.** Each module (Deck, HandEvaluator, UserManager, AIEngine, GameEngine, Renderer, ThemeManager) is a self-contained JS object/class inside the single file. Modules communicate only through their defined public interfaces. No module touches localStorage except UserManager.

3. **Theme system uses CSS variables + swappable assets.** No hardcoded colours anywhere in layout code. All colours reference CSS variables defined in a theme block. Adding a new theme = adding one CSS block + assets.

4. **Repository pattern for storage.** UserManager namespaces all data under `users["default"]` in localStorage. This is intentional — v2 will add real user login and the namespace structure is already ready for it.

5. **Admin panel is part of the game UI.** Accessible via a gear icon button. Stores config in `texas_holdem_data.config.admin` in localStorage. All AI parameters, blind structure, and game settings are configurable there.

6. **Test hooks via URL params.** Every module has unit tests activated by `?test=<name>`. Tests display results in an overlay panel. Do not remove these hooks.

---

## Implementation Order (approved)

Complete each step and run its tests before moving to the next:

1. `Deck` module → tests T01, T02 (`?test=deck`)
2. `HandEvaluator` module → tests T03, T04, T05 (`?test=eval`, `?test=odds`)
3. `UserManager` module → tests T06, T07 (`?test=user`)
4. `AIEngine` module → tests T08, T09 (`?test=ai`, `?test=odds`)
5. `GameEngine` → tests T10, T11, T12 (`?test=hand`)
6. `Renderer` + `ThemeManager` → test T15 (`?test=theme`)
7. Admin panel UI
8. Full integration → tests T13, T14, T16, T17, T18
9. Polish, bug fixes, update design docs if anything changed

---

## Key Design Decisions Summary

### AI Engine
- Three-layer pipeline: OddsLayer + PatternLayer + BluffLayer → DecisionCombiner
- Weights: equity(0.5) + pattern(0.3) + bluff(0.2) — all configurable in admin panel
- OddsLayer: Monte Carlo simulation, default 500 runs — configurable
- Raise sizing: mixed — hand-strength ranges with ±15% randomization noise
- Decision thresholds: score < 0.3 → fold, 0.3–0.6 → call, > 0.6 → raise — configurable
- PatternLayer confidence scales with handsPlayed (low trust below 10 hands)

### Opponent Profile (persisted in localStorage)
```javascript
{
  userId: "default",
  handsPlayed: 0,
  vpip: 0.0,
  aggressionFactor: 1.0,
  foldToBluff: 0.5,
  showdownWinRate: 0.5,
  handHistory: []           // max length configurable, default 50
}
```

### Blind Structure
- Default: SB 25 / BB 50
- Escalation: off by default, configurable
- Dealer button alternates each hand

### Chips & Game End
- 1000 chips each at start
- Play until one player reaches 0
- Bust check after every HAND_OVER state

### Visual Theme
- Starting theme: clean & modern
- CSS variables for all colours/fonts
- Three planned themes: modern (now), casino (future), retro (future)

### Animations
- Full card dealing animation (cards slide in from deck)
- Chip movement animation (chips slide to pot / to winner)
- AI think delay: 800ms default (configurable)

### Hints System
- Toggle button in game UI
- Basic mode: hand name + simple advice
- Advanced mode: equity estimate + pot odds
- Default off (configurable in admin)

---

## Full Test List

| Test ID | Level | Module | What it verifies | URL param |
|---|---|---|---|---|
| T01 | Unit | Deck | 52 unique cards, no duplicates after shuffle | `?test=deck` |
| T02 | Unit | Deck | Deal reduces deck size correctly | `?test=deck` |
| T03 | Unit | HandEvaluator | All 9 hand ranks identified correctly | `?test=eval` |
| T04 | Unit | HandEvaluator | Tiebreaker logic correct | `?test=eval` |
| T05 | Unit | HandEvaluator | Equity within ±3% of known values | `?test=odds` |
| T06 | Unit | UserManager | Read/write/namespace isolation | `?test=user` |
| T07 | Unit | UserManager | Profile persists across simulated reload | `?test=user` |
| T08 | Unit | AIEngine | OddsLayer equity within ±3% on known hands | `?test=odds` |
| T09 | Unit | AIEngine | DecisionCombiner respects weight thresholds | `?test=ai` |
| T10 | Integration | GameEngine | Full hand plays to showdown without illegal state | `?test=hand` |
| T11 | Integration | GameEngine | Fold at any street resolves correctly | `?test=hand` |
| T12 | Integration | GameEngine | Blinds post correctly, pot math is exact | `?test=hand` |
| T13 | Integration | AIEngine | AI never acts on missing/undefined state | `?test=ai` |
| T14 | Integration | AIEngine | AI folds gracefully when equity is very low | `?test=ai` |
| T15 | Integration | ThemeManager | Theme swap doesn't break layout or game state | `?test=theme` |
| T16 | Gameplay | AIEngine | Bluff frequency within expected range over 100 hands | `?test=bluff` |
| T17 | Gameplay | AIEngine | AI adjusts behavior after 20+ hands of pattern data | `?test=bluff` |
| T18 | Gameplay | GameEngine | Game correctly ends when a player reaches 0 chips | `?test=hand` |

---

## What Has NOT Been Decided (implement with sensible defaults)

- Exact card rendering style (SVG cards drawn in CSS, standard suits/values)
- Exact animation durations (suggest 300ms deal, 200ms chip slide)
- Exact layout dimensions (responsive, works at 800px–1400px wide)
- Sound effects (not requested — do not add)
- Mobile responsiveness (nice to have but not required for v1)

---

## v2 Hooks Already In Architecture

These are built into the design now so v2 changes are minimal:
- `UserManager` namespaces by userId (`users["default"]`) → swap for real user ID
- Admin panel checks `user.role === "admin"` → add role to UserProfile in v2
- ThemeManager has a theme registry → add casino/retro theme entries
- All AI params in AdminConfig → no hardcoded values to hunt down
