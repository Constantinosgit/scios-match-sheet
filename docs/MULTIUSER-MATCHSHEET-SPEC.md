# SCIOS Match Sheet — Multi-User (Hosted) Spec

**The shift:** the current tool is single-device (data in the browser). This spec turns it into a **shared, multi-user system**: Marios sends each coach a link, the coach fills only their team on their phone, submits, it **locks**, and Marios is **notified** and sees everything in one dashboard. Only admins add teams/players and can edit a locked sheet.

**Stack (decided):** Supabase (Postgres DB + auto REST/Realtime APIs + Auth + Edge Functions for email). Coach access = **magic link per match** (no login). Hosted; reuses the existing match-sheet UI for the fill screen.

---

## 1. Roles

| Role | Can do |
|---|---|
| **Admin** (Marios + chosen staff) | Everything: add/edit teams & players, create the weekend's matches, generate coach links, view all lineups, **unlock/edit** any submitted sheet, get notifications. |
| **Coach** (external, via magic link) | Open their match link → fill **only their team's** lineup → submit. After submit it is **locked** to them. No login, no access to other teams/matches. |
| **Viewer** (optional) | Read-only dashboard of submitted lineups. |

Admin auth = Supabase email login. Coach "auth" = the unguessable token in their magic link (no account needed).

---

## 2. The weekend workflow (the whole point)

```
ADMIN (Marios), once per weekend:
  1. Open admin dashboard → pick the weekend's fixtures (from imported fixtures)
  2. For each match, the system creates two coach links (home + away)
  3. Marios copies/sends each coach their link via WhatsApp
     e.g. "AEK U19 — fill your lineup: https://matchsheet.scienceofsports.net/fill/x7k2..."

COACH (on phone or desktop, no login):
  4. Opens the link → sees ONLY his team + the squad
  5. Taps his XI, picks formation, adds subs/numbers (same UI as now)
  6. Presses SUBMIT → confirmation → the sheet is now LOCKED for him

SYSTEM:
  7. On submit → Marios is notified ("AEK U19 lineup submitted by <coach>")
  8. The lineup appears in Marios's dashboard, marked submitted/locked

ADMIN:
  9. Marios sees all matches in one view: which are pending / submitted / locked
  10. Only Marios can UNLOCK + edit a sheet (audit who/when)
  11. Export PDF / appearances data per match as today
```

---

## 3. Data model (Supabase / Postgres)

```
teams        (id, name, league, gender, age_group)         -- admin-managed
players      (id, scisports_id, name, team_id, default_number)
matches      (id, date, competition, home_team_id, away_team_id,
              home_score, away_score, created_by, created_at)
lineup_slots (id, match_id, team_id, side 'home'|'away',
              formation, status 'pending'|'submitted'|'locked',
              fill_token UNIQUE,          -- the magic-link token
              submitted_by, submitted_at, locked, updated_by, updated_at)
lineup_players (id, lineup_slot_id, player_id, shirt_number,
                position_code, role 'starter'|'sub', xi_order,
                sub_on_minute, sub_off_minute)
profiles     (id -> auth.users, role 'admin'|'viewer')      -- admins only
```

**Locking:** `lineup_slots.status='locked'` after submit. Coach token can read but not write a locked slot. Admin (authenticated, role=admin) can set status back and edit — every edit stamps `updated_by/updated_at` for audit.

**Row-Level Security (RLS):** the key safety layer —
- A coach (anon, holding a `fill_token`) can only read/write the **one** `lineup_slots` row matching that token, and only while `status != 'locked'`.
- Admins (authenticated, role=admin) can do everything.
- No one else can read anything. This is what makes "external users can fill but not edit others" actually enforced, not just hidden in the UI.

---

## 4. Screens

1. **Coach fill page** `/fill/:token` — reuses the current match-sheet UI, but scoped to one team, read-only on opponent, with a big **Submit** button. After submit → "Lineup submitted ✓ (locked)". Works phone + desktop.
2. **Admin dashboard** `/admin` — table of the weekend's matches: each row shows Home/Away, status chips (Pending/Submitted/Locked), buttons: copy link, view, unlock, export PDF. A "New weekend" action to bulk-create matches from fixtures.
3. **Admin team/player editor** — manage teams & rosters (admin-only), as today but server-stored.

---

## 5. Notifications (on submit)

Supabase **Edge Function** triggered on a `lineup_slots` row turning `submitted`:
- **Email** to Marios (cheapest, reliable) — "AEK U19 lineup submitted for 27/09/2025".
- **WhatsApp** (optional, later) via a provider (Twilio/360dialog) — more setup/cost.

Recommendation: **start with email**, add WhatsApp later if needed.

---

## 6. What carries over from the current tool

- The whole **match-sheet UI** (tap-to-pick XI, formations + position codes, pitch, subs, PDF export, DD/MM/YYYY) is reused for the coach fill page — we don't rebuild it.
- The **players/fixtures data** (CSV import) seeds the Supabase `teams`/`players` tables once.
- The **appearances** structure (for season position tracking) is produced server-side from `lineup_players`, so the year-end report becomes a simple query.

---

## 7. Honest scope & sequencing

This is a real project, not an edit. Suggested phases:

- **Phase 1 — Foundation:** Supabase project, schema, RLS, seed teams/players. Admin login.
- **Phase 2 — Coach fill:** magic-link fill page + submit + lock. (The core value.)
- **Phase 3 — Admin dashboard:** create weekend matches, generate links, status view, unlock/edit.
- **Phase 4 — Notifications:** email on submit (WhatsApp later).
- **Phase 5 — Reports:** season position report from `lineup_players`.

**Cost:** Supabase free tier covers this scale (a few hundred lineups/season). Email via a free/cheap provider.

**Decision needed from Constantinos before Phase 1:**
1. Create the Supabase project (free account) — who owns it (SCIOS account)?
2. Admin emails (who, besides Marios, is admin)?
3. Confirm email-first notifications (WhatsApp later)?
4. This lives at `matchsheet.scienceofsports.net` (replacing the static tool) or a new subdomain?
