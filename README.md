# SCIOS Match Sheet Builder

A standalone, single-file web tool to build football match sheets — pick a fixture, set both teams' starting XIs in a formation, record subs, export a PDF or share to WhatsApp, and export the lineup as JSON for the SCIOS platform.

**One file, no build, no dependencies.** Everything lives in `index.html`. Open it by double-clicking, or host it at a URL — both work identically. All data is stored in the browser (localStorage), so it works fully offline.

## Exporting a clean PDF

The browser (not this tool) adds a date/URL header & footer to printouts — it **cannot** be removed by code, only via the print dialog. To get a clean sheet, in the print window:
- Open **"More settings"**
- **Uncheck "Headers and footers"** → removes the date/URL at the top & bottom
- **Check "Background graphics"** → makes the pitch print green

The browser remembers both settings for next time. (The tool shows a one-time reminder of this when you first export.)

## Using it

1. **Teams & Rosters tab** (admin only — see below):
   - **Import players (CSV)** — export the Cyprus Operation Sheet "Corrected players" tab to CSV, import it. Groups players by `TeamName`, uses the `Correct name` where present. You can **select several CSVs at once** (one per spreadsheet tab) — they're parsed and merged, with a single Replace/Merge choice applied to the whole batch.
   - **Import fixtures (CSV)** — export the "Fixture Import" tab to CSV, import it (1,080 fixtures).
2. **Match Sheet tab**:
   - **Load a fixture**: Type (🏟️ Club / 🌍 National) → Men/Women → age group → round → match. Auto-fills both teams + date.
   - **Tap players** in the Squad to add them to the Starting XI (tap again to remove). Pick a formation; players snap onto the pitch.
   - Add **substitutions** (minute + off/on), set the **score**.
   - Use the **Home / Away** toggle to fill one team at a time (both print to PDF).
3. **Export / share** (header buttons):
   - **Export PDF** — print-to-PDF; the file is named `Home_vs_Away_DD-MM-YYYY.pdf`, ready for WhatsApp.
   - **📲 Share** — on mobile, opens the native share sheet (WhatsApp/email); on desktop, copies a text summary.
   - **↗ SCIOS** — downloads the match as JSON for the platform (`source:scios-match-sheet`, `version:2`). Contains:
     - `meta` — `match_key` (deterministic: competition|date|home|away — see note), date, scores, competition, referee.
     - `home_team`/`away_team` — `formation`, `coach`, `starters[]`/`subs[]` (each with `player_id` [SciSports/source id], `shirt_number`, `position_abbr`, `subbed_on_minute`, `subbed_off_minute`, `minutes_played`), plus per-team `goals[]` (`player_id`, `minute`, `type`: goal/penalty/own_goal) and `cards[]` (`player_id`, `minute`, `type`: yellow/red).
     - `appearances[]` — one flat row per player per match (per PLAYER-POSITION-TRACKING-SPEC §5) with `position_code`, `role`, sub minutes, `minutes_played`, and `goals`/`yellow`/`red` tallies — ready for season player/team stats.
     - **Open item for the platform team:** imported fixtures carry no unique id, so `match_key` is a string key, not the platform's fixture id. Confirm whether the source fixtures expose an id to import and carry through.

## Admin gate

Importing players & fixtures and "Clear all" are **admin-only**. Tap **🔒 Admin** in the Teams tab and enter the passcode (set in `index.html` → `ADMIN_CODE`, currently `scios2026`). Anyone can build, export, and share sheets without the passcode.

## Other features

- **Auto-save**: the in-progress sheet is saved on every change and restored when the tool reopens — work is never lost.
- **Saved sheets**: press Save to keep a sheet under the Saved tab (by teams + date).
- **Light / Dark mode**: 🌙/☀️ toggle, remembered per device. Uses the SCIOS platform palette.
- **DD/MM/YYYY** dates throughout (EU locale).

## Deploying to a SCIOS URL

Because it's a single static file, hosting is trivial — any static host works:

- **Quick**: drop `index.html` into any web root (Nginx/Apache), an S3/Cloudflare bucket with static hosting, Netlify/Vercel drag-and-drop, or a Google Sites/Drive share.
- **Suggested**: serve at e.g. `matchsheet.scienceofsports.net` → point it at this file. HTTPS is required for the 📲 Share (Web Share API) and clipboard fallback to work on mobile.
- No server code, no environment variables, no database — the browser holds all state.

## Data source

- Players: Cyprus Operation Sheet → "Corrected players" tab.
- Fixtures: Cyprus Operation Sheet → "Fixture Import" tab.
- The fixtures file's `Gender` column is unreliable (all "Male"); gender is derived from the competition name instead ("Women's…" → Women).

## Notes / future

- Shirt numbers import sequentially (the source sheet has no number column); edit per team or per sheet.
- Possible next steps: real player positions for richer SCIOS export, team crests on the PDF, a platform import endpoint to make the SCIOS hand-off fully automatic, captain marker, coach/referee fields.
