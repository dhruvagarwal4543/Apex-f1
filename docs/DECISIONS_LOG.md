# Decisions Log

Every design decision or change to the blueprint is recorded here with its date and reason. Newest entries go at the bottom.

Format: **ID · Date · Step · Decision · Reason · Blueprint section affected**

---

### D-001 · 2026-09-25 · Step 0
- **Decision:** Adopt `MASTER_PROJECT_BLUEPRINT.md` v1.0 as the project specification.
- **Reason:** Reviewed and finalised before implementation.
- **Affects:** All sections.

### D-002 · 2026-09-25 · Step 1
- **Decision:** Use Python **3.11**.
- **Reason:** The blueprint requires 3.11+. 3.11 is installed on the development machine, and every pinned package has a ready-made Windows wheel for it (checked with a test install).
- **Affects:** §32 Step 1.

### D-003 · 2026-09-25 · Step 1
- **Decision:** Create the virtual environment **outside OneDrive** (`%USERPROFILE%\.venvs\f1-race-prediction`), not inside the project folder.
- **Reason:** The project folder is synced by OneDrive. A venv holds thousands of files (about 500 MB) that would sync constantly and can break. The blueprint lists this as a common Step 1 error.
- **Affects:** §32 Step 1 (common errors).

### D-004 · 2026-09-25 · Step 1
- **Decision:** `requirements.txt` pins the **directly used** packages only. The blueprint's "pip freeze matches requirements.txt" check is interpreted as "the pinned packages appear in `pip freeze` with exactly these versions". A full `requirements-lock.txt` can be generated with `pip freeze` if needed.
- **Reason:** Pinning about 100 indirect dependencies by hand is error-prone. pip resolves them consistently from the direct pins.
- **Affects:** §32 Step 1 (expected output).

### D-005 · 2026-09-25 · Step 2
- **Decision:** Register the project venv as its own Jupyter kernel, **"Python 3.11 (f1-race-prediction)"** (`python -m ipykernel install --user --name f1-race-prediction`). All project notebooks use this kernel, and each notebook's setup cell checks package versions against `requirements.txt`.
- **Reason:** The default `python3` kernel on this machine points to a different Python installation, so notebooks silently ran with the wrong packages. Found during the first Step 2 run.
- **Affects:** Reproducibility (§32 Step 1–2).

### F-001 · 2026-09-25 · Step 2 · Verified API findings (`notebooks/01_api_exploration.ipynb`, 2024 data)
- **Pagination:** default `limit` = 30, max 100 (larger values silently capped). `offset` is exact and contiguous. `MRData.total` = number of result records (not races) and is constant across pages. A full 2024 walk gave 479 = total with 0 duplicates. **Races are split across pages** (2024 rounds 6, 11, 16, 21).
- **Types:** every value, including numbers, is a JSON **string**.
- **Silent "no data":** a nonexistent round or season returns **HTTP 200 with total = 0**. Invalid `limit` values fall back to 30 and `offset=-1` to 0. A non-numeric round gives HTTP 404 (HTML); an unknown endpoint gives HTTP 400 (JSON).
- **Fields:** sprint records have the same fields as race results. Qualifying has number, position, Driver, Constructor, Q1–Q3 (Q2/Q3 only for drivers who reached them). `Time` and `FastestLap` are missing for some non-finishers.
- **Status/position:** `position` is numeric for every driver. Non-finishers are ordered by laps completed (24/24 races). positionText codes seen: R (Retired), **W (status "Did not start", with a non-zero grid value)**, D (Disqualified). **6 "Retired" drivers were classified with a numeric positionText. One disqualification (Belgian GP, RUS) has a numeric positionText "20".**
- **grid = 0:** one 2024 case (OCO, Mexico City GP; qualified 19th). Consistent with a pit-lane start; the API does not state the reason.
- **IDs:** driverIds are stable (21 drivers identical 2023→2024; `hamilton` covers 394 results since 2007). Constructor renames produce **new IDs** (`alfa`→`sauber`, `alphatauri`→`rb`, 2023→2024). The `Driver` object reflects **current** details (e.g. permanentNumber); the result's `number` is the car number at race time.
- **Headers:** no rate-limit or retry headers. `Cache-Control: max-age=600`, Cloudflare. Documented limits: 4 req/s burst, 500 req/h, HTTP 429 when exceeded, "will decrease" in future.
- **Affects:** Step 3 client (check `total`, handle non-JSON errors, 429). The DNF, DSQ and non-starter rules need a decision at Step 7 (see below). **No blueprint rule has been changed.**

### D-006 · 2026-09-25 · Step 3 · **needs approval**
- **Decision:** The cache file path **mirrors the request URL**, with parameters sorted by name: `/{path}.json?limit=100&offset=0` → `data/raw/jolpica/{path}__limit-100__offset-0.json` (e.g. `data/raw/jolpica/2024/results__limit-100__offset-0.json`). The blueprint §15 example was `data/raw/jolpica/results/2024_0000.json`.
- **Reason:** The cache key must depend on the complete request (path + all parameters). One rule then works for every endpoint (results, qualifying, sprint, standings, schedule) and stays deterministic and human-readable. The folder root (`data/raw/jolpica/`) and the manifest are unchanged.
- **Affects:** §15 file patterns, §16 storage layout (naming only).

### D-007 · 2026-09-25 · Step 3
- **Decision:** API client rules:
  (a) `limit` outside 1..100 or `offset` < 0 raises `ValueError` before any request, because the API would silently change these values (F-001).
  (b) Retry only on HTTP 429/500/502/503/504, connection errors and timeouts: at most 5 retries (6 attempts), waits of 2, 4, 8, 16, 32 s from `config.yaml`. HTTP 400/404 and other errors are raised at once.
  (c) Only valid JSON responses containing `MRData` are cached. Errors are never cached. A `total = 0` response **is** cached (it is a valid "no data" answer).
  (d) Cache files hold the exact response bytes, written atomically (temp file + rename). Unreadable cache files are downloaded again.
  (e) Pages are merged by (season, round). Exact duplicates are detected with the key (season, round, position, driverId); position is included so 1950s drivers with two cars in one race are not dropped. The number of unique records must equal `MRData.total`, otherwise `JolpicaAPIError` is raised.
  (f) The client never converts data types.
- **Reason:** Implements the verified Step 2 behaviour (F-001) with the simplest safe rules.
- **Affects:** §11.3 client design.

### D-008 · 2026-09-25 · Step 3
- **Decision:** Test layout: `pytest.ini` (sets `pythonpath = .` so tests can `import src`, and registers a `live` marker). All API-client tests are mocked except one `live` test. The live smoke test is the script `tests/smoke_api_client.py` (not collected by pytest). `src/config.py` currently only loads `config.yaml`; seed-setting will be added when a later step needs it.
- **Reason:** Tests must run without depending on the live API. The smoke test must be a deliberate, manual run.
- **Affects:** §31 folder structure (adds `pytest.ini`).

### D-006 approval · 2026-09-25
- D-006 (URL-mirrored cache naming) **approved** by the project owner after the Step 3 review. The cache naming system stays as implemented.

### D-009 · 2026-09-25 · Step 4
- **Decision:** Step 4 runs as a **2024 pilot** (blueprint "What we should do first", action 5) before the 1950–2025 collection. Details:
  (a) Collection uses **season-level paginated endpoints** (`/{season}/results.json`, `/qualifying.json`, `/sprint.json`, `limit=100`), as blueprint §15 specifies, not one request per race. Pages are split back into races and reconciled race by race.
  (b) Collection and flattening code lives in `src/collect.py` (blueprint §31) so the full collection can reuse it. Flattening keeps the **API's raw text values**; type conversion stays in Step 8.
  (c) Pilot tables are saved as `data/interim/{season}_{table}.csv` (races, results, qualifying, sprints, drivers, constructors). Pilot check tables are saved as `outputs/tables/pilot_{season}_*.csv`.
  (d) Two extra small season lists are collected (`/{season}/drivers.json`, `/{season}/constructors.json`, 1 request each) to reconcile driver and constructor information.
  (e) **Official standings are not collected in the pilot.** Blueprint §15 and §20 use them only to validate our derived standings at Step 11, never as features.
  (f) Primary key checked: `(season, round, driver_id)`. Duplicates are counted and reported, never dropped.
- **Reason:** Proves the pipeline end-to-end on one season with a small request budget (15 requests) before scaling.
- **Affects:** §15 (pilot first), §31 (`src/collect.py`, notebook name `02_pilot_collection_2024.ipynb`).

### F-002 · 2026-09-25 · Step 4 · Verified 2024 pilot findings (`notebooks/02_pilot_collection_2024.ipynb`)
- 24 races (rounds 1–24, no duplicates), `race_time` present for all. Sprint rounds 5, 6, 11, 19, 21, 23; calendar and sprint data agree.
- Row counts equal `MRData.total`: results 479, qualifying 479, sprint 120, drivers 25, constructors 10. Season-level data is identical to the per-round Step 3 responses (R1 results, R1 qualifying, R5 sprint).
- 0 duplicate `(season, round, driver_id)` keys; 0 missing race, driver or constructor IDs. The qualifying driver set equals the results driver set in every race.
- **A non-participating driver has no row at all:** Australian GP (R3) has 19 results. `sargeant` is absent and Williams has one entry. Such drivers are not recorded as "W" rows.
- **The season driver list includes a driver with no records:** `antonelli` is in `/2024/drivers.json` but has no race, qualifying or sprint row. The reason is not in the API data.
- grid = 0 once (OCO, R20). Status × positionText (race): Finished/num 287, Lapped/num 138, Retired/R 43, Retired/num 6, Did not start/W 3, Disqualified/D 1, Disqualified/num 1. Sprint: Finished/num 116, Retired/R 3, Retired/num 1.
- Re-running with a new client: 0 network requests, 15 cache hits, byte-identical CSVs. 0 existing raw files changed.
- **No DNF/DSQ/non-starter rule was applied** (still pending for Step 7).

### D-009 approval · 2026-09-25
- D-009 (2024 pilot approach) **approved** by the project owner after the Step 4 review.

### D-010 · 2026-09-25 · Step 5
- **Decision:** `api.jolpica.min_seconds_between_requests` raised from **1.0 to 7.5 s** in `config.yaml`.
- **Reason:** The full collection needs about 800 requests. At 1 s spacing that is up to 3,600/hour, far above Jolpica's documented **500/hour** sustained limit (F-001), so HTTP 429 would be triggered. 7.5 s = 480/hour. Cache hits are not affected.
- **Affects:** §11.1 (rate limits). Cost: about 95 minutes for the first full collection.

### D-011 · 2026-09-25 · Step 5
- **Decision:** Historical collection runs as a command-line script, `python -m src.collect --start 1950 --end 2025` (also `--output-prefix` for test runs), not a notebook. Reports are built by a new module, `src/collection_checks.py`. Outputs use the blueprint §16 names `data/interim/{races,results,qualifying,sprints,drivers,constructors}.csv`. The 2024 pilot files (`2024_*.csv`) are kept unchanged.
- **Reason:** A 95-minute run is safer as a resumable script with a log file (`outputs/logs/`) than as a notebook cell. The report logic is kept separate from collection so it can be tested offline.
- **Affects:** §31 (adds `src/collection_checks.py`; blueprint notebook `02_collect_f1_data` is replaced by the script).

### F-003 · 2026-09-25 · Step 5 · Verified historical collection findings (1950–2025)
- **Completeness:** 76 seasons, **1,149 races, 25,873 results, 10,973 qualifying, 480 sprint rows**, 3,248 season-driver and 1,121 season-constructor rows. **Every season matches every `MRData.total`.** 817 unique drivers, 203 unique constructors, 77 circuits. 0 missing race, driver or constructor IDs.
- **Run:** 749 new downloads, 485 cache hits (1950–1951 subset + 2024 pilot + total reads), **3 retries, all recovered** (one read timeout, two HTTP 500 on `/2004/qualifying.json`), runtime 94.6 min. Re-running from the cache: **0 requests**, 2.7 s, 17 output files byte-identical, 779 raw files and 780 manifest lines unchanged.
- **Duplicate keys:** `(season, round, driver_id)` repeats in **172 rows / 40 races, all 1950–1964** (shared drives: one driver credited in two cars). Positions repeat in 46 races up to 1964 (shared cars, e.g. 1951 French GP: fangio and fagioli both P1 in car 8). **Kept and reported, not dropped.** Qualifying and sprint keys are unique.
- **Qualifying coverage:** none before 1994. **1994–2002 incomplete** (races with qualifying: 1994 15/16, 1995 17/17, 1996 7/16, 1997 10/17, 1998 7/16, 1999 3/16, 2000 4/17, 2001 1/17, 2002 2/17). Complete for every race 2003–2025.
- **Race start time:** missing for **all** races 1950–2004 (731), present for all 2005–2025.
- **FastestLap** only from 2004.
- **positionText codes in the API: only R (9,121), W (358), D (159).** E, F and N never occur. Drivers who failed to qualify have **no result row**.
- **grid = 0: 267 rows**, mostly non-starters (202 with positionText W, 219 with 0 laps, most often status "Withdrew"). Only 40 are classified finishers.
- **Non-starters** carry many statuses (Withdrew 246, Accident 29, Engine 18, Did not start 9, Injury 8, ...). **Disqualified/Excluded:** Disqualified 153, Excluded 8, Underweight 1.
- **136 distinct race status values**, 245 (source, status, code) combinations.
- **Indianapolis 500:** 11 races (1950–1960). **circuitId `indianapolis` is also used for the United States GP 2000–2007**, so the blueprint's filter by race name is required.
- **Identity:** no constructorId has more than one name, no name has more than one ID, and no driverId has more than one identity (the API supplies one current name per ID). No unexpected JSON fields.
- **Other flags:** 793 absent-between-appearances (mostly before 1990), 142 season-list drivers without records, 363 races whose result count differs from the season median.

### F-004 · 2026-09-25 · Step 5.5 · Historical policy analysis (read-only)
- Produced by `python -m src.policy_analysis` from `data/interim/*.csv`, which was **not modified** (hashes verified before and after). Outputs: `outputs/tables/historical_policy_analysis.csv` (27 rows), `historical_policy_examples.csv` (104), `historical_status_category_proposal.csv` (246), `historical_qualifying_coverage.csv` (36).
- **Shared drives:** 83 duplicate (season, round, driver_id) groups = 172 rows in 40 races (1950–1964). **Every group has a different car number per row**, and `(season, round, driver_id, driver_number)` is unique across all 25,873 rows. Shared positions: 105 groups, 228 rows, 46 races, **all the same car and constructor** (co-drivers). 3 shared wins (1951 French, 1956 Argentine, 1957 British GP).
- **Status:** 136 race statuses. **78 statuses occur in more than one category** (e.g. Engine: R 1,855 / W 18 / numeric 138), so status alone cannot define DNF. Proposed categories (count): finished 15,554 · classified_not_running 674 · started_not_classified 9,121 · disqualified 162 · did_not_start 338 · ambiguous 24 (20 W with laps > 0, all "Withdrew"; 3 classified "Withdrew"; 1 classified "Not classified"). Total 25,873.
- **grid = 0 (267):** 202 non-starters/ambiguous (143 of them 1950–1969); **48 started with laps > 0, all 2015–2024** (pit-lane starts); 17 started categories with 0 laps (13 disqualified).
- **Populations:** 25,511 started rows (candidate model rows before Indy exclusion); 338 non-starters with a row (136 of them with grid > 0); 7 failed-to-qualify rows (visible only 1994+); 142 list-only driver-seasons (62 drivers never raced).
- **Constructors:** 203 IDs ↔ 203 names (strict 1:1). 20 IDs reused after gaps of 5+ seasons (aston_martin 1959–60 & 2021–25, mercedes 1954–55 & 2010–25, alfa 4 blocks, renault 3 blocks...). 28 automatic "succession" candidates mix real renames with driver moves.
- **Retiree ordering:** 7,998 adjacent R pairs, **8 laps inversions** (8 races, 1951–1992, none after 2009). 4 races have an R row above a classified row (1954 Indy 500; 2005 Australian GP: button/sato R at positions 11 and 14; 2023_18 and 2024_14: numeric-code disqualifications).
- **Indianapolis:** `race_name == "Indianapolis 500"` selects exactly 11 races / 405 rows / 107 drivers (4 also raced other GPs: ascari, ruby, ruttman, ward). circuit_id `indianapolis` is also used by the 8 US GPs 2000–2007.

## Historical data policies P-01 … P-11: **APPROVED** (2026-09-25, Step 5.6)

Proposed in Step 5.5 (F-004), **approved by the project owner** and applied in Step 5.6 by `python -m src.policies` (code: `src/policies.py`, tests: `tests/test_policies.py`).

**How they are applied:** as extra columns on copies of the interim tables:
- `data/interim/results_policy.csv`: all 25,873 rows, every original column unchanged, plus 18 policy columns.
- `data/interim/races_policy.csv`: all 1,149 races, plus race-level flags.
- Counts: `outputs/tables/policy_application_summary.csv`.

`results.csv`, `races.csv`, `qualifying.csv` and all raw API files are **not modified**; 813 files were verified by SHA-256 before and after. No target columns, features or recoded grid numbers are created yet.

| ID | Approved policy | Implemented as | Rationale | Consequence (numbers from the applied run) |
|---|---|---|---|---|
| P-01 | Keep every car entry. In the model dataset, a driver with several entries in a race is represented by the **best-placed started entry**; an entry count is kept. Raw entries are never deleted. | `entry_count`, `multi_entry_driver`, `is_primary_entry` (ranking: started first, then lowest position, then car number) | 1950–1964 shared drives are separate car entries with different car numbers; `(season, round, driver_id, driver_number)` is unique | 172 rows of multi-entry drivers; 89 secondary entries kept but outside the model population. Derived season points must later sum **all** entries |
| P-02 | Keep co-driver rows at their official position with a shared-position flag. Shared-win races are excluded **only** from winner-specific metrics. | `shared_position`, `shared_position_count`, `race_has_shared_win`; race level `exclude_from_winner_metrics` | Co-drivers of one car are credited with the same position | 228 rows flagged; 3 races (1951 French, 1956 Argentine, 1957 British GP) excluded from winner metrics only |
| P-03 | Missing qualifying stays missing; never drop a race for it. Availability indicators at race and driver level. | `race_has_qualifying`, `has_qualifying_row` (availability flags only; no values imputed) | No qualifying before 1994, gaps 1994–2002 | 631 races without qualifying kept; 14,907 result rows without a qualifying row kept |
| P-04 | grid = 0: started with laps > 0 → pit-lane start; started with 0 laps → unknown; non-starter → non_starter. Raw grid preserved. No era cut-off. | `grid_status` ∈ {given, pit_lane_start, unknown, non_starter}, `pit_lane_start` | grid 0 is mostly non-starters historically; every started grid-0 row with laps > 0 is 2015–2024 | 267 grid-0 rows unchanged; 48 pit-lane starts; 17 unknown. The numeric pit-lane value (field_size + 1) is set when the model dataset is built |
| P-05 | Preserve raw status and positionText; derive categories separately. | `result_category_detail` (9 categories), `result_category` (after P-07) | Status alone is ambiguous (78 of 136 statuses occur in more than one category) | 136 raw statuses kept. result_category: finished 15,554 · started_not_classified 9,141 · classified_not_running 678 · did_not_start 338 · disqualified 162 |
| P-06 | Primary DNF definition = **started, not classified**. Classified-but-not-running is flagged separately and is not DNF. | `is_dnf(category)` function (no target column yet); `retired_while_classified` | Matches blueprint §14 (positionText R); classified retirements hold a legitimate classified position | 9,141 rows would be DNF; 678 flagged as retired while classified |
| P-07 | The 24 ambiguous rows count as started, with an ambiguity flag. | `ambiguous_status`; mapping: W + laps > 0 → started_not_classified (20); classified "Withdrew"/"Not classified" → classified_not_running (4) | They completed laps or hold a classified position | 24 rows added to the started population, all flagged |
| P-08 | Model population = drivers who actually started, excluding the Indianapolis 500. Non-starters are not counted as starts in history features. | `started`, `in_model_population` (= started ∧ primary entry ∧ not Indy 500) | The prediction point is after the grid is final; non-starters are known then | 25,535 started rows; 338 non-starters excluded; **25,066 model-population rows**, unique per (season, round, driver_id) |
| P-09 | Use API `constructor_id` as-is; no automatic merging by name or rename detection. | No new column; constructor_id untouched | Automatic succession detection mixes real renames with driver moves | 203 IDs unchanged; constructor history restarts when the API ID changes (e.g. alphatauri → rb) |
| P-10 | Use the official API position order; flag the 8 ordering exceptions; never re-sort. | `laps_order_exception` (both rows of each inverted pair); race level `race_has_laps_order_exception` | Order follows laps in 7,990 of 7,998 adjacent pairs | 16 rows / 8 races flagged; positions unchanged |
| P-11 | Exclude races with `race_name == "Indianapolis 500"` (exact); never by circuit_id. | `is_indianapolis_500` (row and race level); used in `in_model_population` | circuit_id `indianapolis` is also used by the 2000–2007 US GPs | 11 races / 405 rows outside the model population; the 8 US GPs stay. History features must also skip these rows |

### D-012 · 2026-09-25 · Step 6 · Weather collection design
- **Decision:**
  (a) Source: Open-Meteo Historical Weather (archive) API, hourly: temperature_2m, relative_humidity_2m, precipitation, rain, cloud_cover, wind_speed_10m, wind_direction_10m, weather_code. No soil/surface variables (no clear link to race conditions).
  (b) New module `src/weather.py` with its own small cached client (same behaviour as the Jolpica client: deterministic cache, 1 s spacing, retries on 429/5xx/timeouts). Cache: `data/raw/openmeteo/{weather,timezones}/{name}__{hash}.json`, where the hash is SHA-1 of the sorted request parameters. Separate logs `data/raw/openmeteo/_manifest.csv` and `_failed_requests.csv`, so no existing Jolpica cache or manifest file is touched.
  (c) One request per race: 72 UTC hours (start date −1 … +1).
  (d) Start time: official UTC for 2005+; **1950–2004 assumed 14:00 local** (no official times, F-003), flagged `imputed_local_14h`.
  (e) Windows relative to H0 (start rounded down): prestart −3…−1 h, race 0…2 h. Accumulated variables use the following-hour values (a+1…b).
  (f) Outputs keep raw observations (`weather_hourly.csv`, −6…+6 h, unchanged values) separate from simple window summaries (`weather_race.csv`: mean/min/max/sum, circular mean for wind direction). **No wet flag, no thresholds** (deferred).
  (g) **Indianapolis 500 (P-11): not requested**, weather_status `excluded_indianapolis_500`, never model-usable; the US GP 2000–2007 is included.
  (h) Sanity checks only flag (severity info/warning/error); `model_use_allowed` = no error-level flag.
- **Reason:** Blueprint §11.2, §13.4, §19.6. The design keeps raw and derived data separate and makes collection resumable and reproducible.
- **Affects:** §11.2 (single `_manifest.csv` → separate Open-Meteo manifest), §13.4.

### D-013 · 2026-09-25 · Step 6 · Weather requested in UTC; local time from zoneinfo
- **Decision:** Request weather with `timezone=GMT` (UTC timestamps). Use Open-Meteo `timezone=auto` **only** to look up each circuit's IANA timezone name (77 cached requests). Compute local times with Python `zoneinfo` (historical daylight-saving rules).
- **Reason (verified):** Open-Meteo's `auto` mode applies **one fixed, current UTC offset to every date**. Las Vegas on 18 Nov 2023 returned `utc_offset_seconds = -25200` (UTC−7), but the correct offset is UTC−8 (DST ended 5 Nov). Its "local" hours matched UTC hours shifted by exactly 7 (65/65), not 8. Trusting it would misalign about half of all races by one hour.
- **Affects:** config `api.openmeteo.timezone` changed from "auto" to "GMT".

### F-005 · 2026-09-25 · Step 6 · Verified weather collection results
- **Requests:** 1,145 network requests + 70 cache hits (the 2023–2024 subset) on the first full run, **0 retries, 0 failures**, 19.4 min. Re-runs: 0 requests, 1,215 cache hits, 9 output files byte-identical. 1,215 raw files (1,138 races + 77 circuit timezones). 800 pre-existing Jolpica raw/manifest/interim files verified unchanged.
- **Coverage:** 1,149 races; 11 Indianapolis 500 excluded; **1,138 / 1,138 eligible races have weather (100%)** in every era. 0 missing hourly values (14,794 hourly rows, 8 variables).
- **Sanity:** no implausible temperature, humidity, precipitation, wind, cloud or weather-code values; no duplicate or irregular timestamps; no missing window hours; all units as expected. Weather grid cell 0.7–8.9 km from the circuit (median 4.2). Race-window ranges: temperature 6.6–36.9 °C, precipitation 0–16.9 mm (321 races > 0 mm, 157 ≥ 0.5 mm), max wind 1.1–41.8 km/h.
- **Start times:** 418 official (2005–2025), 731 imputed (1950–2004). **2005–2006 official `race_time` values are local clock times labelled as UTC** (see P-12). Almost all are "14:00:00Z" across Melbourne, Tokyo, Europe and Montréal, which as UTC gives local starts of 01:00, 23:00, 16:00 and 09:00. The local starts are spread out (R = 0.545 and 0.530) while every season 2007–2025 has R ≥ 0.833. **All 37 races flagged (error), `model_use_allowed = False`, values not changed.**
- Local date differs from the API date in 5 races: Las Vegas 2023–2025 (correct: the API date is the UTC date) and 2005/2006 Australia (a symptom of the clock issue).
- **Wet-race plausibility:** all 9 reference wet races (general-knowledge list, not verified) show precipitation in the reanalysis. Example: 2024 São Paulo, 3.2 mm in the race window.
- **Model-usable weather:** 1,101 races (all 329 races 2010–2025).

## P-12 · 2005–2006 start times: **APPROVED** (2026-09-25, Step 6.1)

| ID | Approved policy | Implemented as | Rationale | Consequence |
|---|---|---|---|---|
| P-12 | The 2005–2006 Jolpica `race_time` values (e.g. "14:00:00Z") are **local clock times**, not UTC. They are read as local time at the circuit and converted to UTC with the IANA timezone database before weather alignment. The original value is never changed. | config `weather.race_time_local_clock_seasons: [2005, 2006]`; `race_start(..., local_clock=True)`; `start_time_source = "p12_local_clock"`; flag `p12_local_clock_applied`; info check `p12_local_clock_interpretation` | F-005: the same "14:00:00Z" across Melbourne, Tokyo, Europe and Montréal gives impossible UTC-based local starts (01:00, 23:00, 16:00, 09:00); local-start clustering R = 0.545/0.530 vs ≥ 0.833 in all other seasons | 37 races re-aligned; detector R after P-12 = 0.994 (2005) and 0.986 (2006), so no longer flagged; all 37 model-usable |

### F-006 · 2026-09-25 · Step 6.1 · P-12 applied and verified
- Re-run from cache: **0 network requests** (1,215 cache hits; the corrected UTC starts fall on the same request dates). 2,031 raw and interim files (Jolpica raw + manifest, Open-Meteo cache + logs, all non-weather interim CSVs) verified unchanged by SHA-256.
- **Only the 37 P-12 races changed.** The other 1,112 race rows are identical in every pre-existing column (one new column, `p12_local_clock_applied`), and their 14,313 hourly rows are identical. 481 hourly rows re-aligned.
- Corrected local starts: 14:00 for most races, Bahrain 14:30, Malaysia/Turkey 15:00, Canada 13:00, 2006 British GP 12:00 (World Cup schedule). Historical DST is handled correctly: Melbourne 6 Mar 2005 UTC+11 (DST) vs 2 Apr 2006 UTC+10 (DST already ended), Indiana 2005 UTC−5 (no DST observed) vs 2006 UTC−4.
- Quality report: **0 errors, 0 warnings.** Info only: 720 imputed pre-2005 starts, 37 P-12 interpretations, 3 local-date differences (Las Vegas 2023–2025, correct).
- **Usable weather: 1,138 of 1,138 eligible races** (1,149 − 11 Indianapolis 500). Start-time sources: 381 official UTC (2007–2025), 37 P-12 local clock (2005–2006), 731 imputed 14:00 local (1950–2004).

### D-014 · 2026-09-25 · Step 7 · Raw validation design
- **Decision:** Step 7 is implemented as the read-only validator `src/validate_raw.py` (`python -m src.validate_raw`). It imports no API client (0 requests) and fingerprints all inputs, including both API manifests, before and after. It:
  (a) compares about 200 checks with the documented values from F-003 … F-006 and P-01 … P-12 (PASS/FAIL), or reports them as INFO;
  (b) **recomputes** all 18 policy columns from the raw results with `src/policies.py`, and every weather start time and window summary with `src/weather.py`, and compares them field by field;
  (c) checks 14 cross-table key relationships in both directions;
  (d) classifies every missing value as structurally missing / historically unavailable / legitimately absent / unexpected;
  (e) writes anomalies as `known_handled` (documented and covered by a policy) or `review`.
- **No cleaning beyond the approved policies:** no row is removed, no value recoded or imputed, no target, feature or split created. The policy-annotated files (`results_policy.csv`, `races_policy.csv`) and the weather tables remain the cleaned-by-policy foundation for EDA.
- **Reason:** Blueprint Step 7 (validation) + Step 8 (cleaning limited to approved rules), as instructed.

### F-007 · 2026-09-25 · Step 7 · Validation results (read-only)
- **200 checks: 193 PASS, 7 INFO, 0 FAIL.** 14/14 cross-table relations as expected. Two runs gave byte-identical outputs; 2,035 raw and interim files unchanged; no network used.
- **Confirmed:** 1,149 races (11 Indianapolis 500, 1,138 eligible, the 8 US GPs at `indianapolis` kept); 25,873 result rows; 10,973 qualifying rows; 480 sprint rows (24 weekends 2021–2025, matching the calendar); **25,066 model-population rows**, one per driver per race, covering all 1,138 eligible races; all 18 policy columns reproduce exactly from raw data; weather 1,138/1,138 with 0 failures, 0 missing window values, starts reproducing from raw data (381 official UTC / 37 P-12 / 731 imputed), grid distance ≤ 8.9 km.
- **New source properties (not repaired, listed in `raw_anomalies.csv` as `review`):**
  - **Official position numbering skips positions in 24 races (1956–2002)**, mostly before withdrawn entries (entries not present in the data, e.g. non-qualifiers). In 7 of them one started driver (usually disqualified or not classified) comes after the gap.
  - **45 model-population races do not have consecutive official positions 1..k**, because of numbering gaps, P-01 secondary entries, P-02 shared positions, or non-starters numbered between classified drivers (e.g. 2024 São Paulo).
  - **Repeated qualifying_position in 5 races (12 rows, 2023–2025)**, e.g. 2024 Monaco (two drivers each at 14 and 15); 2 rows have Q2/Q3 times without Q1 (e.g. 2024 Dutch GP, albon). Source anomalies; values kept (P-03).
  - **Unexpected missing values: 286** = modern fastest-lap fields for some classified cars (results: 20–47 per column), sprint fastest_lap_rank (98) and lap/time (2 each), one "Finished" row without race time, and 14 season-list drivers without date of birth/nationality.
- **Missing data is otherwise fully explained:** race start time (before 2005), driver code / permanent number (driver-level attributes the API gives only for some drivers), race time (only cars finishing on the lead lap), fastest lap (from 2004), Q2/Q3 (session formats and knock-outs), weather (Indianapolis 500 not requested).

### Open question for the later target/model-dataset step (not decided in Step 7)
| Question | Evidence |
|---|---|
| Should the finishing-position target use the **official position** (may have gaps, F-007) or the **rank among model-population starters** (1..k)? | 45 model-population races with non-consecutive official positions; 24 races with numbering gaps |
| Wet/dry threshold for race weather (not chosen in Step 8) | F-008: precipitation resolution 0.1 mm; era-stratified evidence in `eda_weather_wet_by_era.csv` |
| Training window given strong drift across eras | F-008: DNF rate, pole conversion, grid–finish correlation and points systems all change by era |

### D-015 · 2026-09-25 · Step 8 · EDA design
- **Decision:** EDA is implemented in `src/eda.py` (`python -m src.eda`), read-only and deterministic, with `notebooks/03_eda.ipynb` presenting the results. Outputs: 31 tables `outputs/tables/eda_*.csv` and 19 figures `outputs/figures/eda/*.png` (every figure has a table twin). Eras: 1950–1969, 1970–1989, 1990–2009, 2010–2025. The descriptive population for outcomes is the P-08 model population; "all rows" is used for status and non-starter frequencies. Outcome statistics use the **official position** (no re-ranking). DNF is described with the P-06 definition, without creating a column.
- **Leakage labelling:** `eda_variable_roles.csv` labels every column as pre-race / POST-RACE OUTCOME / DURING RACE (oracle proxy) / population filter / metadata. Correlations are labelled with the timing of both variables and are descriptive only.
- **Charts:** reference palette from the dataviz method, validated with its script (categorical slots 1–5: all checks pass, contrast WARN relieved by legends and table twins; era ordinal blue ramp: pass). One axis per panel, no dual axes.
- **Reason:** Understand the data before defining targets and features (blueprint §22), without creating either.

### F-008 · 2026-09-25 · Step 8 · EDA findings (descriptive; no decisions)
- **Scale:** 76 seasons, 1,149 races (1,138 eligible), 25,066 model rows, 687 drivers in the model population (817 with any result), 203 constructor ids, 77 circuits, weather 100%.
- **Drift across eras (strong):** DNF rate among starters 45.4% / 48.1% / 37.7% / **14.9%**; pole win rate 40.7% / 32.6% / 46.7% / **52.4%**; podium from pole 51.7% / 50.7% / 69.8% / **80.5%**; grid vs position Spearman 0.42 / 0.33 / 0.45 / **0.63**; mean starters per race 19.4 / 24.5 / 22.3 / 20.9; drivers per season 46.8 → 23.1; points scales change (max 9–10 per race before 2010, 25+ after).
- **Qualifying:** none before 1994, partial 1994–2002, complete 2003+. Q2 from 2005, Q3 from 2006. From 2003: grid == qualifying for 72.9% of starters, 1–3 places better 19.8% (penalties to others), worse 6.0%, pit lane 0.5%. Qualifying position vs grid ρ = 0.97.
- **grid = 0:** 1950–1969 all non-starters (143); 2010–2025 mostly pit-lane starts (48 of 56).
- **Position change:** classified finishers gain 2.9 places on average (mainly because retirements ahead of them move them up; not overtaking skill).
- **Missing data:** all missingness explained by era, driver attribute, knock-out qualifying or outcome. **286 unexpected values = 258 post-race fastest-lap/time fields (no action) + 28 = 14 drivers (2025 list) without dob/nationality.**
- **Weather:** precipitation resolution 0.1 mm (> 0 and ≥ 0.1 mm are identical: 321 races, 28%); ≥ 0.5 mm 158 races (13.9%); ≥ 2 mm 53 (4.7%). Pre-start vs race: temperature ρ 0.96, precipitation ρ 0.62 (98 races wet only before, 98 only during). **Within 1990–2025, wet races show higher DNF (e.g. 2010–25 ≥ 0.5 mm: 18.5% vs 14.4%) and lower pole-win rates (27–47% vs 48–53%). 1950–1989 shows no clear signal** (assumed 14:00 windows, small samples). Pooled weather–outcome correlations are weak (|ρ| ≤ 0.17) and confounded by era.
- **Anomalies still needing a later decision:** position target (official vs rank among starters: 45 races, 24 numbering-gap races), repeated qualifying positions (5 races), driver dob (14), the wet/dry threshold, and the training window (drift).

---

## Open assumptions to verify (from the blueprint)

| Assumption | Verify at | Status |
|---|---|---|
| Jolpica rate limit ≈ 4 req/s burst, 500 req/h | Step 2 | **Confirmed from official docs** (F-001); no rate-limit headers are sent |
| Open-Meteo free limits ≈ 600/min, 5,000/h, 10,000/day | Step 6 | **Confirmed** from the Open-Meteo terms (also 300,000/month; data CC-BY 4.0) |
| Official `race_time` is UTC from 2005 (Appendix A) | Step 6 | **Resolved by P-12 (APPROVED):** UTC from 2007; 2005–2006 values are local clock times, converted with the timezone database |
| `grid = 0` means pit-lane start | Step 7 | **Resolved by P-04:** pit-lane start only when the driver started and completed laps (48 rows); otherwise unknown or non-starter |
| `positionText` codes D, E, F, N meanings | Step 7 | **Resolved (F-003):** only R, W, D exist in 1950–2025. E, F, N never occur. W covers several non-start statuses |
| `position` orders retirees by laps completed | Step 7 | **Confirmed for 1950–2025 (F-004):** 8 inversions in 7,998 adjacent pairs, none after 2009 |
| circuitId `indianapolis` reused for 2000–2007 US GP | Step 7 | **Confirmed (F-003):** US GP 2000–2007 uses it. Filter the Indy 500 by race name |
| 1950s shared drives appear as separate rows | Step 7 | **Confirmed (F-003):** separate rows; driver_id repeats within a race (172 rows, 1950–1964) |
| Qualifying available from 1994 (blueprint §11.1, §13.2) | Step 7 | **Only partly (F-003):** 1994–2002 has qualifying for only some races; complete from 2003 |
| Constructor renames produce different constructorIds | Step 11 | **Confirmed for 2023→2024** (alfa→sauber, alphatauri→rb); earlier renames open. No ID ever has two names in the API |
| Open-Meteo Historical Forecast API exists (Future Work only) | — | Open |

## Earlier pending questions: **resolved by the approved policies**

| Question (raised in Steps 2–5) | Resolved by |
|---|---|
| Does a *classified* retirement count as DNF? | P-06: no; flagged `retired_while_classified` |
| DSQ detection must also use `status` (positionText can be numeric) | P-05: `result_category_detail` uses positionText **and** status |
| The non-starter filter must not use grid (DNS rows carry a grid value) | P-04 / P-08: `started` comes from the category, not from grid |
| Drivers absent from a race (no row) or listed without records | P-08: they never become model rows; no rows are invented |
| Duplicate (season, round, driver_id) rows and shared positions before 1965 | P-01 / P-02 |
| 1994–2002 qualifying gaps | P-03 |

---

## Phases 1–10 (autonomous build, 2026-11-06/07)

### D-016 · 2026-11-06 · Phase 1 · Model base table and target
- **Decision:** `src/dataset.py` writes `data/processed/master_base.csv`: all 25,873 result rows, typed, joined with qualifying (times parsed to seconds), the same-weekend sprint, and race weather (pre-start and race window), plus every approved policy column. Keys: `row_id = race_id_driver_id_carnumber` (unique for all rows) and `race_driver_id = race_id_driver_id` (unique within the P-08 model population).
- **Target (resolves the Step 7 open question):** `target_finish_position` = rank of the official position **among the model-population starters** of the race (`rank(method="min")`), so every race is 1..k. Co-driver shared positions (P-02) keep their tie. The official position is kept as `official_position`. `target_winner = (target == 1)`, `target_podium = (target <= 3)`, `target_dnf = P-06`.
- **Grid (P-04):** given → grid; pit-lane start → `field_size + 1`; unknown or non-starter → NaN. `field_size` = number of model-population starters (known at T0).
- **Reason:** race-level metrics need a per-race permutation; 45 races have non-consecutive official positions (F-007). No approved policy or raw file is changed.

### D-017 · 2026-11-06 · Phases 2–4 · Features, protocol and selection
- **Features (`src/features.py`, 46 features, `feature_list_v1.json`):** grid/qualifying (9), sprint (2), driver form (6), driver career (7), championship before the race (5), constructor form (5), circuit history (5), pre-start weather (7). History uses aggregate → shift(1) → roll. Team and circuit features are aggregated per (group, race) **before** shifting. Standings use rounds < current only. Race-window weather is **excluded** (oracle proxy; E6 ablation only).
- **Protocol (`src/train.py`):** training window from 1950 (config). Hyperparameters tuned on expanding season folds 2017–2021 (Grid/RandomizedSearch over the blueprint §26 spaces, reduced: RF 8 and HGB 10 random configs, RF 200 trees). Every model is then refit **walk-forward** (train seasons < S, predict S) for S = 2017…2022.
- **Selection rule (pre-declared before results were seen):** among ML models (baselines excluded), the best **pooled walk-forward 2017–2022** primary metric: winner and podium = PR-AUC, finishing position = MAE. The 2022 hold-out is reported separately. Reason for pooling: 2022 alone has only 22 winners, too few to separate models (blueprint §27.3 allows out-of-fold data when one season is too noisy).
- **Race-level coherence:** win = `p_i^τ / Σ p_j^τ` with τ fitted on 2017–2021 walk-forward predictions (winner log-loss). Podium = capped rescaling so each race sums to exactly 3 and every value is ≤ 1. Podium method (E4) chosen by pooled podium overlap among {regression, podium classifier, winner classifier}. The final order is the podium (ordered by win probability) followed by the regression order; ties go to better grid, then driver_id (unique and deterministic).
- **Historical replay = walk-forward models.** A race in season S is predicted only by the pipelines trained on seasons < S (`models/walkforward/S/`), so the website never shows in-sample "predictions". Seasons ≤ 2016 are not replayable (every saved model saw them). The 2023–2025 replay stays locked until the approved final evaluation (`models/final/` = trained on 1950–2022).
- **Bug found and fixed:** race metrics broke ties in win probability by row order, and the feature file is sorted by actual finish, so a constant (dummy) model scored 100% winner accuracy. Ties are now broken only by pre-race information (grid, driver_id). Regression test: `tests/test_evaluate.py::test_tied_probabilities_do_not_leak_row_order`.

### D-018 · 2026-11-06 · Phases 7–8 · Application architecture
- FastAPI (`src/api.py`) + React/Vite (`frontend/`) + Plotly. FastAPI also serves the built site at `/`. No database: the processed CSVs and joblib pipelines are the data source.
- Dependencies added: fastapi 0.115.12, uvicorn 0.34.3, httpx 0.28.1 (TestClient). Frontend: react 18.3.1, vite 5.4.11, plotly.js-dist-min 2.35.2, playwright-core 1.49.1 (browser test against the installed Edge; no browser download).
- Outcomes are returned only by `/api/race-results`, and the UI calls it only after an explicit "Reveal" click.
- **Custom (hypothetical) driver lineups are not implemented.** The models are trained on real full fields with race-context features (field size, teammate gap, championship ranks within the race). Re-normalising over an arbitrary subset would present invalid probabilities, so driver selection only filters and compares within the real field (prompt §3B).

## Custom race simulator (2026-11-07) — project owner removed the earlier freeze restrictions

### D-019 · 2026-11-07 · Scenario model (Mode B)
- **Why v1 cannot simulate custom races:** v1 inputs include facts that exist only for a real race: `field_size`, `grid_pct`, `teammate_grid_diff`, championship rank among the race's entrants, qualifying gap to pole and sprint result. Its probabilities are also normalised over a real full field.
- **Model:** Plackett–Luce ranking model with linear strength u = w·z over 22 scenario features (z = median-imputed, missing-indicator, standardised): grid (log grid, front row, pole), grid × circuit pole-win rate, grid × wet, driver form/career, season points share, constructor form, driver/constructor circuit history. Implementation in `src/ranking.py` (analytic gradient, tested against finite differences and simulated data).
- **Driver/constructor snapshots** (`src/snapshots.py`): features for an extra pseudo race appended after season S, computed with the training feature code itself. Only (driver, season) pairs with at least one start exist. Low-evidence snapshots (< 10 career starts or < 5 starts that season) are flagged.
- **New feature columns** (era-comparable, same leakage rules): `driver_last5_avg_finish_pct`, `driver_circuit_avg_finish_pct`, `constructor_last5_avg_finish_pct`, `constructor_circuit_avg_finish_pct`, `wx_prestart_wet`. The 46 v1 features are unchanged (verified identical).
- **Wet definition:** pre-start precipitation ≥ 0.5 mm (`config weather.wet_precip_threshold_mm`). Reanalysis weather codes ≥ 51 were not used: they flagged 31% of starts as wet (light drizzle), versus 11.8% of races by precipitation.
- **Probabilities:** win = softmax(u) (exact). Positions, podium and top-5 come from seeded Gumbel-max Monte Carlo (default 10,000 races). The predicted order is the most probable order (decreasing u).

### D-020 · 2026-11-07 · Scenario model selection and use of 2023–2025
- **Run 1** (candidates frozen: pl_grid, hybrid_gbm, pl_linear × start {1950, 1981} × half-life {none, 8}). Selection by lowest pooled walk-forward 2017–2022 winner log-loss → `pl_linear_from1981_hl8`. In the same run, **2023–2025 outcomes were accessed for the first time** (one-time test). Saved as `outputs/tables/final_test_2023_2025_run1_frozen.csv`. The v1 system's 2023–2025 result there is a clean hold-out.
- **Finding:** full-order PL win probabilities were poorly calibrated (winner log-loss 1.99 vs v1 1.36 on 2017–2022). The cause is that the noisy midfield and retirement order dominates the likelihood.
- **Run 2:** added truncated top-k PL (k ∈ {1, 3, 5, 10}). Selected by the same rule: **`pl_linear_from1981_hl8_top1`** (dev winner log-loss 1.325; v1 1.362; difference not significant). Its 2023–2025 numbers are labelled **post-hoc** everywhere, because those outcomes had been seen in run 1.
- **Production:** refit on 1981–2025 (`models/scenario/ranker_v1.joblib`).
- **Historical replay:** 2023–2025 is unlocked and uses `models/final/` (trained ≤ 2022, out-of-sample). Seasons ≤ 2016 are now available as **retrospective fits**, labelled in the UI and API (`mode: retrospective`) as not genuine predictions.
- **Backups:** v1 models, tables and processed features as of the start of this phase are in `backups/v1_2026-11-07/`.

### D-021 · 2026-11-07 · Full-grid simulator with prime form and customisable cars
- **Requirement:** a default full grid at official capacity; prime-form drivers; any car for any driver; no manual grid (estimated by the model); car changes must matter through learned weights.
- **Problem found:** v1 scenario driver inputs (finish percentiles, win rates, points share) were earned in a specific car, so a car swap double-counted the original car.
- **Car-neutral driver measures** (`src/features.py`, `src/driver_ratings.py`):
  - teammate gaps (finish and grid percentile vs. teammates in the same race and car; last 10 races), teammate beat rate;
  - a **teammate-network rating** per (driver, season): sparse regularised least squares on all teammate pairs (fusion λ = 4 across consecutive seasons, ridge λ = 0.05). Gaps are divided by each season's RMS teammate gap for era comparability.
  - Leakage-safe: a race in season S uses ratings fitted on seasons ≤ S−1; a snapshot after season S uses ≤ S (`tests/test_driver_ratings.py`).
- **Car features** (constructor-season snapshot): race pace (finish percentile, points share, win/podium rates), qualifying pace (grid percentile, average and best car), reliability (DNF rate), circuit history, and **crew skill** (mean/best rating of the drivers who raced the car), so the model can separate car from driver.
- **Models** (selected by pre-declared rules on walk-forward 2017–2022):
  - Qualifying model `quali_best_pl_hl8`, chosen by grid Spearman: 0.7227 vs 0.7217.
  - Race model `carneutral_pl_hl8_top1`, chosen by winner log-loss: 1.352. The car-confounded v1 scenario model scores 1.325, so the cost of car-neutrality is small and within noise.
  - Two-stage simulator, grid not given: 1.815 winner log-loss, 46.3% winner top-1, 58.5% podium overlap.
  - 2023–2025: post-hoc only.
- **Prime form:** the season with the highest teammate-network rating (quali + race pace) among seasons with ≥ 8 starts; otherwise the season with most starts (low evidence). The first rule tried, the model's combined driver-skill contribution, produced an implausible ranking (noisy collinear short-window terms) and was replaced before any user-facing release. Known bias: drivers with unusually strong teammates (e.g. Hamilton) are underrated by teammate comparisons.
- **Grid capacity:** largest field at the circuit since 1990 (or ever, if none since), capped at 26 (long-standing regulatory maximum; reports that the 2026 rules set 24 are unverified).
- **Default line-ups:** "All-time greats" = drivers ranked by prime rating with ≥ 50 career starts; "2025 field" = 2025 drivers, filled with greats. Each driver gets the car of their prime season.
- **Simulation:** each simulated race draws a grid from the qualifying model, then a race order from the race model with that grid (`ranking.two_stage_simulation`). Probabilities are Monte Carlo frequencies (default 5,000 races, seeded). The predicted order is by expected position.
- **Removed:** manual grid positions from the interface and API. Earlier simulator artifacts (`ranker_v1.joblib`, `metadata.json`) are kept for the model comparison.

### D-022 · 2026-11-07 · Interactive ML algorithm comparison (course handout coverage)
- **Source of truth:** `Machine Learning_Course Handout_July - December 2026.pdf`. It lists regression (Linear, Ridge, Lasso), classification (Logistic Regression, SVM, Decision Tree, KNN), ensembles (Decision Forest, Bagging, Boosting), unsupervised learning (DBSCAN, BIRCH, K-Means; anomaly detection by density estimation), Bayesian learning (Bayes rule, MLE), PCA and an introduction to ANNs.
- **Race predictors (18, `src/algorithms.py`):** all use the same car-neutral scenario features on the same chronological data.
  - Classification of **podium** (15% positives; winners at 5% are too rare for KNN/SVM): Logistic Regression, Decision Tree, KNN, SVM (RBF, trained 1995+ for O(n²) cost), Gaussian Naive Bayes (Bayesian learning), Random Forest, Bagging, AdaBoost, Gradient Boosting, MLP (intro to ANNs).
  - Regression of the **finishing percentile**: Linear, Ridge, Lasso, Decision Tree, Random Forest and Gradient Boosting regression.
  - Also: the Plackett–Luce ranking model (MLE over finishing orders) and a grid-only baseline.
- **From score to race probabilities:** classifiers give logit(p) (SVM: decision function; Naive Bayes: log-probability ratio, because its probabilities saturate at 0/1); regressors give −prediction. A per-algorithm race temperature τ (maximum likelihood of real winners) is the validation-based calibration, and races are drawn Plackett–Luce from τ·score. τ differs a lot between algorithms (e.g. Naive Bayes 0.05 = severely over-confident; regressors ≈ 14 because percentiles are on a 0–1 scale).
- **Fair protocol:** walk-forward 2017–2022; hyperparameters (small grids) and τ chosen on 2017–2019; every metric on 2020–2022 (61 races). The production model is refit on ≤ 2025 with τ refit on all 2017–2022 walk-forward scores. "Best validated" = lowest 2020–2022 winner log-loss (Plackett–Luce 1.370; logistic regression 1.421; grid baseline 1.705; worst decision-tree regression 1.747). It is offered, never forced.
- **Fixes found during validation:** Naive Bayes scored 0.55 ROC-AUC because clipped 0/1 probabilities became ties (fixed: log-probability ratio → 0.930). SVM picked the winner in only 21% of races with a fixed balanced class weight (fixed: class_weight added to its grid; calibration chose None → 47.5%).
- **Simulation with common random numbers** (`src/multi_sim.py`): every algorithm sees the same sampled qualifying grids and the same race noise. Scores are precomputed as an N×N driver × grid-slot table (exact, because features depend on the grid only through the slot). Compare-all (18 algorithms × 5,000 races) takes ~2 s. Qualifying is shared (the validated qualifying model), so differences come from the race algorithm.
- **Explanations:**
  - Model-specific: exact weight × value contributions (linear models, Plackett–Luce); the decision path (trees); the 8 nearest past driver-races (KNN); class-conditional means (Naive Bayes); impurity importance (forests and boosting).
  - For every algorithm: permutation importance on 2020–2022, and a model-agnostic local sensitivity (group → field median) on the shared τ·score scale, labelled "not causal".
- **Unsupervised (`src/unsupervised.py`), not race predictors:**
  - Gaussian-mixture density of real driver–car profiles (components chosen by BIC). It flags *hypothetical* pairings below the 1% density quantile of real entries. Real prime pairings are not flagged: rare, but real.
  - Car-season archetypes: K-Means (k = 2, best silhouette, capped at 3 for the chart palette), DBSCAN (k-distance eps) and BIRCH on a PCA map.

### D-023 · 2026-11-11 · Circuit-aware course models (pre-registered before retraining)
- **Why:** on an identical 2026 grid, the 16 course models already react to the circuit, but only through three signals: the pole-to-win rate × grid slot, the driver's starts at the circuit, and the team's past finishes there. Both decision trees are completely insensitive (they never split on those features).
- **Change:** add two leakage-safe circuit features that already exist in `master_features.csv` (expanding means over *previous* races only, see `features.py`):
  - `circuit_dnf_rate`: attrition at the circuit;
  - `constructor_circuit_avg_grid_pct`: the team's past qualifying pace there.
  No circuit length, corner counts or layout statistics: the project has no verified source for them, and none are invented. Inference computes both features with the same definitions (mean over all previous races at the circuit / of that team there). The Plackett–Luce default and grid baseline do not use this feature list and are not retrained.
- **Protocol:** unchanged (D-022): walk-forward 2017–2022, hyper-parameters and τ chosen on 2017–2019, metrics on 2020–2022.
- **Adoption rule (decided before seeing results):** adopt if the mean winner log-loss of the 16 models on 2020–2022 is no worse than before by more than 0.01 (≈ run-to-run noise on 61 races). Otherwise restore `models/algorithms_v1_backup/`. Either way, report per-model circuit sensitivity before and after.
- **Outcome (2026-11-11): ADOPTED.**
  - Accuracy: mean winner log-loss of the 16 models on 2020–2022 went from 1.5188 to 1.5252 (+0.0064, inside the 0.01 rule). The change is not an accuracy gain. KNN got worse (+0.054); the neural network (−0.064) and decision-tree regression (−0.013) got better.
  - Circuit sensitivity: on the identical 2026 grid at six circuits (Monaco, Monza, Silverstone, Bahrain, Marina Bay, Spa), the mean change in expected position across circuits rose for 13 of the 16 models.
  - Decision-tree regression went from fully insensitive to sensitive.
  - The depth-4 classification tree still never splits on a circuit feature, so it legitimately gives the same answer at every circuit.
  - Previous models: `models/algorithms_v1_backup/`; previous tables: `outputs/tables/algorithm_*_v1_backup.csv`.

### D-030 · 2026-11-11 · 2025 reference season, evidence-based driver pool, measured circuit elevation
- **Reference season:** the data ends in 2025, so the app predicts with 2025 drivers in 2025 cars. The synthetic 2026 layer (2026 grid with 2025 car proxies, imputed rookies) and the "Legends" preset are removed from the app and the API (`/api/default-grid` has no preset). No 2026 outcome or entry list is used.
- **Driver pool (`simulate.reference_pool`, `/api/driver-pool`):** a 2025 driver is selectable only with ≥ 10 career starts, ≥ 5 starts in 2025, all six driver-skill features observed (no imputed ability) and a 2025 car profile. Circuit history is not required (imputed for any first visit, as in training). Result: 20 eligible (the full 2025 field, each in their end-of-2025 seat); excluded: Jack Doohan (7 career starts). `/api/race-prediction` rejects anyone outside the pool, non-2025 seasons and more than two cars per team. Nothing is retrained: features, models and artefacts are unchanged.
- **Elevation (`src/circuit_geometry.py`):** Copernicus DEM GLO-30 sampled along each georeferenced centreline and on a terrain grid; smoothed along the track (σ 60 m). A circuit keeps its profile only if residual noise ≤ 2.5 m and the steepest 100 m grade ≤ 20%; otherwise it is drawn flat and labelled. No banking data exists, so the road cross-section is level. Visualisation only, never a model feature.
- **Race replay:** illustrative. Cars move from grid slot to predicted finishing slot; intermediate positions are not lap-by-lap predictions and are labelled so.

### D-031 · 2026-11-11 · Ten-second predicted race, lineup editor, crossover bridge, 2025 race numbers
- **Race = prediction.** The visual race lasts 10 s at 1× (one lap, time-based so it is the same at any frame rate; `frontend/src/motion.js`). It starts when the models answer; while they compute, start lights show and the cars wait on the grid. Server compute time is measured per request (`inference_ms`) and shown separately; no time guarantee is claimed. Results are revealed at the flag or on Skip to Results.
- **Overtakes are illustrative and deterministic.** The grid order is sorted into the predicted order by rounds of adjacent swaps (odd-even transposition sort). Swapping pairs in a round are disjoint, so a car is only ever alongside the one car it is passing, a full lane apart; all other cars are at least one slot (≥ 1.7 car lengths) away. No randomness: every speed and every replay shows the same race, and the finish is exactly the backend order.
- **Lineup editor.** Edits are pure functions (`frontend/src/lineup.js`) with undo/redo. Constructors are the 2025 cars only. A hypothetical pairing uses the driver's 2025 skill features with that team's 2025 car features (`simulate.build_scenario`, unchanged); the backend still rejects ineligible drivers, other seasons and more than two cars per team.
- **Stale results.** A prediction is keyed by circuit, weather, both models and the ordered (driver, car) list. Any change clears the shown result instead of dimming it.
- **Suzuka crossover.** The DEM gives both branches the same ground height. The back straight (second pass in lap order) is raised on a 220 m ramp either side so it clears the lower road by 6 m. **Assumption:** 6 m is a visual approximation, not a surveyed figure; it is labelled "approx." in the view and is never a model input. The measured profile elsewhere is unchanged.
- **2025 race numbers.** The source stores each driver's current permanent number (Norris 1, Verstappen 3). For the 2025 reference season two numbers are corrected from the 2025 entry list: Verstappen 1, Norris 4 (`simulate.REF_NUMBERS`). All other 2025 numbers equal the permanent numbers in the data.
- **Assets.** Driver photos: the lead image of each driver's own Wikipedia article, kept only when the Commons licence is free (CC BY / BY-SA / CC0 / OGL), stored as 250 px thumbnails with author and licence shown in the editor. Some photos were taken after 2025; they are portraits only, not data. No official team logos or liveries are used (trademarks); original badges and a generic car drawing stand in.
- **Verification.** 218 Python tests, 17 frontend unit tests, 100+ browser checks (headless), and a real-GPU run (`npm run e2e:gpu`).

### D-032 · 2026-11-11 · Historical driver and car pool, F1 start, timing tower, header circuit picker
- **Purpose.** APEX is an ML experimentation platform, not a 2025 simulator: any sufficiently supported driver in any sufficiently supported car, one classification + one regression model, watched as a 3D race.
- **Eligibility (`simulate.history_pool`, one rule for every era).** Driver-season: season >= the models' first training season (1981, `models/algorithms/metadata.json`; SVM trains on 1995+ for cost, its inputs are the same relative features), `eligibility()` = supported (>= 10 career starts, >= 5 that season, all six skill features observed). Car-season: >= 1981, >= 5 races that season (the last-5 window is real), all eight model car features observed. Result: 208 drivers, 537 car-seasons (65 constructors). Excluded: 423 drivers who raced only before 1981, 56 with thin evidence (e.g. Doohan). The form season is the prime season when eligible, else the best eligible season by race rating.
- **Rejected internally (`check_entries`, HTTP 422 with a reason):** drivers outside the pool, a season the driver is not eligible in, car-seasons outside the pool (pre-1981, < 5 races, missing car features), more than two entries per car-season. The 2025 audit (`reference_pool`) stays for `driver_audit` and the default grid.
- **Normalisation.** Car features are season-relative (percentiles of finishing/grid position, points share, win/podium/DNF rates over recent races, crew rating); driver skill is the teammate-network rating on one connected scale. Grid percentile can slightly exceed 1 (pit-lane starts). Known limit: DNF rates are relative to the car's own era, so a 1980s car carries that era's retirement rate.
- **Race numbers** come from the most-used number in that season's results (Senna 12 in 1988, Verstappen 1 in 2025), replacing the 2025-only override of D-031.
- **Start and race.** 10 s total: lights 0–2 s (five lights at fixed compressed intervals, out at 20%), race 2–10 s (one lap). Grid: alternating files, pole on the LEFT (visual assumption; no pole-side data), slots >= 1.7 drawn car lengths apart, merging into one file before any overtake. The timing tower uses `orderAt`, the same plan that places the cars, so it changes exactly when a car passes another.
- **Interaction.** Playback speeds removed. Clicking empty track toggles pause; a click on a car selects it; drags and the wheel never toggle and switch the camera to manual (tracking suspended until a camera button is pressed). The camera is kept above the ground by a downward ray onto the terrain and road meshes.
- **Environment.** Only dry/wet: `log_grid_x_wet` is the models' only weather feature. Other weather variables would need retraining and validation; not exposed.
- **Assets.** Logos: lead image of the constructor's own Wikipedia article (Jolpica URL), kept only if the Commons licence is free; reviewed by eye (car photos and 2026-sponsor marks rejected); logos with period sponsors are shown only for those seasons (`seasons` in `logos.json`). Credits moved from the editor to `frontend/public/credits.html`.

### D-033 · 2026-11-11 · Pre-start weather features (pre-registered before retraining)
- **Approved by the user:** retrain and validate the course models with more weather inputs.
- **Audit (1981-2025, 807 races):** Open-Meteo reanalysis at the circuit's grid cell (median 4.2 km away), pre-start window 3 h to 1 h before the start (known before the lights; the race window is excluded as future data). Coverage 100% in every era for air temperature (°C, 2 m), relative humidity (%), precipitation (mm summed over the 2 h window), wind speed (km/h, 10 m). Before 2005 the start time is assumed 14:00 local (48% of races), so those values are less precise. **Not usable:** track-surface temperature (not in the data; air temperature is NOT track temperature) and wind direction (a bearing has no consistent meaning across differently oriented circuits; the stored value is an arithmetic mean of degrees).
- **Features (v2 = v1 + 9):** race-level rain_log = log1p(precip), air_temp_c, humidity_pct, wind_kmh, and interactions log_grid × rain, rain × driver race skill, rain × car pace, (temp − 21.1) × car DNF rate, wind × car pace. Driver/car/circuit features and their cross-era normalisation are unchanged.
- **Protocol:** unchanged D-022 walk-forward; hyper-parameters and tau chosen on 2017-2019; metrics on 2020-2022; plus a chronological test on 2023-2025 computed once with everything fixed.
- **Adoption rule (decided before results), per model:** use v2 if its 2017-2019 winner log-loss is lower than v1's AND its 2020-2022 winner log-loss is not worse than v1's by more than 0.01. Otherwise keep v1 for that model. The 2023-2025 test is reported, never used to choose.
- **Backup:** previous production models copied to `models/algorithms_d023_backup/`.
- **Outcome (2026-11-11):** v2 adopted for 7 of 16 models: logistic regression, decision tree, naive Bayes, gradient boosting (classification); linear, ridge, lasso (regression). v1 kept for KNN, SVM, random forest, bagging, AdaBoost (no change: it never used the weather features), neural network (2017-2019 log-loss 1.341 -> 1.497), and the three tree/boosting regressors. Predictive quality is effectively unchanged: winner log-loss changes of -0.04 to +0.03, ROC-AUC within ±0.005, regression MAE / R² equal to 3 decimals. On the 2023-2025 test the adopted linear models improved slightly (e.g. ridge 1.156 -> 1.147, decision tree 1.433 -> 1.346); gradient boosting was slightly worse (1.211 -> 1.220). Tables: `outputs/tables/weather_model_comparison.csv`, `weather_sensitivity.csv`.
- **What actually moves predictions** (mean |change in expected position| from the dry median, 4 circuits): rain is the main effect (heavy rain 0.3-1.6 places; wet 0.13-0.83); temperature small (0.01-0.30, largest in gradient boosting and the decision tree); wind small (gradient boosting 0.18, naive Bayes 0.60, logistic 0.07, ~0 elsewhere); humidity only in the two tree models (0.2-0.55) and zero in the linear ones. Models on v1 react to rain only through the wet flag. All four variables are inputs of at least one installed model, so all four are user-adjustable; the panel shows which of the two selected models read them.
- **App:** Dry / Damp / Wet presets (training medians; damp = rain below the 0.25 mm/h wet threshold), sliders for rain 0-4 mm/h, air temperature 10-33 °C, humidity 20-95 % (≥ 45 % when raining: no wetter-air race in the data), wind 0-35 km/h, Reset. Old requests with only "weather": dry/wet still work.

### D-034 · 2026-11-11 · Driver portraits and era-specific constructor logos
- **Sources:** the en.wikipedia lead image of each driver's / constructor's own article (Jolpica link), then Wikidata P18 (person image) and P154 (logo, with P580/P582 dates) and the driver's Commons category. Only Commons files with a free licence (CC BY / BY-SA / CC0 / PD / OGL) are stored; every candidate was reviewed by eye. Official F1 or team-site photos were not used: they are copyrighted without an open licence.
- **Portraits:** 189 of 208 drivers. All are 256 px square JPEGs with one crop rule (about 17 KB each, 3.5 MB total, lazy-loaded). 20 lead images were rejected (car, helmet, crowd or too-small face) and replaced from the driver's Commons category where a clear single portrait existed (7). 19 drivers have no clear free portrait (monogram): baumgartner, pizzonia, inoue, gugelmin, modena, bernard, nannini, satoru_nakajima, sala, caffi, campos, dumfries, ghinzani, fabi, manfred_winkelhock, hesnault, giacomelli, rebaque, gilles_villeneuve.
- **Logos:** 21 of 65 constructors show a logo for at least some seasons (32 files). A logo appears only on cars of its seasons (Wikidata dates or reviewed ranges, narrowest range wins: Uralkali Haas 2021-22, Atlassian Williams 2025, McLaren wordmark 2017-2025...). Rejected: car photos, the 2026 TGR Haas mark, Mastercard McLaren, Ligier marks from after its F1 years. Other cars show the original badge.
- **Car imagery:** every constructor card shows an original team-colour car illustration. No season-specific car photos: Jolpica does not link chassis articles, so a correct photo per car-season could not be verified.
- **Credits:** `frontend/public/credits.html` (author, licence, link for every file); nothing in the editor.

### D-035 · 2026-11-12 · Constructor logo completion (visual only)
- **Coverage:** 45 of 65 constructors now show a verified logo for at least one of their seasons (was 21). 24 constructors gained a logo, and Williams, Sauber and Mercedes gained period variants. Review table and rejections: `outputs/asset_review/logos-completion/REPORT.md`. Sources and licences: `frontend/src/logos.json` and `/credits.html`.
- **Method:** Commons file search per constructor, free licences only, contact sheets reviewed by eye. Each logo's era comes from the file's own description or design date and is clipped to that constructor id's seasons. Separate identities stay separate. Picks are recorded in `LOGO_PICKS` in `src/fetch_wikidata_assets.py`; rerun with `python -m src.fetch_wikidata_assets picked`.
- **Corrections:** Honda 2006–08 had the 2024 "new H" (replaced by the Honda Racing F1 mark). BMW Sauber had the generic BMW roundel (replaced by the BMW Sauber F1 Team mark). Mercedes 2010–25 had the 2026 logo (replaced by Mercedes GP Petronas for 2010–11 and Mercedes AMG Petronas for 2012–25).
- **Rejected:** car-nose photos, user-drawn approximations, a parent company's mark for a joint team (EuroBrun), and sponsor strips too thin to read (Vodafone McLaren, Williams Martini). Where no verified mark exists, the car keeps its badge.
- **Display:** marks always follow the car season (`pickLogo` in `lineup.js`), never the driver's season. They appear in the constructor cards, grid slots (compact; falls back to the team dot), the selected-driver card and the podium cards. The timing tower and grid summary are left as text because they are too compact. The tile is now 62×46 for both logos and badges so wide wordmarks stay legible.
- **Tests:** `e2e/logos.test.mjs` (manifest, files, eras, cross-era) and `npm run e2e:logos` (browser: all 65 constructors, reassignment, podium, driver card, failed-image fallback, screenshots).

### D-036 · 2026-11-12 · Fixed 22-driver races and automatic peak constructor cars
- **Recovered implementation:** the interrupted local work already implemented `peak_cars`, `peak_default_grid`, `check_app_grid`, `/api/driver-pool` peak teams, and the search-only 22-slot editor. These were retained and verified rather than rebuilt.
- **Race format:** 22 distinct eligible drivers, 11 distinct constructors, two entries per constructor. Driver form and peak car seasons are resolved internally; explicitly conflicting seasons are rejected. Incomplete editor grids are allowed, prediction is blocked until valid. General historical simulator endpoints retain their existing car-season access.
- **Historical preservation:** 208 eligible drivers, 65 constructor identities and 537 eligible historical car-seasons through 2025. The 11-team limit applies to a race only.
- **Peak selection:** maximize the negative of 40% average within-race qualifying-grid percentile, 40% average finishing percentile and 20% constructor DNF share minus its season-field DNF share. Grid position is a qualifying-performance proxy, not lap time. Prefer eligible seasons with at least 8 races; otherwise use the team's eligible seasons and flag low evidence. Exact score ties choose the newer year. All selected ML car features come from one actual season snapshot. Diagnostics retain score components, chosen season, evidence, candidates and runner-up. No feature-wise composite or win-total selection.
- **Default lineup:** take strong peak constructors with two unique available eligible drivers who really drove that selected car-season, ordered by peak score and driver rating. Driver skill still comes from each driver's existing validated form season.
- **Editor:** full displayed-name A–Z lists and accent-insensitive name search, no era/year selectors or per-driver car dropdowns. Drag/click assignment, lower-starting-seat swap for full constructors, Move/Swap, Undo/Redo and restore retained. Frontend validation also rejects duplicate IDs and non-peak assignments.
- **Models:** no retraining or recalibration. All 10 classifier and 6 regressor artifacts accept the 22-row peak-car configuration in dry and wet weather. Constructor reassignment changes real car features while all four requested hypothetical driver/car pairings retain their separate driver features. Retrospective peak selection is used only for hypothetical scenarios; chronological evaluation and model artifacts are unchanged, with no accuracy-improvement claim.
- **Verification and launch:** production build, 43 frontend unit tests, 118 main browser checks, 79 logo checks, portrait/asset tests, real RTX 4060 GPU tests (~240 FPS), and 231 camera/visual checks. The existing current backend on port 8012 was reused; old 8010/8011 processes were left untouched. Full test results, source changes, caveats, completion table and PowerShell commands: `docs/APEX_TAKEOVER_REPORT.md`.

### D-037 · 2026-11-12 · Reproducible public repository and visual showcase
- **Authorisation:** the project owner explicitly approved repository cleanup, commits and publication to `dhruvagarwal4543/Apex-f1`. Current local files and existing Git history remain authoritative; no remote overwrite or force-push.
- **Publication bundle:** commit canonical interim/processed datasets, required model bundles/metadata, local circuit elevation/geometry, final licensed assets, scientific tables and required tests. Retain weather candidate/baseline bundles because tests and selection scripts read them. Ordinary Git is sufficient: the largest file is below 18 MiB. SHA-256 manifest verifies 409 bundled artifacts without downloading or retraining.
- **Cleanup distinction:** per-path candidates, sizes and reasons were recorded before deletion. Automated safety review rejected both deletion commands, so local copies remain. Reviewed generated files are excluded from publication; previously tracked logs are removed only from the index. Unique/uncertain legacy data and active environments are preserved locally. Full details: `docs/CLEANUP.md`.
- **Dependencies:** preserve all serialized-model package versions; add explicit Pillow 13.0.0, a tested Windows/Python 3.11 lockfile, and frontend test/showcase commands. Upgrade Vite to 6.4.6 within the existing React plugin's supported peer range to clear current development-tool advisories. Final npm audit: zero vulnerabilities. No model retraining or prediction/UI behavior change.
- **Showcase:** rewrite README around nine real optimized browser screenshots, tested PowerShell setup, concise ML explanation and proper per-source credits. Jolpica data retain CC BY-NC-SA 4.0; weather and visual assets retain their individual terms. No blanket source-code licence was invented.
- **Clean-copy verification:** new dependency environment, Git-index-only export, 268 Python tests, 43 frontend unit tests, 121 main browser checks, 79 logo checks, portrait/GLB suites and hardware GPU checks passed. Complete evidence and limitations: `docs/VERIFICATION.md`.
