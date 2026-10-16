# 2026-07-17 refresh-all stall pattern

Observed during a cron watchdog run in `/home/stathis/atlas-tx`.

## What happened
- `npm run refresh:all` was started and left running in the background.
- After about 7 minutes, the process still produced no useful final output and had not rewritten `public/cache/pipeline-health.json`.
- Process tree at the time of the stall showed:
  - top-level `tsx scripts/refresh-all.ts`
  - child `tsx scripts/refresh-county-month-drought.ts`
  - child `tsx scripts/refresh-county-month-nws-flood-alerts.ts`
  - child `tsx scripts/refresh-county-month-temperature.ts`

## On-disk evidence
Artifacts that moved during the stuck run:
- `public/cache/city-open-data-tx.json`
- `data/county-month-temperature-open-meteo.partial.json`

Artifacts that did not move and therefore remained stale:
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/analytics/county-history.json`
- `public/cache/twdb-hydrology-tx.json`
- `public/cache/surface-water-quality-tx.json`

## Operational interpretation
- Do **not** infer success from the refreshed city catalog file alone.
- Do **not** treat a moving `*.partial.json` weather artifact as evidence that production-facing refresh completed.
- If the site health endpoint is 200 but these public artifacts remain weeks old, the correct operator conclusion is still ALERT and "not fresh enough for production use."

## Cleanup action
After capturing evidence, terminate the top-level `refresh-all` process and leftover `tsx scripts/refresh-*` children so the next cron run does not overlap with orphaned work.
