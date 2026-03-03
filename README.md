# FPL Analytics Hub 2.0

A rebuilt Fantasy Premier League analytics dashboard, motivated by maintainability issues in the original version and the positive reception it received after sharing on Reddit. The volume of requests for the `.pbix` file made me realise the importance of clean organisation, proper documentation, and a pipeline others could actually follow.

---

## What Changed and Why

The original dashboard was built as a learning project. Early mistakes and bad habits accumulated over time, making the model increasingly difficult to maintain and scale. Two specific problems drove the rebuild.

### 1. Messy Power Query

The original dashboard handled all transformation and loading within Power Query. Steps were inconsistently named, logic was hard to trace, and debugging required navigating a long chain of tightly coupled steps. This also caused long refresh times and occasional freezing despite the model not being particularly large. Much of this stemmed from the absence of global keys for players and teams across seasons.

### 2. Player and Team IDs Breaking Between Seasons

FPL rebuilds its internal player IDs every season. The original dashboard had no mechanism to handle this — my workaround was a series of complex joins that technically worked but were hard to follow and contributed to the performance issues mentioned above.

---

## Architectural Changes

### Python Data Pipeline

All heavy data extraction now happens in Python scripts rather than Power Query. Power Query is only responsible for reading CSV files and pulling small reference tables directly from the API. This separates concerns cleanly and makes each part independently maintainable.

### Folder-Based Incremental Loading

Each gameweek is saved as its own CSV file (`gw1.csv`, `gw2.csv` etc.) in a dedicated folder. Power BI reads the whole folder and appends automatically.

Previously I merged each new gameweek into a single master file in Python before Power BI read it. This caused a painful incident where I accidentally ran the merge script twice and had to unpick duplicated data. The new approach also means I can update mid-gameweek rather than waiting for the full gameweek to finish — running the script simply overwrites the current gameweek file.

### Cross-Season Player Identity

The rebuild uses the `code` field from the FPL `bootstrap-static` endpoint, which maps to the Opta player ID and remains stable across seasons. This replaces the fragile internal `id` field as the global player key, making cross-season joins straightforward and reliable.

---

## Data Model

| Table | Type | Key | Relates To | Source |
|---|---|---|---|---|
| FactTable | Fact | player_code + gw +  | Players / Teams / Opponent / Date / Season | Python (CSV folder) |
| Players | Dimension | player_code (Opta ID) | FactTable / Position | Python (CSV) |
| Teams | Dimension | team_code | FactTable / Fixtures | Power Query (API) |
| Opponent Teams | Dimension | opponent_id | FactTable / Fixtures | Duplicate of Teams |
| Position | Dimension | position_id | Players | Dax Table |
| Date | Dimension | Date | FactTable / Fixtures | Dax Table |
| Season | Dimension | season_id | FactTable | Power Query |
| Fixtures | Fact | team_code | Teams / Opponent | Power Query (API) |
| fpl fantasy data | Reference | gameweek_id | Standalone | Power Query (API) |
| fpl player scores | Reference | player_code | Standalone | Python (CSV) |

> Fixtures is unpivoted to one row per team, enabling the team slicer to filter correctly regardless of whether the selected team is home or away.

---

## Tech Stack

| Tool | Role |
|---|---|
| Python (requests, pandas) | API extraction, data shaping, CSV output |
| Power Query (M Code) | File ingestion, light transformation and merging, reference table API calls |
| DAX | Measures, KPIs, dynamic filtering |
| Power BI Desktop | Data modelling and visualisation |
| FPL API | Primary data source |
