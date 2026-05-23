# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file personal health records dashboard for a diabetic patient and spouse. Hosted on GitHub Pages at [capulongfarms/diabetic](https://github.com/capulongfarms/diabetic). Built with vanilla HTML/JS and Firebase Firestore (with localStorage fallback). No build system — everything lives in `index.html`.

## Running the App

Open `index.html` directly in a browser, or serve via a local HTTP server (needed for ES module imports to work reliably):

```powershell
python -m http.server 8080
# then open http://localhost:8080
```

There is no build step, no npm, and no compilation.

## Architecture

### Single-file app

All application code is in `index.html` — styles, markup, and a large `<script type="module">` block. CDN dependencies (pinned versions):
- Firebase JS SDK 10.12.2
- Chart.js 4.4.0
- SheetJS (XLSX) 0.18.5

### Startup flow

`init()` runs on `window.load`. It initializes Firebase, hides the auth screen immediately (no login gate), calls `loadAllData()`, then `renderSummary()`. The auth screen HTML remains in the DOM but is never shown in the current version — the PIN pad is only used for write-guarding, not login.

### Core data model

The in-memory `_data` object is the single source of truth:

```js
_data = { readings:[], tests:[], wifesum:[], wifev:[] }
_sel  = { reading:null, test:null, wifesum:null, wifev:null }  // selected row IDs
```

Each array maps to a Firestore sub-collection under `users/{FIXED_UID}/{name}`. The fixed UID (`64oGBkwTKJOfCpGUW10eTrLdMgJ2`) means this is a single-user app.

**Reading record schema** (fields saved by `saveReading`):

| Field | Type | Notes |
|---|---|---|
| `date` | string | YYYY-MM-DD, required |
| `dayOfWeek` | string | Auto-computed (Mon–Sun) |
| `weight` | number\|null | kg |
| `bpSys` / `bpDia` / `pulse` | number\|null | |
| `glucoseMgdl` / `glucoseMmol` | number\|null | mmol auto-computed from mg/dL ÷ 18.016 |
| `readingTime` | string\|null | HH:MM |
| `timeOfReading` | string | Morning / Afternoon / Evening / Bedtime |
| `mealType` | string | Breakfast / Lunch / Dinner / Others |
| `dinnerMeal` / `stressSign` / `exercise` / `remarks` | string\|null | |
| `medHypert` / `medDiab` | string\|null | Pipe-delimited medication list |

Duplicate detection: same `date + timeOfReading` combination triggers a `confirm()` dialog before saving.

### Storage layer

Four database primitives handle all persistence:

| Function | Purpose |
|---|---|
| `dbAll(name)` | Fetch all docs from a collection |
| `dbAdd(name, data)` | Add a new doc |
| `dbSet(name, id, data)` | Update an existing doc |
| `dbDel(name, id)` | Delete a doc |

Each function tries Firebase first, falls back to localStorage automatically. localStorage keys follow the pattern `data_{FIXED_UID}_{name}`.

### Dashboard panels

Eight panels are toggled by showing/hiding `<section id="dash-*">` elements. Lazy rendering: `renderCharts()`, `renderDiet()`, `renderBP()`, and `renderSummary()` are called on tab switch (not on load).

| Panel ID | Purpose |
|---|---|
| `dash-summary` | Overview stats + 30-day glucose trend chart |
| `dash-readings` | Daily records CRUD table (50/page, searchable, sortable) |
| `dash-charts` | Chart.js visualizations (glucose, BP, weight by day/month/year) |
| `dash-diet` | Meal impact analysis and rankings |
| `dash-bp` | Blood pressure analysis with AHA staging |
| `dash-tests` | Lab results — split-pane session list + per-criterion results |
| `dash-wife` | Spouse tracking — treatment periods and visit records |
| `dash-settings` | Preferences, PIN management, Firebase config, data backup/restore |

### Security model

Defined in the `_SEC` constants object:

- **PIN storage:** SHA-256(`pin + '_diabetic_salt'`) stored in localStorage key `sec_pin_h`.
- **Brute-force protection:** 5 failed attempts → 60-second lockout (stored in `sec_lock`).
- **Perpetual PIN mode:** `SESSION_TTL = 0`, so every guarded action requires a PIN re-entry regardless of recency.
- **Guarded operations:** All write operations are wrapped in `_guardedAsync()`, which pops the `modal-pin-prompt` dialog and resolves only on correct PIN.

### Modals

Six `<dialog>`-like overlay elements (CSS class `.modal-overlay`):

| ID | Purpose |
|---|---|
| `modal-reading` | Add / edit daily reading |
| `modal-test` | Add / edit lab test session |
| `modal-wifesum` | Add / edit spouse treatment period |
| `modal-wifevis` | Add / edit spouse visit record |
| `modal-changepin` | Change admin PIN |
| `modal-pin-prompt` | PIN re-entry guard for writes |

Open with `openModal(id)`, close with `closeModal(id)`.

### Preferences

Stored in `localStorage` under the key `prefs`. Contains user/spouse names, glucose thresholds (mmol/L), and BP thresholds (systolic/diastolic). These drive chart coloring and status indicators throughout the app.

## Key Conventions

- **No framework** — UI updates are direct DOM manipulation (`innerHTML`, `textContent`, `classList`). `$ = id => document.getElementById(id)` is the only selector shorthand.
- **`window.*` exposure** — render and action functions are attached to `window` so inline HTML `onclick` handlers can reach them (e.g., `window.saveReading`, `window.renderSummary`).
- **Toast notifications** — `toast(message, type)` for all user feedback (`type` is `'success'`, `'error'`, or `'info'`).
- **Button debounce** — save buttons are disabled immediately on click and re-enabled in a `finally` block to prevent double-submits.
- **Firebase config** is hardcoded in the script; this is intentional for this private single-user app.
- **Data files** in `/Files/` are manual backups and exports — not used at runtime.
