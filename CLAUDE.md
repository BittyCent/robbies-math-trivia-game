# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Game

Open `index.html` directly in a browser — no build step, no server, no dependencies. There are no tests, no linter, and no package manager.

To preview changes during development:
```
open index.html          # macOS
```

## Architecture

Everything lives in a single file: `index.html`. There are no imports, modules, or external scripts. The only external resources are Google Fonts (loaded via `@import`) and `Pixel Peeker Polka - slower.mp3` (not currently wired to the game).

The file is structured in this order:
1. `<style>` — all CSS, ~900 lines
2. `<body>` HTML — two main page divs (`#gameContent`, `#storePage`) plus overlays
3. `<script>` — all JavaScript, ~1100 lines

### Pages

Two "pages" are shown/hidden via `display: none/flex`. Never both visible at once.

- **`#gameContent`** — the active game: question card, status badges, multiplication toggle, mini cake-progress bar, "My Candy Store" button
- **`#storePage`** — the 1920s Willy Wonka Sweet Shoppe; acts as the score/collection screen. Contains the SVG store scene (walls, clerk, register), candy jars on a counter, cotton candy corner, lollipop bouquet, and premium display case.

Navigation: `showStore()` / `showGame()`.

### Canvas Layers

Two fixed canvases, both `pointer-events: none`:

| Canvas | z-index | Purpose |
|--------|---------|---------|
| `#bgCanvas` | 0 | Animated gradient background + ambient floating candy particles (`CandyParticle` class). Always running. |
| `#fxCanvas` | 50 | Burst particle effects on correct answers (`BurstParticle` class). Self-terminating loop — only runs while particles are alive. |

The `bgCanvas` gradient is driven by `BG_STOPS` — an array of RGB triples that is **mutated in-place** by `applyWorldTheme()` when world tier changes.

### Game State

Single `state` object:
```js
state = {
  streak, totalCorrect, questionNum, answering,
  multiplyEnabled,
  collection: { gummy, skittle, mms, choc, sundae, marshmallow, cotton, sandwich, lollipop, cake },
  pendingOverlay,  // set before showing a candy reward popup
  currentAnswer,
}
```

### Reward System

Two reward triggers fire independently after a correct answer:

- **Streak rewards** (`STREAK_REWARDS`): streak 1–9 each map to a candy key. At streak 9 the player earns a lollipop. Streaks above 9 repeat the lollipop reward.
- **Cake reward** (`CAKE_REWARD`): fires every 10 total correct answers. Triggers Sugar Crush animation first, then the birthday cake overlay.

Only one overlay is shown at a time via `state.pendingOverlay`. If both a streak reward and a cake reward are earned on the same answer, the cake takes priority (Sugar Crush fires, then cake overlay).

### World Theme System

Three themes keyed by `totalCorrect` thresholds:

| Tier | Name | Threshold | Particles |
|------|------|-----------|-----------|
| 0 | Lollipop World | 0–9 | Lollipop shapes |
| 1 | Cake World | 10–24 | Gold stars |
| 2 | Sundae World | 25+ | Cherry shapes |

`applyWorldTheme(force)` applies CSS variables to `:root`, swaps `BG_STOPS`, updates `<meta name="theme-color">`, and updates the stars bar. It is called on every `renderQuestion()` and after every correct answer. It is a no-op if the tier hasn't changed (unless `force=true`).

CSS variables controlled: `--theme-bg`, `--theme-card-bg`, `--theme-card-dot`, `--theme-q-color`, `--theme-badge-bg`, `--theme-badge-border`, `--theme-h1-shadow`.

### Difficulty Scaling

`getTier()` returns 0–3 based on **current streak** (not total correct):
- 0: streak 0–2 → Easy
- 1: streak 3–5 → Medium
- 2: streak 6–9 → Hard
- 3: streak 10+ → Expert

This is separate from the world tier. It controls answer-number ranges in `generateQuestion()`.

### Store Page Rendering

`renderStore()` is called every time the store is opened. It fully rebuilds:
- Jar SVGs (fill height computed from `Math.min(count, 10) / 10 * 83px`)
- Cotton candy icon (scaled 0.5×–1.5× based on count 0–15)
- Lollipop bouquet (fan arc −40° to +40°, up to 8 lollipops, then overflow badge)
- Display case shelves (up to 8 icons per shelf via `renderShelf()`)

### SVG Gradient ID Namespacing

The `CANDY_ICONS` object holds inline SVG strings for each candy used in the reward overlay (80×80). These use `ov-` prefixed gradient IDs (e.g. `ov-gb`, `ov-sk`). The game-page hidden count elements use different IDs. The store jar SVGs use `jclip-{key}` for clip paths. The store scene SVG uses `sg-` prefixed gradient IDs. Keep these namespaces distinct to avoid cross-SVG gradient conflicts.

### Key Element IDs

Game page: `streakCount`, `correctCount`, `questionNum`, `diffBadge`, `diffLabel`, `questionText`, `choices`, `feedback`, `streakFire`, `starsBar`, `cakeBar`, `cakeProgressLabel`, `multiplyTrack`, `multiplyLabel`.

Hidden candy counts (preserved for `pulseCandy()`): `cnt-gummy`, `cnt-skittle`, `cnt-mms`, `cnt-choc`, `cnt-sundae`, `cnt-marshmallow`, `cnt-cotton`, `cnt-sandwich`, `cnt-lollipop`, `cnt-cake`.

Store page: `jarRow`, `cottonIconWrap`, `cottonStoreBadge`, `bouquetContainer`, `shelf-cake`, `shelf-sundae`, `shelf-sandwich`, `clerkBubble`.

### Fonts

- **Fredoka One** — headings, numbers, labels, buttons
- **Nunito** (400/700/900) — body text, store labels, shelf labels
