# 2026-08-11 refresh-all silent stall with stale public artifacts

## Summary
A scheduled watchdog run started `npm run refresh:all` and observed a long silent execution that never regenerated `public/cache/pipeline-health.json`. The run was treated as incomplete and the leftover refresh processes were terminated.

## Observable pattern
- `npm run refresh:all` produced no stdout for the entire observed run.
- At ~3–6 minutes, child processes included both:
  - `refresh-county-month-temperature.ts`
  - `refresh-county-month-nws-flood-alerts.ts`
- By ~9–13 minutes, the run had collapsed to repeated/remaining `refresh-county-month-nws-flood-alerts.ts` children while `refresh-county-month-temperature.ts` had exited.
- `public/cache/pipeline-health.json` did not move off the prior completed snapshot from 2026-08-09.

## Files that refreshed during the incomplete run
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.partial.json`

## Files that did not refresh
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/surface-water-quality-tx.json`
- `public/cache/twdb-hydrology-tx.json`

## Snapshot interpretation
The stale snapshot remained the 2026-08-09 completed run:
- `overallStatus: degraded`
- visible failed steps:
  - `refresh-county-month-precipitation` (Open-Meteo 429)
  - `refresh-cid` (missing seed input)
- visible completed whole-run duration:
  - about 892 seconds (~14m52s)

## Operational takeaway
Treat this run as an incomplete refresh, not a fresh degraded snapshot. Report the mixed state explicitly: some upstream inputs moved, but production-facing public artifacts did not. For production-use judgments, the stale fast-lane artifacts make the result not fresh enough even though some upstream `data/` files advanced.
