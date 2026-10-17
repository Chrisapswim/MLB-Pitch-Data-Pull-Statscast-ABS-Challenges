# 2026 MLB Pitch Data Pull: Statcast + ABS Challenges

## Overview

This folder contains the pipeline that builds a pitch-level dataset of the full MLB regular season (Currently set to 2026). Each pitch comes from Baseball Savant's Statcast data, with Automated Ball-Strike (ABS) challenge labels and derived strike-zone columns added.

## Data Sources

All data is public and comes from Baseball Savant:

| Data | Source |
|---|---|
| Pitch-level Statcast data | [Statcast Search](https://baseballsavant.mlb.com/statcast_search) CSV export ([field definitions](https://baseballsavant.mlb.com/csv-docs)) |
| ABS challenge labels | The same export, filtered with Savant's ABS challenge filters (`hfABSFlag`) and joined back to each pitch on `game_pk`, `at_bat_number`, `pitch_number` |
| Validation | [ABS Challenge Leaderboard](https://baseballsavant.mlb.com/leaderboard/abs-challenges) |

Coverage: 2026 regular season, from Opening Day (2026-03-26) through 2026-09-27.

## Output

**File:** `statcast_2026_regular_abs.parquet`
**Size:** 716,534 pitches × 127 columns (119 Statcast + 8 custom), about 116 MB

| Season totals | Count |
|---|---|
| Challengeable pitches | 345,684 |
| Challenged pitches | 10,557 |
| Overturned | 5,686 |
| Challenged by batter / catcher / pitcher | 4,765 / 5,613 / 179 |

### Custom Columns

| Column | Type | Description |
|---|---|---|
| `challengeable_pitch` | 0/1 | Pitch was eligible for an ABS challenge |
| `abs_flag` | 0/1 | Pitch was challenged |
| `challenge_overturned` | 0/1 | The challenge overturned the umpire's call (0 for unchallenged pitches) |
| `challenger` | category | `batter`, `pitcher`, `catcher`, or `fielder` (fallback only); missing if not challenged |
| `zone_dist_in` | float | Distance in inches from the ball's center to the nearest edge of the ABS zone. Positive = outside, negative = inside. Uses the diagonal for pitches off a corner |
| `miss_margin_in` | float | `zone_dist_in` minus the 1.45" ball radius. Positive = ball by that much, negative = strike by that much |
| `abs_call` | string | `strike` if any part of the ball touches the zone (`miss_margin_in` ≤ 0), else `ball`. Filled for every tracked pitch, including swings |
| `original_call` | string | The umpire's call **before** any challenge. Missing for pitches that weren't a called ball or strike |

The zone columns use `plate_x`, `plate_z`, `sz_top`, and `sz_bot`. The ABS zone is modeled as 17" wide (the plate) and from `sz_bot` to `sz_top` vertically. All four zone/call columns are missing for the 2,678 pitches with no tracking data (mostly pitch-clock automatic balls and strikes).

> **Note:** Statcast's `description` column holds the **final** call, after any challenge. `original_call` holds the umpire's call before the challenge.

## Folder Structure

```
MLB ABS Analysis/
├── Baseball Data Pull.ipynb            # Pipeline: download, label, derive, validate, save
├── statcast_2026_regular_abs.parquet   # Final dataset
├── daily_cache/                        # One parquet per game day + daily_checks.csv
└── README.md
```

## Requirements

- Python 3.11+
- `pandas`, `numpy`, `requests`, `pyarrow` (for parquet)

```bash
pip install pandas numpy requests pyarrow
```

**R users:** read the parquet with the `arrow` package (`install.packages("arrow")`, then `arrow::read_parquet()`).

## How to Run

Open `Baseball Data Pull.ipynb`. The notebook is organized into numbered sections, each with a description.

**If `daily_cache/` is already complete (no downloads):**
Run sections 1–6, then 9 and 10. This rebuilds the final parquet from the cache in about a minute.

**Fresh pull from scratch:**
Run sections 1–10 in order. Section 8 downloads the season one day at a time:
- A game day takes 8 requests (1 for all pitches + 7 ABS filters), and an off day takes 1. That's about 1,470 requests in total, roughly 1 to 1.5 hours.
- Requests go out one at a time, at least 2 seconds apart, with exponential backoff on rate limits and server errors.
- Each finished day is cached, so an interrupted run resumes where it left off. Delete `daily_cache/` to force a full re-download.

| Section | Purpose |
|---|---|
| 1. Setup | Imports and run settings (season dates, output format, cache folder, request pacing) |
| 2. Savant download helpers | Polite, retrying HTTP requests |
| 3. ABS challenge flags | Fetch challenge filters and join them onto pitches |
| 4. Derived columns | Zone distance, ABS call, original umpire call |
| 5. Daily pull + save helpers | One day end to end, with data-quality counters |
| 6. Unit tests | Offline checks of the zone math and call logic |
| 7. Smoke test (optional) | Pull and check one date (8 requests) |
| 8. Full-season pull | Cached, resumable download |
| 9. Combine, validate, save | Build and save the final dataset |
| 10. Diagnostics | Zone-column sanity checks |
| Appendix | Legacy pitch-only pull, disabled by default (`RUN_LEGACY_PULL = False`) |

Set `OUTPUT_FORMAT = "csv"` in section 1 to write a CSV instead (about 460 MB vs 116 MB).

## Pipeline Notes

- **Day-by-day pulls:** Savant's CSV export silently truncates at 25,000 rows, so the season is pulled one date at a time. The pipeline stops with an error if any day hits the cap.
- **ABS filter escaping:** Savant ignores `hfABSFlag` filters unless every `.` is sent as `\.`. A server that ignores a filter returns every pitch with no error, so the pipeline checks every day that each filter actually filtered.
- **Challenger labels:** Savant's "fielder" flag is the union of pitcher and catcher challenges, so it is only used as a fallback label.
- **Data-quality log:** `daily_cache/daily_checks.csv` records per-day pitch and challenge counts plus four problem counters that should be 0.

## Validation

- **Leaderboard match:** challenge totals by role (4,765 batter / 5,613 catcher / 179 pitcher) exactly match Savant's ABS Challenge Leaderboard.
- **Zone accuracy:** on the 10,557 challenged pitches, where `description` is the official ABS ruling, `abs_call` agrees **99.95%** of the time. All 5 disagreements are within 2 inches of the zone edge.
- **Integrity checks:**
  - No duplicate pitches, and no rows lost when the days are combined.
  - Every flagged pitch matched a pitch in the data.
  - Overturned ⊆ challenged ⊆ challengeable.
  - Unit tests on hand-built pitches confirm the zone geometry, including corner pitches and missing data.

## Known Issues

- **One labeling quirk:** game 825000, at-bat 31, pitch 3 (2026-05-30) was challenged but not flagged as challengeable by Savant. It is left as-is (1 of 10,557 challenges).
- **Zone approximation:** Statcast measures `plate_x`/`plate_z` at the front of the plate, while the ABS zone is judged at mid-plate. Calls within about an inch of the edge are the least certain.
- **No umpire identity:** the `umpire` column in the export is empty.
- **Cache is not refreshed automatically:** if Savant revises past data, delete `daily_cache/` and re-pull.

## Acknowledgments

Data courtesy of [Baseball Savant](https://baseballsavant.mlb.com) / MLB Advanced Media. Used for academic, non-commercial purposes.
