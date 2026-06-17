# Player Position Tracking — Data Model & Spec

**Goal:** capture, for every player, **which formation-specific position he played in each match**, across all teams/age groups he appears for during the season — so at year-end we can report e.g. *"Charalambous: RCD ×8, CCD ×3, RB ×2"*, unified across APOEL U17 / U19 / Cyprus U17, with shirt-number changes tracked per match.

Status: spec / blueprint. The standalone Match Sheet tool captures this data and exports it; the SCIOS platform is the intended permanent home (a `match_lineups` table).

---

## 1. The core principle: three linked things

Track **three separate concepts**, linked by a stable player ID. Do **not** store position on the player record (it changes match to match).

### a) Player (the person — one record per human)
Keyed by the **SciSports `Id`** (e.g. `754608`). VERIFIED: the same Id is used for the same person across teams/age groups (2,577 unique Ids; 55 players confirmed appearing in >1 team with the same Id). This Id is the spine that unifies a player across APOEL U17, APOEL U19, and Cyprus U17.

```
player_id (SciSports Id) · name · dob? · nationality?
```

### b) Appearance / Lineup entry (one row per player, per match) — THE KEY TABLE
Captured every time a match sheet is built and saved.

```
player_id        754608          → links to the person (unifies across teams)
match_id         <fixture id>    → which match
match_date       13/09/2025
team_name        "APOEL U17"     → who he played for that day (team, not person)
competition      "Cyprus U17 League Division 1"
shirt_number     4               → his number IN THIS MATCH (captured per appearance)
formation        "1-3-4-3"       → the team shape that match
position_code    "RCD"           → formation-specific slot (see §2)
role             starter | sub
sub_on_minute    63 | null
sub_off_minute   null | 70
```

### c) Formation position vocabulary (reference)
Defines the slot names per formation so `position_code` is consistent (§2).

### Why this works
Every appearance carries player_id + position_code + team + formation + number. Year-end report = filter all appearances for a player_id and count by position_code. Teams unify automatically (shared player_id); shirt-number changes are natural (per-appearance field).

---

## 2. Position vocabulary per formation (APPROVED — Constantinos, 2026-06-17)

Convention: **left → right as the player faces the opponent's goal** (mirrored — LB on the left of the diagram), **defence → attack**. Slot order matches the Match Sheet pitch (index 0 = GK). These are locked in the tool.

| # | Formation | Position codes (GK first, defence → attack) |
|---|---|---|
| 1 | **1-4-4-2** | GK · LB LCB RCB RB · LW LDM RDM RW · LS RS |
| 2 | **1-4-3-3** | GK · LB LCB RCB RB · LCM CDM RCM · LW ST RW |
| 3 | **1-4-2-3-1** | GK · LB LCB RCB RB · LDM RDM · LW AM RW · ST |
| 4 | **1-4-1-4-1** | GK · LB LCB RCB RB · CDM · LW LCM RCM RW · ST |
| 5 | **1-3-5-2 (DM)** | GK · LCD CCD RCD · LWB LCM DM RCM RWB · LS RS |
| 6 | **1-3-5-2 (AM)** | GK · LCD CCD RCD · LWB LCM RCM RWB · AM · LS RS |
| 7 | **1-3-4-3** | GK · LCD CCD RCD · LWB LDM RDM RWB · LW ST RW |
| 8 | **1-5-3-2** | GK · LWB LCB CCB RCB RWB · LCM DMF RCM · LS RS |
| 9 | **1-4-5-1** | GK · LB LCB RCB RB · LW LCM DMF RCM RW · ST |

**Code key:**
- Defence: `LB`/`RB` full-backs · `LWB`/`RWB` wing-backs · `LCB`/`CCB`/`RCB` centre-backs (back-4/5) · `LCD`/`CCD`/`RCD` centre-defenders (back-3).
- Midfield: `CDM` central defensive mid · `DM`/`DMF` single defensive mid · `LDM`/`RDM` paired defensive mids · `LCM`/`RCM` central mids · `AM` attacking mid · `LW`/`RW` wide (used for wingers and wide-mid roles).
- Attack: `ST` striker · `LS`/`RS` strike pair.

Notes per Constantinos: two **1-3-5-2** variants exist — `(DM)` with the single mid BEHIND the two centre-mids, `(AM)` with the playmaker AHEAD of them. `DM` (in 3-5-2) and `DMF` (in 5-3-2 and 4-5-1) are kept as separate codes intentionally.

---

## 3. How position is captured (auto from the pitch)

When building a sheet, each starter occupies a **pitch slot** in the chosen formation. That slot's `position_code` (from §2) is recorded automatically. Tap-to-swap lets the user fix placements before saving. So no extra data entry — the position falls out of where the player stands on the pitch.

Subs: a substitute's position can be (a) left blank, (b) set to the slot of the player he replaced, or (c) tagged manually. **Recommendation:** record subs with `role=sub` and no position unless the user sets one (most subs don't have a fixed slot).

---

## 4. SCIOS platform implementation (the permanent home)

A new table, mirroring §1b:

```
platform.match_lineups
  id              uuid pk
  player_id       uuid  → platform.players(id)   (resolved from SciSports Id)
  match_id        uuid  → platform.cfa_fixtures / scisports match
  team_id         uuid  → platform.teams(id)
  shirt_number    int
  formation       varchar(12)
  position_code   varchar(8)   null for subs without a slot
  role            varchar(8)   'starter' | 'sub'
  sub_on_minute   int null
  sub_off_minute  int null
  created_at      timestamptz
  source          varchar(24)  'match-sheet'
  INDEX (player_id), (match_id), (team_id)
```

- **Ingest endpoint:** `POST /v1/match-lineups` accepting the appearances JSON the tool exports (§5).
- **Player resolution:** match the tool's SciSports `Id` → `platform.players` (the bridge already exists via scisports player mapping).
- **Year-end report:** `GET /v1/players/:id/positions?season=…` → counts grouped by `position_code` and `team`, plus shirt numbers used. Powers the end-of-season player profile.

This is real backend work (migration + endpoint + report query) — a defined project, not a quick edit.

---

## 5. Tool export format (feeds the platform)

On Save/Export, the Match Sheet emits an **appearances** array — one object per player per match — ready to POST into `match_lineups`:

```json
{
  "match": { "date": "13/09/2025", "competition": "Cyprus U17 League Division 1",
             "home_team": "APOEL U17", "away_team": "AEL U17",
             "formation_home": "1-3-4-3", "formation_away": "1-4-4-2",
             "home_score": 2, "away_score": 1 },
  "appearances": [
    { "player_id": "754608", "name": "Charalambous", "team": "APOEL U17",
      "shirt_number": 4, "formation": "1-3-4-3", "position_code": "RCD",
      "role": "starter", "sub_on_minute": null, "sub_off_minute": 70 },
    { "player_id": "754999", "name": "Andreou", "team": "APOEL U17",
      "shirt_number": 17, "formation": "1-3-4-3", "position_code": null,
      "role": "sub", "sub_on_minute": 70, "sub_off_minute": null }
  ]
}
```

The `player_id` is the SciSports Id (already stored as `srcId` in the tool). This is the join key the platform uses to unify a player across all his teams.

---

## 6. Open decisions for Constantinos
1. **CB naming in back-3/5:** `RCB/CCB/LCB` vs `RCD/CCD/LCD` — pick one.
2. **Any formations to add/remove** beyond the 8 above?
3. **Subs position:** leave blank, inherit replaced player's slot, or manual? (Recommended: blank unless set.)
4. Confirm the per-formation code lists in §2 (rename any you'd say differently).
