## Day 3 - Verifying automation, starting dbt

Short on time today — logging progress and next steps rather than a full write-up.

**In progress:**
- Manually triggering the GitHub Actions workflow to verify Day 2's automation 
  actually works end-to-end (auth + BigQuery row updates), not just "job succeeded"
- Starting dbt setup (`dbt-bigquery`) to build a staging layer on top of the raw 
  BigQuery tables — goal is to move the currency conversion logic (`price_usd`) 
  out of `ingest.py` and into a proper dbt staging model, where transformation 
  logic belongs instead of the raw ingest layer

**Next up:** finish workflow verification, get `stg_listings.sql` running with 
`not_null`/`unique` tests on `listing_id`.
