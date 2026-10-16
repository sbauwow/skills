# 2026-08-21 refresh-all silent stall with stale public artifacts

## Situation
A cron operator run in `/home/stathis/atlas-tx` launched `npm run refresh:all` and the process stayed completely silent for roughly 13m54s.

## Evidence
- The tracked background process never produced stdout.
- The live process tree stayed pinned on `refresh-all.ts` plus repeated `refresh-county-month-nws-flood-alerts.ts` children instead of advancing into later steps.
- `public/cache/pipeline-health.json` was **not** rewritten; it remained on the prior `2026-08-15T18:00:19.044Z` degraded snapshot.
- During the stalled attempt, several upstream inputs did refresh:
  - `public/cache/city-open-data-tx.json`
  - `data/county-month-drought-usdm.json`
  - `data/county-month-temperature-open-meteo.partial.json`
  - `data/county-month-precipitation-open-meteo.json`
- Core downstream public artifacts did **not** refresh:
  - `public/cache/pipeline-health.json`
  - `public/cache/city-open-data-ranked-tx.json`
  - `public/cache/roadmap-open-data-botnet.json`
  - `public/cache/surface-water-quality-tx.json`
  - `public/cache/twdb-hydrology-tx.json`

## Interpretation
Treat this as an **incomplete run**, not a fresh degraded refresh. The authoritative production snapshot remained stale, so operator reports must judge production readiness from the unchanged public artifacts rather than from the partially refreshed upstream `data/` files.

Because the stale snapshot still included a non-CID fast-lane failure (`refresh-county-month-temperature`) and the fast-lane public artifacts were far beyond the 12h freshness threshold, the operator-facing report should begin with `ALERT` and explicitly say the core public artifacts are not fresh enough for production use.

## Cleanup pattern
After collecting evidence, kill the tracked `refresh-all` background process and any orphan `refresh-county-month-nws-flood-alerts.ts` children so later scheduled refreshes do not overlap.
