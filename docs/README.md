# Texas Hold'em — 1v1 vs AI

A single-file browser-based Texas Hold'em game with a sophisticated AI opponent.

## Quick Start

Open `src/index.html` in any browser. No server, no build step required.

## Project Structure

```
texas-holdem/
  docs/
    DESIGN.md              Master design document
    ADMIN_PARAMETERS.md    All admin-configurable parameters
    HANDOFF.md             Claude Code handoff instructions
  src/
    index.html             The complete game (single file)
  tests/
    (test output logs)
  README.md                This file
```

## Running Tests

Append a URL parameter to `index.html` to run a test suite:

| Suite | URL |
|---|---|
| Deck | `index.html?test=deck` |
| Hand evaluator | `index.html?test=eval` |
| Equity / odds | `index.html?test=odds` |
| User manager | `index.html?test=user` |
| AI engine | `index.html?test=ai` |
| Full hand | `index.html?test=hand` |
| Theme system | `index.html?test=theme` |
| AI bluff rate | `index.html?test=bluff` |

## Admin Panel

Click the gear icon in the game UI to access the admin panel. See `docs/ADMIN_PARAMETERS.md` for full parameter reference.

## Design

See `docs/DESIGN.md` for the complete architecture, state machines, and AI design.

## v2 Roadmap

- User login and multi-user profiles
- Additional themes (luxury casino, retro Vegas)
- Admin role-gating
