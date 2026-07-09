[← Back to triad showcase](index.html)

---

# Triad: Development Log

A running record of every meaningful change, decision, and discovery
made during the build. Updated every session.

Format: `[YYYY-MM-DD HH:MM] :: TYPE :: Description`

Types: `DECISION` `BUILD` `DISCOVERY` `FIX` `REFACTOR` `TEST` `DOCS` `INFRA`

---

## May 28, 2026

**[2026-05-28 10:14] :: DECISION :: Project scoped as BYU-Idaho Senior Project**
Decided to build a fitness data pipeline as the senior project deliverable.
Core requirement: must include an R or Python package. Chose Python.
Motivation: three fitness apps (Zepp, Cal AI, Hevy) each track one variable
in isolation. No tool connects them. This is the gap.

**[2026-05-28 10:31] :: DECISION :: Research question defined**
Settled on: *"Can sleep quality and prior training load predict optimal daily
workout type and caloric targets for an individual athlete?"*
Rejected a broader framing: needed something measurable and answerable
within a semester timeline.

**[2026-05-28 10:45] :: DECISION :: Project named "triad"**
Named for the three inputs: sleep, nutrition, training.
One word, lowercase, clean as a Python package name.
Full title for proposal: "Triad: An Adaptive Fitness Intelligence Pipeline."

**[2026-05-28 11:02] :: DISCOVERY :: Data source validation: Hevy**
Confirmed Hevy has an official REST API at `api.hevyapp.com/v1`.
Requires Pro subscription. API key accessible at `hevy.com/settings?developer`.
Paginated endpoint: `GET /v1/workouts?page=N&pageSize=10`.
API key in hand: confirmed working.

**[2026-05-28 11:18] :: DISCOVERY :: Data source validation: Cal AI**
Cal AI API at `docs.calai.app` is a food *scanning* API for developers: not a personal history retrieval endpoint. Cannot pull logged meals via API.
Workaround: Cal AI added a PDF export feature (Summary Report).
PDF becomes the ingestion method. Viable.

**[2026-05-28 11:35] :: DISCOVERY :: Data source validation: Zepp/Amazfit**
No official public API for Zepp. Reverse-engineered APIs exist but are fragile.
Clean workaround: Zepp syncs sleep data to Apple Health natively on iPhone.
Apple Health becomes the bridge. For automation: Health Auto Export app
pushes Apple Health data daily to a REST endpoint via scheduled HTTP POST.
This enables a fully automated pipeline without manual exports.

**[2026-05-28 11:52] :: DECISION :: Architecture collapsed from 3 sources to 2 paths**
On iPhone, Apple Health serves as hub for Zepp (sleep) and Cal AI (nutrition).
Final ingestion paths:
  Path 1: Apple Health XML export → parses sleep + nutrition
  Path 2: Hevy REST API → pulls workouts directly
Cal AI PDF export supplements Apple Health when Apple Health lacks nutrition data.

**[2026-05-28 14:30] :: DOCS :: Senior project proposal written**
One-page PDF generated: `ascona_jonathan_seniorproject.pdf`
Sections: Personal Background, Project Background, Domain to Investigate,
Proposed Deliverables.
Deliverables finalized as three bullet points covering:
  1. triad Python package (ingest + harmonize + features)
  2. Recommendation engine (LLM + feature engineering hybrid)
  3. Cloud pipeline + Plotly dashboard

---

## May 29, 2026

**[2026-05-29 09:15] :: DECISION :: Database design: date as universal join key**
All three sources produce daily data. Date is the natural join key.
`daily_summary` becomes the hub table: one row per calendar day.
Two-layer schema: raw ingestion tables + derived aggregation tables.

**[2026-05-29 09:40] :: BUILD :: ERD designed (8 tables)**
Raw ingestion layer:
  `sleep_records`, `nutrition_logs`, `calai_meals`,
  `workout_sessions`, `exercise_sets`
Derived layer:
  `daily_summary`, `user_features`, `user_profile`, `recommendations`
ERD diagrammed in FigJam via Figma MCP tool.

**[2026-05-29 10:05] :: BUILD :: hevy_ingest.py script written**
First working script. Paginates Hevy API, extracts workout sessions and
exercise sets, loads into local SQLite via `triad.db`.
Note: sandbox environment cannot reach `api.hevyapp.com` (allowlist restriction).
Script delivered for local execution instead.

**[2026-05-29 14:20] :: DECISION :: Proposal deliverable section trimmed**
Original 6 deliverables condensed to 3 at user request.
More concise and appropriate for a one-page document.

---

## June 1, 2026

**[2026-06-01 10:00] :: BUILD :: Real data ingestion: Hevy CSV**
User uploaded `workouts.csv` (Hevy export) and `apple_health.zip`.
Hevy CSV structure: flat, one row per set. 2,308 rows total.
Reconstructed sessions by grouping on `(title, start_time)`.
Result: 167 workout sessions, 2,256 exercise sets (Sep 2025: May 2026).

**[2026-06-01 10:22] :: BUILD :: Real data ingestion: Apple Health XML**
`export.xml` inside zip: 314MB. Too large for memory loading.
Implemented `xml.etree.ElementTree.iterparse()` streaming parser.
Called `elem.clear()` after each record to keep memory flat.

**[2026-06-01 10:35] :: DISCOVERY :: Cal AI barely wrote to Apple Health**
Critical finding: Cal AI only wrote water logs (16 records) and 1 body mass
record to Apple Health. Zero nutrition data (calories, macros) written.
Apple Health is NOT viable for Cal AI nutrition data.
Fallback: Cal AI PDF export required for all nutrition ingestion.

**[2026-06-01 10:41] :: DISCOVERY :: Zepp sleep stage breakdown confirmed**
Zepp writes full stage data to Apple Health:
  `HKCategoryValueSleepAnalysisAsleepDeep` → "deep"
  `HKCategoryValueSleepAnalysisAsleepREM`  → "rem"
  `HKCategoryValueSleepAnalysisAsleepCore` → "core"
  `HKCategoryValueSleepAnalysisAwake`      → "awake"
4,584 sleep stage records available. Full stage breakdown intact.

**[2026-06-01 10:55] :: DECISION :: Sleep date assignment logic**
Problem: sleep starting at 1am Wednesday belongs to Tuesday's night.
Rule implemented: if start time >= 18:00, assign to that calendar date.
If start time < 18:00, assign to the previous calendar date.

**[2026-06-01 11:10] :: BUILD :: Full ingestion pipeline: triad.db created**
All three sources loaded into SQLite. Tables populated:
  `sleep_records`:     4,584 rows
  `nutrition_logs`:       15 rows (MyNetDiary, Jul: Sep 2025 only)
  `workout_sessions`:    167 rows
  `exercise_sets`:     2,256 rows
  `daily_summary`:       271 rows

**[2026-06-01 11:25] :: DISCOVERY :: Critical data gap: 4 days with all three sources**
After joining all three sources by date: only 4 days have sleep + nutrition
+ workout data simultaneously. Not enough to train any model.
Root cause: MyNetDiary nutrition data ends Sep 2025; workouts start Sep 2025.
Nutrition logging consistency identified as the primary project constraint.

---

## June 2, 2026

**[2026-06-02 09:10] :: BUILD :: Cal AI PDF parser written**
User provided Cal AI Summary Report PDF (Nov 29, 2025: May 27, 2026).
Used `pdfplumber` to extract text. Regex pattern for food rows:
  `^(.+?)\s+(\d+)\s+(\d+)g\s+(\d+)g\s+(\d+)g\s+(\d+)g\s+(\d+)g\s+(\d+)mg\s+(\d+:\d+(?:am|pm))$`
Parsed: 236 individual meals across 59 days. Daily totals computed.

**[2026-06-02 09:35] :: BUILD :: nutrition_logs + calai_meals tables populated**
Old MyNetDiary records cleared. Cal AI data loaded.
`calai_meals`:    236 rows (individual food items)
`nutrition_logs`: 236 rows (daily macro totals)

**[2026-06-02 09:50] :: BUILD :: daily_summary rebuilt with Cal AI nutrition**
Complete days (has_all_three = 1) jumped from 4 → 15.
15 days with sleep + nutrition + workout all present.
Date range of overlap: Nov 2025: May 2026.

**[2026-06-02 10:30] :: DECISION :: Recommendation approach: hybrid LLM + stats**
Pure stats rejected: 15 complete days insufficient to train a model (need 40+).
Pure LLM rejected: no access to personal numeric data.
Hybrid chosen: stats compute features → LLM reasons from them.
Goal text ("build muscle and lose fat") passed verbatim to LLM: no parsing.

**[2026-06-02 11:00] :: DECISION :: User profile stores free-text goal**
User's fitness goal stored as unstructured text in `user_profile.goal_text`.
Injected directly into the LLM prompt. Supports any phrasing:
"build muscle and lose fat", "train for a 5k", "recover from overtraining".
No translation layer: LLM interprets naturally.

---

## June 3, 2026

**[2026-06-03 09:00] :: BUILD :: Package structure scaffolded**
Created `triad/` Python package with 5 submodules:
  `db/`, `ingest/`, `transform/`, `profile/`, `recommend/`
`pyproject.toml` written. Package installable via `pip install -e .`

**[2026-06-03 09:30] :: BUILD :: triad/db/store.py**
All SQLite read/write operations centralized here.
`connect()` context manager handles commit/rollback automatically.
`init_schema()` creates all 8 tables with `CREATE TABLE IF NOT EXISTS`.
Key helpers: `get_daily_summary()`, `get_daily_range()`, `get_features()`,
`get_profile()`, `upsert_features()`, `upsert_profile()`, `save_recommendation()`.

**[2026-06-03 10:00] :: BUILD :: triad/transform/features.py**
Feature engineering layer. Five core functions:
  `sleep_debt()`: rolling 7-day cumulative deficit vs. target
  `recovery_score()`: 0: 100 weighted composite (deep 45%, REM 35%, core 20%)
  `training_load()`: total volume over a date window
  `load_change_pct()`: week-over-week volume change percentage
  `caloric_deficit()`: avg daily deficit over rolling window
  `protein_average()`: avg daily protein, skipping unlogged days
  `muscle_fatigue()`: per-muscle-group volume in last 48hrs
`compute_features()` main entry point: fetches rolling windows, computes all
features, saves to `user_features` table.
`interpret_features()` translates raw numbers into severity-labeled signals.

**[2026-06-03 10:45] :: BUILD :: triad/profile/user.py**
`calculate_tdee()`: Mifflin-St Jeor BMR × activity multiplier
`calculate_protein_target()`: 0.8: 1.0g/lb based on goal text parsing
`create_profile()`: saves biometrics + auto-calculates targets
For Jonathan's profile: TDEE = 3,002 cal/day, protein target = 178g/day.

**[2026-06-03 11:20] :: BUILD :: triad/recommend/engine.py**
`build_prompt()`: constructs structured context prompt with user profile,
computed signals with severity levels, and muscle fatigue map.
`call_llm()`: Anthropic API call, strips markdown fences, parses JSON.
`recommend()`: main entry point: compute features → build prompt → call API
→ save to recommendations table → return parsed dict.
`format_recommendation()`: pretty-prints output for terminal.

**[2026-06-03 12:00] :: BUILD :: triad/ingest/ modules written**
`hevy.py`: REST API puller + CSV fallback (`ingest_hevy_csv()`)
`apple_health.py`: streaming XML parser, handles .zip input automatically
`calai.py`: PDF text extraction + regex food row parser, idempotent
`summarize.py`: `build_daily_summary()` aggregates all three sources by date

**[2026-06-03 12:30] :: BUILD :: triad/__init__.py: clean public API**
Top-level exports: `setup()`, `sync()`, `recommend()`, `today()`,
`create_profile()`, `get_profile()`, `update_weight()`, `format_recommendation()`
Public API designed so a user never needs to import submodules directly.

---

## June 4, 2026

**[2026-06-04 09:00] :: TEST :: 22 unit tests written and passing**
`tests/test_features.py`: 17 tests covering all feature engineering functions:
  sleep_debt: perfect, deficit, surplus, empty cases
  recovery_score: perfect, poor, no sleep, bounded at 100
  training_load: sum, empty
  load_change_pct: increase, no prior (divide by zero guard)
  caloric_deficit: on target, under, no logs
  protein_average: normal, skips unlogged days
`tests/test_profile.py`: 5 tests covering TDEE and protein target calculations.
All 22 passing on Python 3.11.

**[2026-06-04 09:45] :: TEST :: Real data validation: April 4, 2026**
Ran `compute_features("2026-04-04")` against real triad.db.
Output revealed cross-signal pattern:
  sleep_debt_7d:      -30.78 hrs  [CRITICAL]
  recovery_score:      95.4/100   [EXCELLENT: that specific night]
  training_load_7d:    77,440 lbs
  load_change_pct:    +110.2%     [OVERREACHING]
  caloric_deficit_3d: -1,429 kcal [LARGE DEFICIT]
  protein_avg_7d:      102g       [LOW: target 178g]
First successful end-to-end feature computation on real data.

---

## June 10, 2026

**[2026-06-10 10:00] :: INFRA :: GitHub repository created**
Repo: `github.com/jonathanascona/triad`
First PAT (fine-grained) lacked repo creation permissions.
Second PAT (classic, `repo` scope) used successfully.
Initial commit: 22 files, 2,618 insertions.
Commit hash: `972b784`

**[2026-06-10 10:15] :: DOCS :: README.md written**
Covers: the problem, architecture diagram, data sources table, installation,
quick start with sample output, database schema overview, package structure,
feature engineering table, configuration, built with section.

**[2026-06-10 10:30] :: DOCS :: docs/schema.md written**
Full column-level documentation for all 8 tables with types and descriptions.
Includes ERD relationship diagram in ASCII and row count table.

**[2026-06-10 10:45] :: DOCS :: docs/how_it_was_built.md written**
Technical development journal covering: data source validation, schema design
decisions, XML streaming approach, sleep date assignment logic, gap discovery,
feature engineering rationale, hybrid LLM approach, and first validation results.

**[2026-06-10 11:00] :: INFRA :: GitHub Actions CI added**
`.github/workflows/ci.yml`: runs 22 tests on push and PR.
Matrix: Python 3.11 and 3.12. Includes import smoke test.
Second commit: `26e2ad5`

**[2026-06-10 11:15] :: DOCS :: docs/origin_story.md written**
Full narrative from initial frustration → data source validation → gap discovery
→ hybrid recommendation approach → first real validation on April 4th data.
Written as a story, not a spec.

**[2026-06-10 11:30] :: DOCS :: docs/erd.md written**
Complete ERD as Mermaid diagram (renders natively on GitHub).
Includes ASCII data flow diagram and row counts table with date ranges.

**[2026-06-10 14:00] :: DOCS :: devlog.md, security_log.md, prompt_engineering_journal.md added**
Three new documentation files backfilled from project start.
All pushed to GitHub in a single commit.


---

## June 10, 2026 (continued)

**[2026-06-10 14:30] :: DECISION :: Removed recommend/ from package**
Identified architectural flaw: package should be a pure data normalization
layer. Recommendation engine has no business being inside the package.
The intelligence layer belongs in the app. Package's job ends at
`daily_summary` + `user_features`. App decides what to do with that data.

**[2026-06-10 14:45] :: DECISION :: Closed source confirmed**
Package will not be published to PyPI. It is a proprietary data layer
for the triad iOS app only. No public distribution.

**[2026-06-10 15:00] :: REFACTOR :: connector architecture built**
Replaced `triad/ingest/` with `triad/connectors/`: a proper plug-in
interface for any data source.
`base.py`: Abstract `BaseConnector` class with `connect()`, `validate()`,
`pull()`. Defines `ConnectorResult`, `SleepRecord`, `NutritionRecord`,
`WorkoutRecord`, `ExerciseSetRecord` as the standard output types.
Three connectors implement it:
  `HevyConnector`         API_BASED  provides: WORKOUTS
  `AppleHealthConnector`  FILE_BASED provides: SLEEP + NUTRITION
  `CalAIConnector`        FILE_BASED provides: NUTRITION
Adding any future source (WHOOP, Garmin, Oura) = one new file.

**[2026-06-10 15:15] :: DECISION :: Backend architecture: Option 3**
iOS app will NOT run Python on device and will NOT reimplement in Swift.
Decision: FastAPI backend on Railway, Supabase for database + auth.
Three repos: triad (done), triad-api (planned), triad-ios (planned).
Auth: Apple Sign In only via Supabase Auth. No username/password.

**[2026-06-10 15:30] :: DOCS :: backend_architecture.md written**
Full blueprint for triad-api including:
  8 v1 API endpoints, file structure, Supabase schema with RLS,
  onboarding API sequence, privacy model, deployment steps.
Added to README nav bar.

**[2026-06-10 15:45] :: DOCS :: app_vision.md written**
Full iOS app vision including:
  6 onboarding screens with wireframes, auth flow per source,
  daily view layout, Hevy write-back feature, privacy model,
  future connector roadmap (WHOOP, Oura, Garmin, MyFitnessPal, etc.)

**[2026-06-10 16:00] :: BUILD :: Package run end to end in sandbox**
Full execution sequence validated against real data:
  setup()                    ✓ Schema created
  create_profile()           ✓ TDEE 3,002 cal, protein 178g
  AppleHealthConnector       ✓ 4,626 sleep records loaded
  CalAIConnector             ✓ 74 nutrition days loaded
  Hevy CSV                   ✓ 167 sessions, 2,308 sets loaded
  build_daily_summary()      ✓ 293 days, 19 complete
  compute_features()         ✓ All signals correct
  interpret_features()       ✓ Severity labels correct
  build_prompt()             ✓ 2,608 character prompt
  Response parsing           ✓ JSON parses cleanly
  format_recommendation()    ✓ Full output rendered
  save_recommendation()      ✓ Persists to DB
  get_latest_recommendation()✓ Retrieves correctly
Only untested: live Anthropic API HTTP call (requires API key).
All 22 unit tests passing.

**[2026-06-10 16:15] :: TEST :: 22/22 tests passing on Python 3.12**
Full pytest run clean. All feature engineering and profile tests green.
