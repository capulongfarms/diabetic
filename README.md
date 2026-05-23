# Diabetic Records Tracker


A personal health records dashboard for managing diabetic and medical data, with support for tracking a second person (spouse/family member). Hosted on GitHub Pages and powered by Firebase Firestore with localStorage fallback.

## Features

- **Daily Readings** — Log glucose levels, blood pressure, weight, meals, and medications
- **Lab Tests** — Track test sessions with multiple criteria results
- **Charts & Trends** — Visualize glucose, BP, and weight by day, month, or year
- **Diet Analysis** — Rank meals by glucose impact with best/worst lists
- **Spouse Tracking** — Separate treatment periods and visit records
- **Data Export** — CSV and full Excel workbook export; JSON backup/restore
- **PIN Protection** — SHA-256 hashed PIN (min 4 digits) required on every write; 5 failed attempts triggers a 60-second lockout
- **Offline Support** — detects when offline on load and falls back to locally cached data automatically

## Storage

Uses Firebase Firestore as primary storage. If Firebase is unreachable (offline or network error), the app falls back to browser localStorage and shows a warning — data is never lost. Firestore Security Rules lock access to the single fixed-user path; all other paths are denied. All data stays private — no accounts, no third-party access beyond Firebase.

## Usage

Open the live app at **https://capulongfarms.github.io/diabetic/** or run `index.html` directly in a browser. On first use, set a PIN and configure your glucose/BP thresholds in **Settings**.

## Architecture

See [Diabetic_Architecture_Diagram.html](./Diabetic_Architecture_Diagram.html) for a full visual diagram of how GitHub Pages, Firebase Firestore, localStorage, and the PIN security model work together.
