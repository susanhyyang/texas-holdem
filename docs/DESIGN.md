# Texas Hold'em — Master Design Document
**Version:** 1.1  
**Status:** Approved — ready for implementation  
**Last updated:** 2026-05-22

---

## 1. Project Overview

A 1v1 Texas Hold'em game (human vs computer) built as a single Vanilla HTML/CSS/JS file. The computer AI is sophisticated: it calculates pot odds, reads opponent patterns, and bluffs. The game persists opponent statistics across sessions via localStorage, making the AI adaptive over time.

---

## 2. Confirmed Design Decisions

| # | Category | Decision |
|---|---|---|
| D1 | AI weights | equity 50% / pattern 30% / bluff 20% — all configurable via admin panel |
| D2 | Monte Carlo sims | 500 default — configurable via admin panel |
| D3 | Opponent profile | 6 fields; hand history length configurable via admin panel |
| D4 | AI raise sizing | Mixed: hand-strength ranges + randomization noise |
| D5 | Blind structure | Fully configurable via admin panel (default SB 25 / BB 50) |
| D6 | Tech stack | Vanilla HTML/CSS/JS, single file |
| D7 | Theme system | CSS variables + swappable assets (Option 2) |
| D8 | Starting theme | Clean & modern |
| D9 | Chips | 1000 each, play until broke |
| D10 | Hints | Optional toggle (on/off) |
| D11 | Animations | Full — card dealing & chip movements |
| D12 | Login | None in v1; single assumed user; architecture ready for v2 multi-user |
| D13 | User storage | UserManager module, namespaced as users["default"] in localStorage; repository pattern |
| D14 | Admin access | Button in game UI; role-gated in v2 when login is added |
| D15 | Split pot | Pot split evenly; odd chip goes to the dealer button holder |
| D16 | All-in | When a called all-in occurs, remaining community cards run out automatically with no further betting |
| D17 | Heads-up position | Standard heads-up rules: dealer = SB, acts first preflop, acts last on all post-flop streets |
| D18 | Bluff score model | Probabilistic random trigger — each street rolls against the bluff rate; if triggered, bluff score = 1.0, else 0.0 |
| D19 | Profile stat weighting | Opponent stats (VPIP, aggressionFactor, etc.) weighted toward recent hands using exponential moving average |
| D20 | Animation gating | GameEngine awaits animateDeal() and animateChips() Promises before advancing game state |

---

## 3. System Architecture

### 3.1 Module Map

```
┌─────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE                      │
│  UserManager                    ThemeManager         │
│  (localStorage, user profiles)  (CSS vars, assets)  │
└────────────────┬───────────────────────┬────────────┘
                 │ reads/writes          │ reads
┌────────────────▼───────────────────────▼────────────┐
│                   CORE LOGIC                         │
│              GameEngine                              │
│         (state machine, betting rounds)              │
└────┬──────────────┬──────────────────┬──────────────┘
     │ commands     │ commands         │ commands
┌────▼────┐  ┌──────▼──────┐  ┌───────▼──────┐
│  Deck   │  │HandEvaluator│  │  AIEngine    │
│shuffle  │  │rank, equity │  │odds+pattern  │
│deal     │  │Monte Carlo  │  │+bluff        │
└────┬────┘  └──────┬──────┘  └───────┬──────┘
     │ results      │ results          │ results
┌────▼──────────────▼──────────────────▼──────────────┐
│                  Renderer                            │
│         (DOM, animations, hints UI)                  │
└─────────────────────────────────────────────────────┘
```

### 3.2 Module Responsibilities

| Module | Owns | Does NOT own |
|---|---|---|
| `GameEngine` | Game state, betting logic, hand progression | Rendering, AI decisions, storage |
| `Deck` | 52-card deck, shuffle, deal | Hand evaluation |
| `HandEvaluator` | Hand ranking, 7-card evaluation, equity simulation | Game flow |
| `AIEngine` | AI decision making, profile updates | Game state mutations |
| `UserManager` | All localStorage read/write, user namespacing | Game logic |
| `ThemeManager` | CSS variable swapping, asset swapping | Layout or game state |
| `Renderer` | All DOM mutations, animations | Game logic |

### 3.3 Module Public Interfaces

```javascript
Deck
  .shuffle()                          → void
  .deal(n)                            → Card[]
  .burnAndTurn()                      → Card

HandEvaluator
  .evaluate(cards[7])                 → { rank, name, tiebreakers }
  .equity(holeCards, board, nSims)    → float (0–1)

GameEngine
  .startGame()                        → void
  .playerAction(type, amount)         → void
  .getState()                         → GameState

AIEngine
  .decide(gameState, profile)         → { action, amount }
  .updateProfile(handResult)          → void

UserManager
  .getProfile()                       → UserProfile
  .saveProfile(profile)               → void
  .getHandHistory()                   → HandSummary[]
  .getConfig()                        → AdminConfig
  .saveConfig(config)                 → void

ThemeManager
  .applyTheme(name)                   → void
  .listThemes()                       → string[]

Renderer
  .render(gameState)                  → void
  .animateDeal()                      → Promise
  .animateChips()                     → Promise
  .showHint(hintText)                 → void
```

---

## 4. Game State Machine

### 4.1 Top-Level States

```
IDLE
  └─► SETUP          (shuffle, post blinds, deal 2 hole cards each)
        └─► PREFLOP   (betting round 1)
              └─► FLOP     (deal 3 community cards, betting round 2)
                    └─► TURN    (deal 1 community card, betting round 3)
                          └─► RIVER   (deal 1 community card, betting round 4)
                                └─► SHOWDOWN  (reveal hands, award pot)
                                      └─► HAND_OVER
                                            ├─► SETUP (next hand, if both players have chips)
                                            └─► GAME_OVER (a player is broke)

Any betting state ──fold──► FOLD_WIN ──► HAND_OVER

All-in path (D16):
Any betting state ──all-in called──► ALL_IN_RUNOUT
  └─► (deal remaining community cards automatically, no betting)
        └─► SHOWDOWN ──► HAND_OVER
```

### 4.2 Betting Round Sub-States (shared by PREFLOP, FLOP, TURN, RIVER)

```
ROUND_START
  └─► PLAYER_TURN  (determine who acts: human or AI)
        ├─► AWAIT_INPUT    (human: show fold/call/raise buttons)
        └─► AI_THINK       (AI: run OddsLayer + PatternLayer + BluffLayer)
              └─► BET_RESOLUTION  (update pot, stacks, hand history)
                    ├─► FOLD_WIN        (someone folded — exit round)
                    ├─► ROUND_COMPLETE  (both acted, bets equal — exit round)
                    └─► PLAYER_TURN     (more action needed — loop)
```

### 4.3 Legal Actions Per State

| State | Human actions | AI actions |
|---|---|---|
| AWAIT_INPUT | fold, call, check, raise | — |
| AI_THINK | — | fold, call, check, raise |
| ALL_IN_RUNOUT | — (observe) | — (observe) |
| SHOWDOWN | — (observe) | — (observe) |
| GAME_OVER | new game | — |

### 4.4 Heads-Up Position Rules (D17)

In heads-up play the dealer button holder posts the **small blind** and acts **first preflop**. On all post-flop streets (flop, turn, river) the dealer acts **last**. This is standard heads-up rules and reverses from full-table position conventions.

### 4.5 Split Pot Rules (D15)

At showdown, if both players hold equal best hands the pot is split evenly. If the pot is an odd number of chips the extra chip is awarded to the **dealer button holder**.

### 4.6 Animation Gating (D20)

`GameEngine` awaits `Renderer.animateDeal()` before opening the first betting round, and awaits `Renderer.animateChips()` before transitioning out of `HAND_OVER`. No game state advances until its associated animation Promise resolves.

---

## 5. AI Engine Design

### 5.1 Decision Pipeline

```
OddsLayer          PatternLayer         BluffLayer
(equity score)     (pattern score)      (bluff score)
     │                   │                   │
     └───────────────────┴───────────────────┘
                         │
                  DecisionCombiner
              equity(w1) + pattern(w2) + bluff(w3)
                         │
                   Action output
              FOLD / CALL / RAISE(amount)
                         │
               Post-hand profile update
          (write updated stats to UserManager)
```

### 5.2 OddsLayer
- Runs Monte Carlo simulation (default 500 runs, configurable)
- Randomly completes the board N times with unseen cards
- Counts wins / total = equity estimate
- Compares equity to pot odds to produce a score

### 5.3 PatternLayer
- Reads opponent profile from UserManager
- Computes a situational score based on:
  - Is opponent loose (high VPIP)? → loosen calling range
  - Is opponent aggressive? → tighten, trap more
  - Does opponent fold to bluffs? → increase bluff score
- Confidence scales with `handsPlayed` (low confidence early on, see `tracking.minHandsForPattern`)

### 5.4 BluffLayer (D18)
- Each street, roll `Math.random()` against the street's bluff rate
- If the roll triggers: bluff score = **1.0** (full bluff intent this hand)
- If the roll does not trigger: bluff score = **0.0**
- Effective bluff rate adjusted before the roll:
  - Adjusted **up** if PatternLayer detects opponent folds to pressure
  - Adjusted **down** if opponent has high showdown win rate (calling station)
- This mixed-strategy model makes the AI unexploitable — same spot, different outcomes

### 5.5 DecisionCombiner
- Final score = `(equity × w1) + (pattern × w2) + (bluff × w3)`
- w1, w2, w3 are admin-configurable weights (default 0.5, 0.3, 0.2)
- Score maps to action thresholds:
  - score < 0.3 → FOLD
  - 0.3 ≤ score < 0.6 → CALL
  - score ≥ 0.6 → RAISE

### 5.6 Raise Sizing
- Base range determined by hand strength (weak: 40–60% pot, strong: 70–120% pot)
- Randomization noise of ±15% applied to base
- All-in considered when stack < 3× big blind

### 5.7 Profile Stat Update Model (D19)

Stats are updated after every hand using an **exponential moving average (EMA)** so recent hands carry more weight than old ones. Smoothing factor α = 0.1 (configurable via `tracking.emaAlpha` if added to admin panel):

```
newStat = α × observedValue + (1 − α) × oldStat
```

This means a single hand has a 10% influence on the stat, recent hands dominate over time, and early tendencies naturally decay.

### 5.8 Opponent Profile Schema

```javascript
UserProfile = {
  userId: "default",                  // v2: real user ID
  handsPlayed: 0,                     // total hands played
  vpip: 0.0,                          // voluntarily put in pot ratio (0–1)
  aggressionFactor: 1.0,              // raise:call ratio
  foldToBluff: 0.5,                   // fold frequency when re-raised (0–1)
  showdownWinRate: 0.5,               // win rate at showdown (0–1)
  handHistory: []                     // last N hand summaries (N configurable)
}

HandSummary = {
  handId: string,
  timestamp: number,
  holeCards: Card[],
  communityCards: Card[],
  actions: Action[],
  result: "win" | "loss" | "fold_win" | "fold_loss",
  potWon: number,
  showedDown: boolean
}
```

---

## 6. Theme System

### 6.1 Architecture
- Each theme is a CSS `:root` variable block + a set of named assets
- Switching themes: `ThemeManager.applyTheme(name)` swaps the CSS block and updates asset references
- All colors, fonts, and spacing reference CSS variables — no hardcoded values in layout code
- Adding a new theme requires: one CSS block + card back SVG + felt texture CSS

### 6.2 Theme Registry Structure
```javascript
THEMES = {
  "modern": {
    cssClass: "theme-modern",
    cardBack: "card-back-modern",    // SVG id or inline CSS
    feltStyle: "felt-modern"         // CSS class for table surface
  },
  "casino": { ... },    // future
  "retro":  { ... }     // future
}
```

### 6.3 CSS Variable Groups (all themes must define these)
- `--color-felt` — table surface color
- `--color-felt-accent` — table border/trim
- `--color-card-bg` — card face background
- `--color-card-border` — card border
- `--color-chip-primary` — main chip color
- `--color-text-primary` — main text
- `--color-text-muted` — secondary text
- `--color-action-fold` — fold button
- `--color-action-call` — call button
- `--color-action-raise` — raise button
- `--font-display` — display/heading font
- `--font-body` — body/UI font

---

## 7. Data Storage

### 7.1 localStorage Schema
```
localStorage
  └── "texas_holdem_data"
        ├── users
        │     └── "default"          (v2: keyed by real user ID)
        │           ├── profile       (UserProfile object)
        │           └── handHistory   (HandSummary[])
        └── config
              └── admin              (AdminConfig object)
```

### 7.2 Repository Pattern
All localStorage access goes through `UserManager`. No other module touches localStorage directly. In v2, `UserManager` can be re-implemented to call a backend API without changing any other module.

---

## 8. Admin Panel

### 8.1 Access
- Button visible in game UI (gear icon)
- In v2: only visible to users with `role: "admin"`

### 8.2 Configurable Parameters
See `docs/ADMIN_PARAMETERS.md` for full reference.

---

## 9. Test Plan

### 9.1 Test Activation
All tests activated via URL parameter: `?test=<name>`  
Results displayed in an overlay panel within the game.

### 9.2 Test Suite

| Test ID | Level | Module | What it verifies | URL param |
|---|---|---|---|---|
| T01 | Unit | Deck | 52 unique cards, no duplicates after shuffle | `?test=deck` |
| T02 | Unit | Deck | Deal reduces deck size correctly | `?test=deck` |
| T03 | Unit | HandEvaluator | All 9 hand ranks identified correctly | `?test=eval` |
| T04 | Unit | HandEvaluator | Tiebreaker logic (kicker, suit) correct | `?test=eval` |
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

## 10. Implementation Order

1. `Deck` module → T01, T02
2. `HandEvaluator` module → T03, T04, T05
3. `UserManager` module → T06, T07
4. `AIEngine` module → T08, T09
5. `GameEngine` → T10, T11, T12
6. `Renderer` + `ThemeManager` → T15
7. Admin panel
8. Full integration → T13, T14, T16, T17, T18
9. Polish, bug fixes, design doc updates

---

## 11. Future (v2) Hooks

- `UserManager` already namespaces by userId — swap localStorage for API calls
- Admin panel visibility already conditional on role field in UserProfile
- ThemeManager already supports multiple theme registry entries
- All AI weights already stored in AdminConfig, not hardcoded
