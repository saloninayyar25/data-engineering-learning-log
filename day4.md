## Day 4 — Resume overhaul + starting Power BI reconnection

Used the finished pipeline work to properly update my resume, then began 
reconnecting Power BI to the new BigQuery data.

**What I did today:**
- Rebuilt my resume: trimmed to 3 strongest projects, fixed formatting bugs 
  (broken link, inconsistent bullets), added a Data Engineering & Cloud skills 
  line, and wrote a professional summary
- Folded my original static Power BI dashboard and the new automated pipeline 
  into a single project entry instead of two separate ones — framed as a 
  before/after story (static one-time analysis → live automated pipeline), 
  since that shows growth rather than reading as repetition
- Quantified the combined project honestly: real dashboard stats (279K+ 
  listings, 182K+ hosts, 5.37M reviews, 32 DAX measures) alongside real 
  pipeline stats (337K+ listings, 11.9M+ reviews, 9-currency normalization) — 
  caught and fixed an earlier inflated number (30K was actually just one 
  city's count, not the true 10-city total)
- Made the GitHub repo public after verifying no credentials were ever 
  committed to its history
- Started planning the Power BI reconnection: mapped every visual from the 
  old dashboard to what's available in the new dbt marts, and identified one 
  real gap — the original "New Listings over time" lifecycle chart depended 
  on a field (`host_since`) that no longer exists in the live source. Decided 
  to drop that visual rather than force a proxy, and move forward with what 
  the new data actually supports
- Connected Power BI Desktop to BigQuery, selected the four mart tables 
  (`dim_listings`, `dim_date`, `fct_listing_snapshot`, `fct_reviews_monthly`), 
  and began verifying the relationships in Model view

**Key concepts learned:**
- ATS-friendly resume formatting (real text vs. text-boxes/tables-as-layout, 
  standard section headers, keyword placement in context)
- Why an honest, traceable number is worth more on a resume than an 
  impressive-sounding but unverifiable one
- Designing around a data gap (dropping a visual) is a legitimate engineering 
  decision, not a failure, as long as it's a deliberate choice and not an 
  oversight

**Next up:** finish verifying Power BI's auto-detected relationships, then 
rebuild the dashboard's visuals on the new data source
