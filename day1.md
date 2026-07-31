## Day 1 — July 31, 2026: From planning to a working multi-city pipeline

Converted a personal Power BI Airbnb dashboard (originally built on a static 
Kaggle/Maven dataset) into the start of a production-style, automated data pipeline.

**What I built today:**
- Planned the shift from a one-time static file to a live, refreshable data source
- Set up a GCP project + BigQuery dataset (`airbnb_raw`), with a scoped service 
  account (`BigQuery Data Editor`, `BigQuery Job User`) for automation
- Built `ingest.py`: downloads listings/calendar/reviews CSVs from Inside Airbnb, 
  cleans them with pandas, loads into BigQuery
- Tested end-to-end on New York City first — ~30K listings, ~11M calendar rows, 
  ~990K reviews loaded successfully
- Diagnosed a real data quality issue: two columns (`host_since`, `instant_bookable`) 
  were 100% null in the live source — traced this to Airbnb no longer publicly 
  exposing those fields, vs. my old static dataset which had them from years ago
- Scaled the script to 10 cities (Bangkok, Cape Town, Hong Kong, Istanbul, Mexico 
  City, New York, Paris, Rio de Janeiro, Rome, Sydney) using a config-driven design
- Caught and fixed a currency bug: prices are in local currency with no currency 
  column — cross-city averages would've silently mixed USD, EUR, THB, etc. Added 
  `currency` and `price_usd` columns with documented fixed exchange rates
- Added schema-drift protection so one city's missing column doesn't crash the 
  whole pipeline
- Verified data integrity: 0 duplicate listing IDs across all 10 cities
- Loaded ~337K listings and ~12M reviews into BigQuery (deliberately excluded the 
  full calendar table for now — would've been 150M+ rows, too much for free-tier 
  storage; noted as a v2 improvement)

**Key concepts learned:**
- IAM service accounts as "robot users" for unattended automation
- Reading/writing gzip CSVs and loading into BigQuery with pandas
- Why source schemas drift over time, and how to verify vs. assume
- Why "just average the numbers" can be a silent, serious bug across mixed sources
- Config-driven pipeline design instead of hardcoding one source
- Managing free-tier storage limits deliberately, not by accident

---
