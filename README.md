# Diabetic Records Tracker

A personal health records dashboard for managing diabetic and medical data, with support for tracking a second person (spouse/family member). Hosted on GitHub Pages and powered by Firebase Firestore with localStorage fallback.

## Features

- **Daily Readings** — Log glucose levels, blood pressure, weight, meals, and medications
- **Lab Tests** — Track test sessions with multiple criteria results
- **Charts & Trends** — Visualize glucose, BP, and weight by day, month, or year
- **Diet Analysis** — Rank meals by glucose impact with best/worst lists
- **Spouse Tracking** — Separate treatment periods and visit records
- **Data Export** — CSV and full Excel workbook export; JSON backup/restore
- **PIN Protection** — SHA-256 hashed 6-digit PIN with brute-force lockout

## Storage

Uses Firebase Firestore as primary storage, with automatic fallback to browser localStorage. All data stays private — no accounts, no third-party access beyond Firebase.

## Usage

Open `index.html` directly in a browser or visit the GitHub Pages URL. On first use, set a PIN and configure your glucose/BP thresholds in **Settings**.

## Architecture

See [Diabetic_Architecture_Diagram.html](./Diabetic_Architecture_Diagram.html) for a full visual diagram of how GitHub Pages, Firebase Firestore, localStorage, and the PIN security model work together.
