# ⚽ FPL Analytics — Power BI Report

A Fantasy Premier League analytics project that pulls live data from the **official FPL API**, processes it with Python, and visualises it in **Power BI**.

---

## 📁 Project Structure

```
FPL-Analysis/
│
├── notebooks/
│   ├── fpl_player_pull.ipynb       # Pulls player metadata → players.csv
│   ├── gameweek_pull.ipynb         # Pulls per-GW stats → gw{N}.csv
│   └── player_scores.ipynb         # Calculates composite scores → player_scores.csv
│
├── 25-26 gws upload/
│   ├── players.csv                 # Player metadata (static reference)
│   ├── gw1.csv                     # Gameweek 1 stats
│   ├── gw2.csv                     # Gameweek 2 stats
│   ├── ...                         # One file per gameweek
│   └── player_scores.csv           # Cumulative composite scores per GW
│
└── fpl_new.pbix                    # Power BI file
```

---

## 🔄 Data Pipeline

### Step 1 — `fpl_player_pull.ipynb` → `players.csv`

Calls the FPL `bootstrap-static/` endpoint and extracts a static player reference table.

**Output columns:**

| Column | Description |
|---|---|
| `id` | FPL internal player ID |
| `code` | Permanent player code (used as join key across all files) |
| `first_name` | Player first name |
| `second_name` | Player surname |
| `web_name` | Short display name |
| `photo` | Full URL to player headshot (PNG, 110×140) |
| `element_type` | Position — 1=GK, 2=DEF, 3=MID, 4=FWD |

> Run this once per season or when the squad changes significantly.

---

### Step 2 — `gameweek_pull.ipynb` → `gw{N}.csv`

Pulls live and historical stats for a single gameweek. Supports **double gameweeks** — it calls `element-summary/{id}/` for any player with multiple fixtures in the same GW, producing one row per fixture.

**Key API endpoints used:**
- `bootstrap-static/` — player and team metadata
- `fixtures/` — fixture list with scores and kickoff times
- `event/{gw}/live/` — live GW stats
- `element-summary/{id}/` — per-player historical data (double GW handling)

**Output columns (one row per player per fixture):**

| Column | Description |
|---|---|
| `player_id` / `code` | Player identifiers |
| `gameweek` | GW number |
| `fixture_id` | FPL fixture ID |
| `name` / `web_name` | Player names |
| `element_type` | Position |
| `team_id` / `opponent_id` | Team IDs |
| `h_a` | Home or Away |
| `kickoff_time` | Fixture kickoff timestamp |
| `team_score` / `opponent_score` | Match result |
| `minutes` | Minutes played |
| `goals_scored` / `assists` | Attacking returns |
| `clean_sheets` / `goals_conceded` | Defensive returns |
| `yellow_cards` / `red_cards` | Discipline |
| `bonus` / `bps` | Bonus points |
| `total_points` | FPL points for that fixture |
| `xG` / `xA` / `xGI` / `xGC` | Expected stats |
| `DefCon` | Defensive contribution |
| `ict_index` / `influence` / `creativity` / `threat` | ICT metrics |
| `in_dreamteam` | Dream Team selection flag |
| `status` / `news` | Availability and injury news |
| `corners_indirect_freekicks_order` / `direct_freekicks_order` / `penalties_order` | Set piece responsibilities |
| `selected_by_percent` | Ownership % |
| `value` | Player price (£) |

> Run once per gameweek after it has been completed. Enter the GW number when prompted (defaults to the current GW).

---

### Step 3 — `player_scores.ipynb` → `player_scores.csv`

Combines all `gw*.csv` files and calculates cumulative **percentile-based scores** for each player at each gameweek snapshot. Scores are calculated within position groups.

**Scoring methodology:**

| Score | Metric | Direction |
|---|---|---|
| `mins_score` | Cumulative minutes vs max possible | ↑ more = better |
| `goals_score` | Cumulative goals | ↑ |
| `assists_score` | Cumulative assists | ↑ |
| `xg_score` | Cumulative xG | ↑ |
| `xgi_score` | Cumulative xGI | ↑ |
| `cs_score` | Cumulative clean sheets | ↑ |
| `bonus_score` | Cumulative bonus points | ↑ |
| `goals_conceded_score` | Cumulative goals conceded | ↓ lower = better |
| `xgc_score` | Cumulative xGC | ↓ |

**Composite score weights:**

```
xGI score        25%
Goals score      20%
Assists score    10%
xG score         10%
Clean sheet score 10%
Minutes score    10%
Bonus score       5%
Goals conceded    5%
xGC score         5%
```

**Output columns:** `code`, `web_name`, `element_type`, `gameweek`, `minutes`, and all individual + composite scores.

> Run after adding each new GW file. It reads all `gw*.csv` files in the folder automatically.

---

## 🚀 Getting Started

### Prerequisites

```bash
pip install pandas numpy requests
```

### Running the pipeline

1. **Pull player metadata** (once per season):
   ```
   Run fpl_player_pull.ipynb
   ```

2. **Pull a gameweek** (after each GW finishes):
   ```
   Run gameweek_pull.ipynb
   → Enter GW number when prompted (or press Enter for current GW)
   ```

3. **Recalculate scores** (after each new GW file):
   ```
   Run player_scores.ipynb
   ```

4. **Refresh Power BI** — open `fpl_new.pbix` and hit **Refresh**.

> Update the `folder` path variable in each notebook to point to your local data directory.

---

## 📊 Power BI Report

The `.pbix` file connects to the three CSV outputs:

| Table | Source file | Join key |
|---|---|---|
| Players | `players.csv` | `code` |
| Gameweek Stats | `gw*.csv` (all combined) | `code` |
| Player Scores | `player_scores.csv` | `code` + `gameweek` |

Player photos are loaded directly from the Premier League CDN URL stored in `players.csv`.

---

## 🔗 API Reference

All data is sourced from the **official FPL API** (no key required):

```
https://fantasy.premierleague.com/api/
```

Key endpoints used:

| Endpoint | Used for |
|---|---|
| `bootstrap-static/` | Players, teams, events metadata |
| `fixtures/` | All season fixtures with scores |
| `event/{gw}/live/` | Live GW stats for all players |
| `element-summary/{id}/` | Per-player history (double GW splits) |

---

## 📝 Notes

- The `code` field (not `id`) is used as the permanent player identifier across all tables. Player `id` values can change between seasons; `code` is stable.
- `element_type` values: `1` = GK, `2` = DEF, `3` = MID, `4` = FWD.
- Double gameweeks produce **two rows** per player (one per fixture) in the GW files.
- All percentile scores are calculated **within position** to fairly compare players against their peers.
- Player prices (`value`) are stored in pounds (e.g. `5.5` = £5.5m). The API returns prices ×10 and the notebook divides by 10.
