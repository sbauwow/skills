# 2026-07-20 refresh-all stall pattern

Observed watchdog pattern while running `npm run refresh:all` in `/home/stathis/atlas-tx`.

## What happened
- Top-level `refresh-all.ts` produced no log output.
- After about 4m49s, the process tree had collapsed to one remaining child refresh:
  - `npm run refresh:county-month-nws-flood-alerts`
  - `tsx scripts/refresh-county-month-nws-flood-alerts.ts`
- `public/cache/pipeline-health.json` was not rewritten and remained at `2026-05-25T18:10:07Z`.

## Mixed refresh evidence captured during the hung run
Refreshed during the attempted run:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.json`
- `data/county-month-precipitation-open-meteo.json`

Still stale during the same check:
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/analytics/*`
- `public/cache/twdb-hydrology-tx.json`
- `public/cache/surface-water-quality-tx.json`

## Interpretation
This is an incomplete refresh, not a successful partial production update:
- some upstream inputs and one public catalog artifact moved,
- but the authoritative production health snapshot and multiple core public artifacts did not.

For production-use judgments, answer **not fresh enough** when the public snapshot and downstream public artifacts remain weeks old even if some `data/` files refreshed today.

## Cleanup step
After capturing evidence, kill the parent `refresh:all` process and any orphan `refresh-county-month-nws-flood-alerts` child so the cron watchdog does not leave overlapping refresh jobs behind.
