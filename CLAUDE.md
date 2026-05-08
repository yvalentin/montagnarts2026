# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-page programme website for the **Les Montagnarts 2026** festival (22–24 May 2026, Valbonnais, France). Deployed via GitHub Pages from `main` at https://yvalentin.github.io/montagnarts2026/.

## Build / run / test

There is **no build system, no package manager, no tests, no linter**. The entire site is one static file: `index.html` (HTML + embedded CSS + embedded vanilla JS, ~1500 lines).

To preview locally:
```
python3 -m http.server 8765
# open http://localhost:8765/index.html
```

To deploy: commit to `main`, push — GitHub Pages serves it directly.

## Architecture

### One file, two layers

`index.html` contains both the **content** (cards in HTML) and the **schedule data** (JS array). These two sources must be kept in sync — see "Adding/editing a show" below.

### Show vs session model

This is the central concept of the codebase:

- A **show** = one HTML `.card[data-show-id="…"]` describing the spectacle (title, description, tags, links). Authored once.
- A **session** = one entry in the `SESSIONS` array (line ~947) representing a specific performance: `{ id, showId, day, start, duration, venue, reserveUrl? }`. A show can have many sessions across days/times.

At init, `distributeCardsByDay()` clones the original card into each day-group where the show has sessions, tagging it with `data-render-day`. Then `renderAllChips()` injects time-chips (one per session, scoped to that day) into the card's `.card-time` slot. Clicking a chip toggles the session in the user's planning (persisted in `localStorage` under `montagnarts-planning-v1`).

### Two views, same data

The sticky filter bar has a `Cartes / Calendrier` toggle (`setView()`, line ~1083):

- **Cards view** (`#main-cards`): the original day-grouped grid.
- **Gantt view** (`#gantt-view`, built by `buildGantt()` at line ~1110): one row per day, blocks positioned absolutely on a 9h–02h time axis (range `GANTT_START_MIN`..`GANTT_END_MIN`). Overlapping sessions get sub-rows via a greedy algorithm. Each block clones the card template and injects a single chip for its session, so all card interactions (chip click, links) work identically — no duplicate event handlers needed.

`applyFilters()` updates both views in one pass, hiding cards/blocks/empty rows by `data-types` and the active day.

### Planning panel

The bottom-right FAB opens an overlay (`renderPlanning()`, line ~1310) that lists selected sessions grouped by day, detects time overlaps (`.planning-conflict-warn`), and offers:
- Share via URL (`?p=id1,id2,…` consumed by `loadFromURL()`)
- Export `.ics` for calendar import
- Print (special `@media print` rules expand the planning panel to fill the page and hide everything else)

### State

All state lives in three module-level variables: `currentDay`, `currentType`, `currentView`, `selected: Set<sessionId>`. `selected` is the only persisted state (`saveToStorage()` / `loadFromStorage()`).

## Adding/editing a show

Two edits are required, in the same commit:

1. **HTML card** in the right `<div class="day-group" id="grp-{day}">` (use `data-show-id`, `data-days="ven sam dim"`, `data-types="theatre cirque …"`). The first day-group it lives in is the template; `distributeCardsByDay()` will clone it to other days.
2. **Session entries** in `SESSIONS`, one per performance: unique `id`, matching `showId`, `day` ∈ `{ven, sam, dim}`, `start: 'HH:MM'`, `duration` in minutes, `venue`, optional `reserveUrl` (use the `RES` prefix constant for billetterie URLs).

The existing `data-types` vocabulary is fixed by the type filter buttons and the `.tag-*` CSS classes: `theatre`, `cirque`, `danse`, `musique`, `conte`, `rue`, `free`. Adding a new type means also adding a filter button, a `.tag-*` style, and a Gantt border-color rule (`.gantt-block[data-types*="…"] .card.in-gantt`).

## Conventions

- All UI text is in **French** — keep new strings consistent with the existing tone (e.g. "Mon planning", "Aucun spectacle pour cette sélection").
- Late-night shows that cross midnight are supported because the Gantt internal range extends to 02:00 (`GANTT_END_MIN = 26 * 60`); only labels go up to 01h.
- Time helpers: `timeToMin('HH:MM')` is the canonical parser; `chipLabel()` renders `HH:MM` as `HHhMM`.
