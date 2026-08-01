## Day 2 - Automating with GitHub Actions

Made the pipeline run on its own, without my laptop, on a schedule.


**What I built today:**
- Set up `.gitignore` to keep credentials out of version control
- Refactored credential handling to work both locally (file) and in CI (secret)
- Pushed the project to a private GitHub repo
- Added the service account key as a GitHub Secret (`GCP_SA_KEY`)
- Wrote a GitHub Actions workflow (`.github/workflows/ingest.yml`) with a weekly 
  cron schedule + manual trigger option

**Key concepts learned:**
- Why secrets must never be committed, and how CI systems inject them safely at runtime
- Cron syntax for scheduling
- GitHub Actions workflow structure (checkout → setup → install → run)
- The real meaning of "production": doesn't depend on a person manually running it

**Next up:** finish testing the automated workflow run, then move to a dbt 
transformation layer (raw → staging → fact/dimension tables), and eventually 
reconnect Power BI to the modeled data instead of the raw import.
