## Day 3 — dbt transformation layer

Turned the raw ingested BigQuery tables into a proper dimensional model, so 
downstream reporting connects to clean marts instead of raw data.

**What I built today:**
- Installed dbt-bigquery, initialized the project, and connected it to BigQuery
- Fixed dbt's default config: wrong region (defaulted to US, needed to match 
  where the actual data lives) and wrong output dataset (dbt's transformed 
  tables need to live separately from raw ingested data, not mixed together)
- Built a staging layer (`stg_listings`, `stg_reviews`) as a thin, renamed 
  pass-through of the raw source tables
- Built a star schema on top: `dim_listings`, `dim_date` (dimensions) and 
  `fct_listing_snapshot`, `fct_reviews_monthly` (facts) — using `ref()` between 
  models so dbt auto-resolves build order instead of me sequencing it manually
- Added automated data quality tests: uniqueness + not-null checks on key 
  columns, foreign key not-null checks on both fact tables
- Investigated a real data quality finding: ~17% of listings have no active 
  price, consistent across all 10 cities (ruled out a city-specific bug by 
  checking the breakdown first). Documented it and downgraded that specific 
  test from a hard failure to a warning, since it reflects genuine source 
  data rather than a pipeline defect

**Debugging along the way:**
- Spent a long stretch chasing a broken `{{ source(...) }}` syntax that kept 
  turning into `{ { source(...) } }` on save — tried several wrong guesses 
  (manual typing, auto-closing brackets, formatter extensions) before finding 
  the real cause: global VS Code settings (`formatOnSave`/`formatOnPaste`) 
  with no SQL-specific override, silently reformatting the file
- A YAML indentation mistake caused a test's `severity: warn` config to be 
  read as if it were a test name — fixed by correctly nesting it under `config:`

**Key concepts learned:**
- `source()` vs `ref()` and why the distinction lets dbt build the dependency 
  graph automatically
- Star schema design: separating facts (measurable events) from dimensions 
  (descriptive attributes)
- Investigate-before-fixing as a habit — the null price rate turned out to be 
  real data, not a bug, and would've been wrong to just delete or fake

**Result:** all 6 models build cleanly (`dbt run`), tests pass with one 
intentional warning (`PASS=5, WARN=1, ERROR=0`)

---

