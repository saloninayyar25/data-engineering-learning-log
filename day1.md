## Day 1 — July 31, 2026

Started converting a personal Power BI Airbnb dashboard into a production-style data pipeline.

**What I built today:**
- Set up a Python ingestion script to pull live data from Inside Airbnb (10 cities: 
  Bangkok, Cape Town, Hong Kong, Istanbul, Mexico City, New York, Paris, Rio de Janeiro, Rome, Sydney)
- Cleaned and standardized raw CSV data with pandas (price parsing, type conversion, 
  handling schema drift across cities)
- Solved a multi-currency problem — converted all prices to USD for cross-city comparison
- Loaded ~340K listings + ~12M reviews into Google BigQuery
- Automated the whole pipeline with GitHub Actions (scheduled weekly runs + manual trigger)
- Set up secure credential handling using GitHub Secrets (never committing real keys)

**Key concepts learned:**
- Service accounts & IAM roles in GCP
- BigQuery load jobs & write dispositions
- Handling schema drift in multi-source pipelines
- CI/CD basics via GitHub Actions (cron scheduling, secrets, workflow_dispatch)

**Next up:** dbt transformation layer (raw → staging → fact/dimension tables), 
then reconnecting Power BI to the modeled data.
