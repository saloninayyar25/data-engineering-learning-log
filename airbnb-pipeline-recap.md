# Airbnb Production Pipeline — Project Recap

A systematic record of everything built so far: **what** was done, **how**, and **why**.

---

## Step 1 - Planning the production upgrade

**What:** Defined the overall plan to turn a static Power BI dashboard into a production system: automated data pipeline → cloud warehouse → transformation layer → refreshed dashboard, tied to a real business problem.

**How:** Mapped out the phases - fix the data layer first, then automation, then transformation, then reconnect Power BI.

**Why:** A dashboard built on a manually-imported file can never be automated or trusted to stay current. Production systems need a live, repeatable pipeline behind them, not a one-time import.

---

## Step 2 - Choosing the data source

**What:** Selected **Inside Airbnb** (insideairbnb.com) as the live data source, using the same 10 cities as the original dashboard: Bangkok, Cape Town, Hong Kong, Istanbul, Mexico City, New York, Paris, Rio de Janeiro, Rome, Sydney.

**How:** Learned the Inside Airbnb URL pattern:
```
https://data.insideairbnb.com/{country}/{region}/{city}/{snapshot_date}/data/{filename}
```
Collected `listings.csv.gz`, `calendar.csv.gz`, `reviews.csv.gz` URLs for all 10 cities (each city has its own snapshot date).

**Why:** The original dashboard used a static, frozen Kaggle/Maven dataset — it could never be automated. Inside Airbnb publishes real, periodically-refreshed data, making it possible to build an actual repeatable pipeline. Reusing the same 10 cities preserves the ability to compare "before vs. after" for portfolio storytelling.

---

## Step 3 - Setting up Google BigQuery (the cloud warehouse)

**What:** Created a GCP project (`airbnb-dashboard-prod`), a BigQuery dataset (`airbnb_raw`), a service account (`airbnb-pipeline-bot`) with scoped IAM roles, and downloaded its JSON key.

**How:**
- Created dataset `airbnb_raw` (region: `asia-south2`, chosen for proximity to India)
- Created a service account and granted only `BigQuery Data Editor` + `BigQuery Job User` (least-privilege, not full admin)
- Generated a JSON key (`gcp-key.json`) — the credential Python uses to authenticate as this "robot user"

**Why:** BigQuery's free tier (1TB queries/month, 10GB storage, no time limit) is the most generous and easiest cloud warehouse to learn on. A dedicated service account (rather than personal login) is how production systems authenticate — it works unattended, without a human logging in, which is essential for automation later.

---

## Step 4 - Setting up the Python environment

**What:** Created a local project folder (`airbnb-pipeline`), a Python virtual environment, and installed required libraries.

**How:**
```
python -m venv venv
venv\Scripts\activate
pip install pandas requests google-cloud-bigquery python-dotenv
```
Stored `gcp-key.json` locally and referenced it via a `.env` file.

**Why:** A virtual environment isolates this project's dependencies from the rest of the system, avoiding version conflicts — standard practice before writing any real pipeline code.

---

## Step 5 - Building the ingestion pipeline (`ingest.py`)

### 5a. Download raw files
**What:** Script downloads `listings.csv.gz` and `reviews.csv.gz` for each city (calendar deliberately excluded — see below).
**How:** `requests.get()` per URL, saved into a `raw_data/<city>/` "landing zone."
**Why:** Keeping untouched raw copies before transforming is standard practice — allows debugging/reprocessing without re-downloading.

### 5b. Inspect the data
**What:** Loaded each file with pandas, checked shape and column names before writing any cleaning logic.
**How:** `pd.read_csv(..., compression="gzip")`, then `.shape`, `.columns`, `.head()`.
**Why:** Real datasets are always messier than expected — inspecting first avoids building cleaning logic on wrong assumptions.

### 5c. Clean the data
**What:** Trimmed `listings` from 90 columns down to ~23 relevant ones; converted `price` from text (`"$150.00"`) to a proper float; parsed dates; tagged every row with `city` and `snapshot_date`.
**How:** Column selection (`keep_cols`), `.str.replace()` + `pd.to_numeric()` for price, `pd.to_datetime()` for dates.
**Why:** Every extra column is unnecessary storage/clutter — only keep what feeds an actual metric. Text prices can't be used in calculations, so they must become real numbers. `snapshot_date` lets you tell pipeline runs apart later once it's automated and running repeatedly.

**Bug caught:** `host_since` and `instant_bookable` came back 100% null in the *live* data — traced to Airbnb no longer publicly displaying those fields (the old static Maven dataset had them because it was scraped years earlier, when Airbnb still showed them). Dropped both columns rather than keep dead data.

### 5d. Load into BigQuery
**What:** Uploaded cleaned DataFrames into BigQuery tables (`listings`, `reviews`).
**How:** `google-cloud-bigquery`'s `load_table_from_dataframe()`, using `WRITE_TRUNCATE` (replace table each run, not append).
**Why:** `WRITE_TRUNCATE` keeps the pipeline idempotent while testing — re-running doesn't pile up duplicate data. (Longer-term, history/append strategy is a deliberate future upgrade, not done yet.)

### 5e. Scale to all 10 cities + handle currency
**What:** Refactored into a config-driven, multi-city pipeline (`CITIES` dict with per-city URL path, snapshot date, currency). Added `price_usd` (converted using fixed, documented exchange rates) alongside the original local `price` + `currency`.
**How:** Looped through all 10 cities; added `safe_select_columns()` to tolerate schema drift (a city missing a column doesn't crash the pipeline); added a `time.sleep(1)` between downloads to avoid rate-limiting; verified 0 duplicate listing IDs across cities.
**Why:** Averaging "price" across cities without currency conversion would silently mix USD, EUR, THB, etc. — a real, easy-to-miss bug with business impact. Schema-drift protection and rate-limit delays are standard defensive practices once a pipeline touches multiple external sources instead of one.

**Deliberate scope decision:** the `calendar.csv.gz` file (365 rows per listing) was excluded from all 10 cities — at ~11M rows per city, 10 cities would mean 100–150M+ rows, risking the BigQuery free-tier storage limit. Documented as a planned v2 improvement (to be pre-aggregated, not loaded raw).

**Result:** ~337,553 listings and ~11.9 million reviews loaded across all 10 cities, currency-converted and verified.

---

## Step 6 - Automating with GitHub Actions

**What:** Made `ingest.py` run automatically on a schedule, on GitHub's servers, without the local machine needing to be on.

**How:**
1. **`.gitignore`** — excluded `venv/`, `raw_data/`, `.env`, `gcp-key.json` from version control (credentials must never be committed)
2. **Credential handling refactor** — `get_bigquery_client()` now checks for an environment variable (`GCP_SA_KEY`) first (for CI), falling back to the local key file (for local runs) — same script works in both places
3. **Pushed to a private GitHub repo** (`airbnb-pipeline`)
4. **Added `GCP_SA_KEY` as a GitHub Actions secret** — the key's JSON content, injected securely at runtime, never stored in the repo
5. **Wrote `.github/workflows/ingest.yml`** — a scheduled workflow (`cron: "0 3 * * 1"`, every Monday 3 AM UTC) plus a manual trigger (`workflow_dispatch`)
6. **Created `requirements.txt`** so GitHub's clean environment knows what to install

**Why:** A pipeline that only runs when a person manually types a command isn't "production" — it depends on a human remembering to run it. Scheduled, unattended execution is what makes a pipeline genuinely automated. Secrets management (never committing real keys) is a non-negotiable security practice.

**Debugging along the way (genuinely useful real-world experience):**
- `requirements.txt` was missing entirely at first → added it
- Initial `pip freeze` output pinned exact versions tied to the local Windows/Python setup → caused a `numpy` version conflict on GitHub's Ubuntu/Python 3.11 environment → fixed by simplifying to unpinned, direct dependencies only
- `pyarrow` was missing (installed locally as a transitive dependency, but not explicitly declared) → BigQuery's `load_table_from_dataframe()` requires it directly → added explicitly

**Result:** Workflow run **#4 succeeded end-to-end** — all 10 cities processed, 0 duplicate IDs, data loaded into BigQuery, entirely on GitHub's infrastructure, in ~4 minutes, with zero manual intervention.

---

## Step 7 - Transformation layer with dbt (in progress)

**What:** Introducing a modeling layer between raw BigQuery tables and the eventual Power BI report, so Power BI connects to clean, purpose-built tables instead of raw data.

**Planned structure (star schema):**
- `stg_listings`, `stg_reviews` — staging models (thin, renamed pass-through of raw sources)
- `dim_listings` — one row per listing (location, property, host info)
- `dim_date` — calendar dimension for time-based analysis
- `fct_listing_snapshot` — price, availability, occupancy, revenue facts
- `fct_reviews_monthly` — reviews aggregated by listing + month

**How (so far):**
1. Installed `dbt-bigquery`
2. Ran `dbt init airbnb_dbt` — set up authentication (service account, same `gcp-key.json`), initially defaulted to region `US` and dataset `airbnb_raw`
3. **Corrected the config** in `~/.dbt/profiles.yml`:
   - `location: US` → `location: asia-south2` (must match where the raw data actually lives, to avoid cross-region query errors)
   - `dataset: airbnb_raw` → `dataset: airbnb_dbt` (keeps dbt's transformed output separate from raw ingested data, avoiding confusion/collisions)
4. Verified with `dbt debug` → **"All checks passed!"**
5. **Next (in progress):** writing `models/staging/sources.yml` (pointing dbt at the raw `listings`/`reviews` tables) and the first two staging models (`stg_listings.sql`, `stg_reviews.sql`)

**Why dbt (vs. Power Query dataflows or raw SQL scripts):** free, open-source, industry-standard for analytics engineering, version-controllable, and keeps transformation logic portable/testable — rather than locking cleaning logic inside Power BI itself.

**Why separate raw vs. transformed datasets:** raw ingested data (`airbnb_raw`) and modeled/transformed data (`airbnb_dbt`) serving different purposes — mixing them in one dataset risks table name collisions and makes it unclear which tables are "source of truth" vs. "derived."

---

## Current state - what's real and working right now

✅ Live data source (Inside Airbnb, 10 cities)
✅ Cloud data warehouse (BigQuery, `airbnb_raw` dataset)
✅ Ingestion pipeline (`ingest.py`) — download, clean, currency-convert, load
✅ Full automation (GitHub Actions, weekly schedule + manual trigger, secrets properly managed)
✅ dbt project initialized and connected correctly

## Not yet done
- ⏳ dbt staging models (`stg_listings`, `stg_reviews`) — next immediate step
- ⏳ dbt fact/dimension models (`dim_listings`, `fct_listing_snapshot`, `fct_reviews_monthly`, `dim_date`)
- ⏳ Calendar/occupancy data — deliberately deferred, needs pre-aggregation before loading
- ⏳ Reconnecting Power BI to the new modeled data (currently still on the old static file)
- ⏳ Row-level security, performance tuning, deployment pipelines (dev/test/prod), documentation/README
- ⏳ Tying the dashboard back to a specific business question/KPI set

---

## Key concepts learned across the project
- IAM service accounts & least-privilege access for automation
- Handling schema drift across multiple data sources
- Currency/unit consistency issues in cross-source aggregation
- Secrets management in CI/CD (never commit credentials)
- GitHub Actions workflow structure (checkout → setup → install → run) and cron scheduling
- Debugging environment-mismatch issues (pinned vs. unpinned dependencies, transitive dependencies)
- Star schema design (facts vs. dimensions) and why raw ≠ modeled data
- Deliberate scoping decisions under free-tier constraints (excluding calendar data for now)
