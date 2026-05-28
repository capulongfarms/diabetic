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

`init()` runs on `window.load` in this order:
1. Checks `navigator.onLine` — toasts an error if offline; registers `online`/`offline` event listeners.
2. Initializes Firebase (`initializeApp` + `getFirestore`) — sets `_db`. These are synchronous and always succeed.
3. Hides the auth screen, shows the app shell immediately (no login gate).
4. Calls `loadAllData()` — fetches all 4 collections in parallel via `dbAll()`. Firebase is always tried first; if `getDocs` throws (offline / rules error), `dbAll` falls back to localStorage and toasts a warning. The `info-storage` status indicator is updated accordingly.
5. Calls `renderSummary()`.
6. Calls `_applySecurityGuards()` — wraps all `window.*` write functions with `_guardedAsync()`. Guards are applied here (end of init) to eliminate any race window.

### Core data model

The in-memory `_data` object is the single source of truth:

```js
_data = { readings:[], tests:[], wifesum:[], wifev:[] }
_sel  = { reading:null, test:null, wifesum:null, wifev:null }  // selected row IDs
```

Each array maps to a Firestore sub-collection under `users/{FIXED_UID}/{name}`. The fixed UID (`64oGBkwTKJOfCpGUW10eTrLdMgJ2`) means this is a single-user app.

**Other global state:**

| Variable | Default | Purpose |
|---|---|---|
| `_readPage` | `1` | Current page in the readings table |
| `PAGE_SIZE` | `50` | Rows per page (constant) |
| `_sortField` | `'date'` | Active sort column |
| `_sortAsc` | `false` | Sort direction (false = newest-first) |
| `_charts` | `{}` | Live Chart.js instances keyed by canvas ID; each is `.destroy()`ed before re-render |

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

| Function | Behavior |
|---|---|
| `dbAll(name)` | Tries Firebase first. On success, **writes result to localStorage** (keeps cache fresh). On failure, falls back to localStorage and shows a warning toast. |
| `dbAdd(name, data)` | Always writes to localStorage first. Then tries Firebase; on failure, toasts user and returns the local ID — data is never lost. |
| `dbSet(name, id, data)` | Always writes to localStorage first. Then tries Firebase `setDoc`; toasts on failure. |
| `dbDel(name, id)` | Always removes from localStorage first. Then tries Firebase `deleteDoc`; toasts on failure. |

This is a **write-through cache** pattern: localStorage is always kept in sync with Firebase, not used as a last resort. The result is that:
- Going offline after a load shows the **current** Firebase state, not stale old data.
- No write is ever silently lost — if Firebase is unavailable, the record lands in localStorage.
- All four collections always have a warm local cache for instant offline access.

localStorage keys follow the pattern `data_{FIXED_UID}_{name}`. `_fbWarnedOffline` (module-level boolean) ensures the offline toast and storage indicator update fire only once per session even though `dbAll` is called 4 times in parallel.

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
- **Brute-force protection:** 5 failed attempts → 60-second lockout; lockout state in `sec_lock`, attempt counter in `sec_attempts` (both localStorage).
- **Perpetual PIN mode:** `SESSION_TTL = 0`, so every guarded action requires a PIN re-entry regardless of recency. Session token is written to `sec_sess` in sessionStorage but expires immediately.
- **Guarded operations:** Most write operations are wrapped in `_guardedAsync()`, which pops the `modal-pin-prompt` dialog and resolves only on correct PIN. Guards are applied once at the end of `init()` via `_applySecurityGuards()`. **Exception — modal Save buttons are never guarded:** the Save button inside any modal (both Add and Edit mode) calls the unguarded `save*()` function directly (e.g. `saveReading`, `saveTest`). For Add, no PIN is needed at all. For Edit, the PIN is collected once at the Edit button click (`editSelectedReading` etc., which are guarded) — the Save button does not prompt again.
- **No-PIN-set behavior:** If `sec_pin_h` is missing (first use or localStorage cleared), only the "Change Admin PIN" action is allowed through. All other writes are blocked and the user is redirected to Settings. This closes the PIN-deletion bypass.
- **`_pendingActionLabel`:** Stored alongside `_pendingAction` in `_showPinPrompt` so `submitPinPrompt` can identify which action is pending and apply the no-PIN-set rule correctly.

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

### Firestore Security Rules

Defined in `firestore.rules` (deployed via Firebase Console). Locks Firestore access to the single fixed-UID path; all other paths return `permission-denied`:

```js
match /users/64oGBkwTKJOfCpGUW10eTrLdMgJ2/{document=**} { allow read, write: if true; }
match /{document=**} { allow read, write: if false; }
```

`firebase.json` points to `firestore.rules` for CLI deployment if needed (`firebase deploy --only firestore:rules --project diabetic-records`).

### Preferences

Stored in `localStorage` under the key `prefs`. Contains user/spouse names, glucose thresholds (mmol/L), and BP thresholds (systolic/diastolic). These drive chart coloring and status indicators throughout the app.

## Key Conventions

- **No framework** — UI updates are direct DOM manipulation (`innerHTML`, `textContent`, `classList`). `$ = id => document.getElementById(id)` is the only selector shorthand.
- **`window.*` exposure** — render and action functions are attached to `window` so inline HTML `onclick` handlers can reach them (e.g., `window.saveReading`, `window.renderSummary`).
- **Toast notifications** — `toast(message, type)` for all user feedback (`type` is `'success'`, `'error'`, or `'info'`).
- **Button debounce** — save buttons are disabled immediately on click and re-enabled in a `finally` block to prevent double-submits.
- **Firebase config** is hardcoded in the script; this is intentional for this private single-user app.
- **Excel export** — `exportAllExcel()` writes all four collections to a single `.xlsx` file with separate sheets via SheetJS, compatible with the companion Python app format.
- **Data files** in `/Files/` are manual backups and exports — not used at runtime.
