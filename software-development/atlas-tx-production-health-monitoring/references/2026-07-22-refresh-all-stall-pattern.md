# 2026-07-22 refresh:all stall pattern

## Trigger
Cron-style production health check that explicitly ran `npm run refresh:all` before evaluating `public/cache/pipeline-health.json`.

## What happened
- `npm run refresh:all` ran silently for about 6m15s and never completed.
- The process tree collapsed to the top-level `refresh-all.ts` plus lingering `refresh-county-month-nws-flood-alerts.ts` children.
- `public/cache/pipeline-health.json` was not rewritten during the attempted run.
- The previous health snapshot remained:
  - `generatedAt`: `2026-05-25T18:00:15.430Z`
  - `overallStatus`: `degraded`
  - visible total run duration in snapshot: 591s

## Partial refresh evidence
These moved during the stuck run:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.partial.json`

These did not move and remained stale:
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`

## Operational interpretation
Treat this as an incomplete/hung refresh, not a successful partial completion.
Do not infer success from the fresh `city-open-data-tx.json` or upstream `data/` movement when the production health snapshot and ranked/roadmap public artifacts remain stale.

## Reporting implications
- Lead with `ALERT` when the caller says to alert on any non-CID critical failure, because the stale health snapshot still records non-CID failures (`refresh-county-month-precipitation`, `refresh-county-month-temperature`).
- Answer the production-use question separately from site reachability. `https://atlastexas.org/api/health` can still be HTTP 200 / `ok: true` while core public artifacts are not fresh enough for production use.
- If the refresh attempt is hung, terminate the leftover `refresh-all.ts` / `refresh-county-month-nws-flood-alerts.ts` children after capturing evidence so the next cron invocation does not overlap.
