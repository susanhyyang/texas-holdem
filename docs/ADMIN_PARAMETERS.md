# Admin Parameters Reference
**Version:** 1.0  
**Last updated:** 2026-05-22

This document lists every parameter configurable via the admin panel, its meaning, default value, valid range, and which module consumes it.

---

## 1. AI Behaviour

### 1.1 Decision Weights

These three weights control how the AI balances its decision-making. They must sum to 1.0.

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Equity weight | `ai.weights.equity` | `0.50` | 0.0 – 1.0 | How much the AI relies on pure hand strength and pot odds. Higher = more mathematical, less creative. |
| Pattern weight | `ai.weights.pattern` | `0.30` | 0.0 – 1.0 | How much the AI relies on reading the opponent's playing style. Higher = more adaptive to your tendencies. |
| Bluff weight | `ai.weights.bluff` | `0.20` | 0.0 – 1.0 | How much the AI's bluffing instinct influences its decisions. Higher = more deceptive, more unpredictable. |

> **Note:** The three weights are normalised automatically if they do not sum to 1.0.

---

### 1.2 Monte Carlo Simulation

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Simulation count | `ai.monteCarlo.simCount` | `500` | 50 – 5000 | Number of random board completions the AI runs to estimate hand equity. Higher = more accurate but slower. 500 is recommended. Below 100 is unreliable. Above 2000 may cause a noticeable pause on slow devices. |

---

### 1.3 Decision Thresholds

These thresholds map the AI's combined score to an action. Score is in range 0.0–1.0.

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Fold threshold | `ai.thresholds.fold` | `0.30` | 0.0 – 0.5 | Scores below this → AI folds. Lower = AI folds less (looser). |
| Raise threshold | `ai.thresholds.raise` | `0.60` | 0.5 – 1.0 | Scores above this → AI raises. Higher = AI raises less (more passive). |

> **Note:** Scores between fold and raise threshold → AI calls.

---

### 1.4 Bluff Frequency by Street

Base probability that the AI considers a bluff at each stage. Actual bluff decisions also depend on opponent pattern reads.

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Preflop bluff rate | `ai.bluff.preflop` | `0.05` | 0.0 – 0.3 | How often AI bluffs pre-flop. Keep low — pre-flop bluffs are rarely profitable. |
| Flop bluff rate | `ai.bluff.flop` | `0.15` | 0.0 – 0.5 | How often AI continuation-bets on the flop with a weak hand. |
| Turn bluff rate | `ai.bluff.turn` | `0.20` | 0.0 – 0.5 | How often AI barrels on the turn with a weak hand. |
| River bluff rate | `ai.bluff.river` | `0.25` | 0.0 – 0.5 | How often AI bluffs on the river. Higher is more aggressive but risky. |

---

### 1.5 Raise Sizing

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Weak hand raise min | `ai.raise.weakMin` | `0.40` | 0.2 – 0.8 | Minimum raise as a fraction of pot when AI has a weak hand. |
| Weak hand raise max | `ai.raise.weakMax` | `0.60` | 0.3 – 1.0 | Maximum raise as a fraction of pot when AI has a weak hand. |
| Strong hand raise min | `ai.raise.strongMin` | `0.70` | 0.4 – 1.5 | Minimum raise as a fraction of pot when AI has a strong hand. |
| Strong hand raise max | `ai.raise.strongMax` | `1.20` | 0.5 – 3.0 | Maximum raise as a fraction of pot when AI has a strong hand. |
| Randomization noise | `ai.raise.noise` | `0.15` | 0.0 – 0.3 | ±fraction added randomly to raise size to prevent predictability. |

---

## 2. Blind Structure

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Starting small blind | `blinds.small` | `25` | 5 – 200 | Chips the small blind player must post at the start of each hand. |
| Starting big blind | `blinds.big` | `50` | 10 – 400 | Chips the big blind player must post. Should be 2× small blind. |
| Escalation enabled | `blinds.escalate` | `false` | true / false | Whether blinds increase over time (tournament style). |
| Escalation interval | `blinds.escalateEvery` | `10` | 3 – 50 | Number of hands between each blind increase. |
| Escalation multiplier | `blinds.escalateBy` | `1.5` | 1.1 – 3.0 | Factor by which blinds are multiplied at each escalation step. e.g. 1.5 = blinds increase by 50% each interval. |

---

## 3. Game Setup

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Starting chips | `game.startingChips` | `1000` | 100 – 100000 | How many chips each player starts with at the beginning of a new game. |
| AI think delay | `game.aiDelay` | `800` | 0 – 3000 | Milliseconds the AI waits before acting. Adds realism. 0 = instant. |

---

## 4. Opponent Tracking

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Hand history length | `tracking.historyLength` | `50` | 10 – 500 | Maximum number of past hands stored in the opponent profile. Older hands are discarded when limit is reached. More history = longer AI memory but more storage used. |
| Minimum hands for pattern trust | `tracking.minHandsForPattern` | `10` | 5 – 50 | Number of hands the AI needs to observe before it trusts the pattern data enough to weight it fully. Below this number, pattern weight is scaled down proportionally. |

---

## 5. Hints

| Parameter | Key | Default | Range | Meaning |
|---|---|---|---|---|
| Hints enabled by default | `hints.defaultOn` | `false` | true / false | Whether the hint toggle starts on or off when a new game loads. |
| Hint detail level | `hints.level` | `"basic"` | `"basic"` / `"advanced"` | Basic: shows hand name and simple advice. Advanced: shows equity estimate and pot odds. |

---

## 6. Admin Config Storage

All admin parameters are stored in localStorage under the key `texas_holdem_data.config.admin` as a single JSON object. Example:

```json
{
  "ai": {
    "weights": { "equity": 0.5, "pattern": 0.3, "bluff": 0.2 },
    "monteCarlo": { "simCount": 500 },
    "thresholds": { "fold": 0.3, "raise": 0.6 },
    "bluff": { "preflop": 0.05, "flop": 0.15, "turn": 0.20, "river": 0.25 },
    "raise": { "weakMin": 0.4, "weakMax": 0.6, "strongMin": 0.7, "strongMax": 1.2, "noise": 0.15 }
  },
  "blinds": {
    "small": 25, "big": 50,
    "escalate": false, "escalateEvery": 10, "escalateBy": 1.5
  },
  "game": {
    "startingChips": 1000,
    "aiDelay": 800
  },
  "tracking": {
    "historyLength": 50,
    "minHandsForPattern": 10
  },
  "hints": {
    "defaultOn": false,
    "level": "basic"
  }
}
```

---

## 7. Resetting to Defaults

The admin panel includes a "Reset to defaults" button that overwrites the stored config with the default values listed in this document. It does not affect the opponent profile or hand history.
