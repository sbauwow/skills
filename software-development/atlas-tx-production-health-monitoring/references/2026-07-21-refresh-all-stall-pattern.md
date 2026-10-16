# 2026-07-21 refresh-all stall pattern

Observed during cron watchdog execution after launching `npm run refresh:all` in `/home/stathis/atlas-tx`.

## Symptoms
- Top-level `refresh-all.ts` stayed silent for ~6 minutes.
- Active child processes included:
  - `refresh-county-month-nws-flood-alerts.ts`
  - `refresh-county-month-temperature.ts`
- `public/cache/pipeline-health.json` was **not** rewritten.

## Mixed-state evidence
Artifacts that moved during the stuck run:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.partial.json`

Artifacts still stale after the attempted run:
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`

## Reporting guidance
- Treat this as an **incomplete/stuck run**, not a successful refresh.
- If the health snapshot still shows old failed non-CID weather steps, raise `ALERT`.
- State explicitly that some upstream files refreshed while core public production artifacts did not.

## Artifact-name correction
When checking production freshness, prefer real on-disk public artifacts over guessed aliases:
- ranked catalog: `public/cache/city-open-data-ranked-tx.json`
- roadmap: `public/cache/roadmap-open-data-botnet.json`
- analytics: inspect the `public/cache/analytics/` family, not a single `public/cache/analytics.json`
