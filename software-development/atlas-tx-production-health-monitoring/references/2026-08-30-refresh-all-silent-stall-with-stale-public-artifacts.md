# 2026-08-30 refresh-all silent stall with stale public artifacts

## Summary
A scheduled `npm run refresh:all` invocation in `/home/stathis/atlas-tx` stayed completely silent for about 12 minutes, never advanced beyond repeated `refresh-county-month-nws-flood-alerts.ts` child processes, and never rewrote `public/cache/pipeline-health.json`. The run had to be killed and the orphan refresh children cleaned up.

## Evidence
- Top-level background process: `npm run refresh:all`
- Live process pattern during the stall:
  - `tsx scripts/refresh-all.ts`
  - repeated `tsx scripts/refresh-county-month-nws-flood-alerts.ts` children
- No stdout from the parent process for the entire observed run window.
- `public/cache/pipeline-health.json` mtime stayed at `2026-08-15 13:13:43 -0500` throughout the attempt.
- Core downstream/public artifacts also stayed stale:
  - `public/cache/city-open-data-ranked-tx.json` → `2026-08-23 04:23:32 -0500`
  - `public/cache/roadmap-open-data-botnet.json` → `2026-08-23 04:23:37 -0500`
  - `public/cache/analytics/*.json` → `2026-08-15 13:12:32 -0500`
- Some upstream inputs were fresh the same day, including:
  - `public/cache/city-open-data-tx.json` → `2026-08-30 13:02:49 -0500`
  - `data/county-month-drought-usdm.json` → `2026-08-30 13:02:45 -0500`
  - `data/county-month-nws-flood-alerts.json` → `2026-08-30 09:35:35 -0500`
  - `data/county-month-precipitation-open-meteo.json` → `2026-08-30 00:48:01 -0500`
  - `data/county-month-streamflow-usgs.json` → `2026-08-30 11:25:05 -0500`

## Interpretation
Treat this as an incomplete run, not as a fresh degraded snapshot. The key signal is that production-facing public artifacts were not regenerated even though several upstream inputs were already fresh or refreshed that day.

## Operator guidance
- Report `ALERT` if asked whether production artifacts are fresh enough for use.
- Distinguish the stale snapshot's recorded failures from the newly attempted run's incomplete/stuck state.
- Explicitly say the site health endpoint can remain `200 OK` while production-facing artifacts are stale.
- After capturing evidence, kill both the tracked `refresh-all` process and any orphan `refresh-county-month-nws-flood-alerts.ts` children.