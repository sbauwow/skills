# 2026-08-23 refresh-all stall with stale analytics artifacts

## Summary
A scheduled `npm run refresh:all` run in `/home/stathis/atlas-tx` stayed completely silent for about 11m51s, remained pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children, and never regenerated `public/cache/pipeline-health.json`.

## Evidence pattern
- Parent process: `npm run refresh:all` / `tsx scripts/refresh-all.ts`
- Child-process frontier never advanced into later downstream steps; live children stayed on repeated `refresh-county-month-nws-flood-alerts.ts`
- `public/cache/pipeline-health.json` remained at `2026-08-15 13:13:43 -0500`
- Downstream public artifacts such as `public/cache/surface-water-quality-tx.json` and `public/cache/analytics/*` remained ~8 days old
- Some upstream inputs refreshed during the stalled attempt:
  - `public/cache/city-open-data-tx.json`
  - `data/county-month-drought-usdm.json`
  - `data/county-month-temperature-open-meteo.partial.json`
  - `data/county-month-nws-flood-alerts.json`
  - `data/county-month-precipitation-open-meteo.json`
  - `data/county-month-streamflow-usgs.json`

## Interpretation
This is an incomplete refresh, not a fresh degraded run. A refreshed `data/county-month-nws-flood-alerts.json` does **not** prove the pipeline advanced past the repeated NWS-child stall pattern; the key signals are that the health snapshot was not rewritten and downstream production-facing artifacts stayed stale.

## Analytics artifact note
The ingest catalog still names `public/cache/analytics-history.json` as the expected analytics output, but production-facing analytics artifacts currently live under `public/cache/analytics/` (for example `county-history.json`, `county-movers.json`, `pressure-risk-scatter.json`, `source-freshness.json`). For production freshness checks, inspect those real files on disk rather than relying on the catalog alias path.
