# 2026-08-25 refresh-all silent stall with stale public artifacts

## Summary
A cron-driven `npm run refresh:all` run in `/home/stathis/atlas-tx` stayed completely silent for about 10m57s and never regenerated `public/cache/pipeline-health.json`. The live child-process pattern stayed pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children instead of advancing into later steps. Treat this as an incomplete run, not as a fresh degraded snapshot.

## Evidence
- Start context already showed an orphan `refresh-county-month-nws-flood-alerts.ts` child from an earlier run; it was killed before starting the new refresh.
- New refresh started via background `npm run refresh:all`.
- At ~75s: live children included `refresh-all.ts`, two `refresh-county-month-nws-flood-alerts.ts` children, `refresh-county-month-drought.ts`, and `refresh-county-month-temperature.ts`.
- At ~150s: only repeated `refresh-county-month-nws-flood-alerts.ts` children remained.
- At ~224s, ~299s, ~500s, and ~653s: the parent was still silent and still pinned on remaining `refresh-county-month-nws-flood-alerts.ts` children; no frontier advance into later steps was visible.
- After ~657s uptime, the tracked background process was still running with empty output; it was killed, then orphan NWS children were cleaned up.

## Artifact movement during the attempted run
Refreshed during the attempt:
- `public/cache/city-open-data-tx.json` → `2026-08-25T04:02:42-0500`
- `data/county-month-drought-usdm.json` → `2026-08-25T04:03:50-0500`

Did not refresh during the attempt:
- `public/cache/pipeline-health.json` remained `2026-08-15T13:13:43-0500`
- `public/cache/city-open-data-ranked-tx.json` remained `2026-08-23T04:23:32-0500`
- `public/cache/roadmap-open-data-botnet.json` remained `2026-08-23T04:23:37-0500`
- `public/cache/twdb-hydrology-tx.json` remained `2026-08-23T04:16:30-0500`
- `public/cache/surface-water-quality-tx.json` remained `2026-08-15T13:12:31-0500`
- `public/cache/analytics/county-history.json` remained `2026-08-15T13:12:32-0500`
- `public/cache/analytics/county-movers.json` remained `2026-08-15T13:12:32-0500`
- `public/cache/analytics/pressure-risk-scatter.json` remained `2026-08-15T13:12:32-0500`
- `public/cache/analytics/source-freshness.json` remained `2026-08-15T13:12:32-0500`
- `data/county-month-nws-flood-alerts.json` remained `2026-08-24T18:36:41-0500`
- `data/county-month-temperature-open-meteo.json` remained `2026-08-16T04:03:30-0500`
- `data/county-month-temperature-open-meteo.partial.json` remained `2026-08-24T13:03:46-0500`

## Health snapshot interpretation
The stale `pipeline-health.json` still reflected the 2026-08-15 completed degraded run:
- `overallStatus: degraded`
- failed steps: `refresh-city-open-data`, `refresh-city-open-data-curated`, `refresh-city-open-data-ranked`, `refresh-county-month-temperature`, `refresh-cid`
- visible run duration from step timestamps: about `804.352s` (~13m24s)
- step coverage remained complete (13/13 expected tasks)

## Operator takeaway
When the run stays silent for ~11 minutes, the child frontier collapses to repeated `refresh-county-month-nws-flood-alerts.ts` children, and `pipeline-health.json` plus downstream public artifacts do not move, report `ALERT`. Separate partially refreshed upstream inputs from production-facing artifacts and say the core public artifacts are not fresh enough for production use.