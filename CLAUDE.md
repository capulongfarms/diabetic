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

All application code is in `index.html` — styles, markup, and a large `<script type="module">` block. Firebase, Chart.js, and XLSX are loaded via CDN.

### Core data model

The in-memory `_data` object is the single source of truth:

```js
_data = {
  readings: [],  // daily glucose, BP, weight, meals, medications
  tests: [],     // lab test sessions with per-criterion results
  wifesum: [],   // spouse treatment periods
  wifev: []      // spouse visit/appointment records
}
```

Each array maps to a Firestore sub-collection under `users/{FIXED_UID}/{name}`. The fixed UID (`64oGBkwTKJOfCpGUW10eTrLdMgJ2`) means this is a single-user app.

### Storage layer

Four database primitives handle all persistence:

| Function | Purpose |
|---|---|
| `dbAll(name)` | Fetch all docs from a collection |
| `dbAdd(name, data)` | Add a new doc |
| `dbSet(name, id, data)` | Update an existing doc |
| `dbDel(name, id)` | Delete a doc |

Each function tries Firebase first, falls back to localStorage automatically. Both paths use the same schema.

### Dashboard panels

Eight panels are toggled by showing/hiding `<section id="dash-*">` elements:

| Panel ID | Purpose |
|---|---|
| `dash-summary` | Overview stats + 30-day glucose trend chart |
| `dash-readings` | Daily records CRUD table (50/page, searchable) |
| `dash-charts` | Chart.js visualizations (glucose, BP, weight by day/month/year) |
| `dash-diet` | Meal impact analysis and rankings |
| `dash-bp` | Blood pressure analysis |
| `dash-tests` | Lab results — split-pane session list + per-criterion results |
| `dash-wife` | Spouse tracking — treatment periods and visit records |
| `dash-settings` | Preferences, PIN management, Firebase config, data backup/restore |

### Security model

- **Authentication:** SHA-256 hashed 6-digit PIN stored in localStorage.
- **Session token:** Stored in `sessionStorage` — survives page refresh, clears on tab close.
- **Brute-force protection:** 3 failed attempts → 30-second lockout.
- **Guarded operations:** All write operations are wrapped in `_guardedAsync()`, which re-prompts for PIN before executing.

### Preferences

Stored in `localStorage` under the key `prefs`. Contains user/spouse names, glucose thresholds (mmol/L), and BP thresholds (systolic/diastolic). These drive chart coloring and status indicators throughout the app.

## Key Conventions

- **No framework** — UI updates are direct DOM manipulation (`innerHTML`, `textContent`, `classList`).
- **Modal pattern** — All forms use a single `<dialog>` element reused across operations; content is rewritten on each open.
- **Toast notifications** — Use `showToast(message, type)` for all user feedback (`type` is `'success'`, `'error'`, or `'info'`).
- **Firebase config** is hardcoded in the script; this is intentional for this private single-user app.
- **Data files** in `/Files/` are manual backups and exports — not used at runtime.
