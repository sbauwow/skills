# 2026-08-20 refresh-all silent stall with stale public artifacts

## Summary
A scheduled watchdog run launched `npm run refresh:all` in `/home/stathis/atlas-tx`. The process produced no stdout for the entire observed run, remained active for about 10m13s, and never regenerated `public/cache/pipeline-health.json`.

## Stall signature
- Parent process stayed silent throughout the run.
- Child-process pattern stalled on repeated `refresh-county-month-nws-flood-alerts.ts` workers.
- Around ~3m20s there were two `refresh-county-month-nws-flood-alerts.ts` children.
- Around ~10m21s only a remaining `refresh-county-month-nws-flood-alerts.ts` child was still alive; the frontier had not advanced into later steps like roadmap or CID.
- To avoid overlap with future cron runs, the watchdog killed the tracked background process and then cleaned up orphan `tsx scripts/refresh-*` children.

## Mixed freshness observed during the failed attempt
Artifacts that refreshed during the stalled attempt:
- `public/cache/city-open-data-tx.json` → 2026-08-20T18:02:38Z
- `data/county-month-drought-usdm.json` → 2026-08-20T18:03:35Z
- `data/county-month-temperature-open-meteo.partial.json` → 2026-08-20T18:02:23Z
- `data/county-month-nws-flood-alerts.json` → 2026-08-20T17:20:03Z
- `data/county-month-precipitation-open-meteo.json` → 2026-08-20T17:38:30Z

Core public artifacts that did **not** refresh:
- `public/cache/pipeline-health.json` stayed at 2026-08-15T18:13:43Z
- `public/cache/city-open-data-ranked-tx.json` stayed at 2026-08-16T09:32:26Z
- `public/cache/roadmap-open-data-botnet.json` stayed at 2026-08-16T09:32:27Z
- `public/cache/analytics/county-history.json` stayed at 2026-08-15T18:12:32Z
- `public/cache/analytics/source-freshness.json` stayed at 2026-08-15T18:12:32Z
- `public/cache/twdb-hydrology-tx.json` stayed at 2026-08-16T09:26:22Z
- `public/cache/surface-water-quality-tx.json` stayed at 2026-08-15T18:12:31Z

## Last completed snapshot still in force
Because the run never rewrote `pipeline-health.json`, the effective production snapshot remained the 2026-08-15 completed run:
- `overallStatus: degraded`
- Visible duration from earliest `startedAt` to latest `endedAt`: 804s (~13m24s)
- Non-CID failed steps still recorded in that snapshot:
  - `refresh-city-open-data`
  - `refresh-city-open-data-curated`
  - `refresh-city-open-data-ranked`
  - `refresh-county-month-temperature`
- CID also failed (`refresh-cid`) due to missing seed input, but that should not overshadow the non-CID critical failures when deciding whether to lead with `ALERT`.

## Reporting lesson
Treat this as an incomplete/stalled refresh, not as a fresh degraded run. In the operator report:
- lead with `ALERT`
- say the latest available `pipeline-health.json` is stale
- separate upstream input refreshes from production-facing public artifact freshness
- explicitly say core public artifacts are not fresh enough for production use
- call out non-CID critical failures from the last completed snapshot, since no newer completed snapshot replaced them
