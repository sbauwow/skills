# 2026-08-15 refresh:all silent stall with stale health snapshot

## Summary
A scheduled watchdog run started `npm run refresh:all`, observed no stdout for the entire attempt, and found the process still running after ~11 minutes. The process tree never advanced beyond repeated `refresh-county-month-nws-flood-alerts.ts` children. `public/cache/pipeline-health.json` was not regenerated, so the repo still reflected the older 2026-08-12 degraded snapshot.

## Evidence pattern
- Parent process: `npm run refresh:all` / `tsx scripts/refresh-all.ts`
- Silent run: no stdout beyond startup banner/deprecation warning
- Child-process pattern persisted as repeated `refresh-county-month-nws-flood-alerts.ts` processes
- Upstream artifacts that refreshed during the attempt:
  - `public/cache/city-open-data-tx.json`
  - `data/county-month-drought-usdm.json`
- Upstream artifacts that did **not** advance during the attempt:
  - `data/county-month-temperature-open-meteo.json`
  - `data/county-month-temperature-open-meteo.partial.json`
  - `data/county-month-nws-flood-alerts.json`
  - `data/county-month-precipitation-open-meteo.json`
- Core public artifacts that remained stale:
  - `public/cache/pipeline-health.json`
  - `public/cache/city-open-data-ranked-tx.json`
  - `public/cache/roadmap-open-data-botnet.json`
  - `public/cache/twdb-hydrology-tx.json`
  - `public/cache/surface-water-quality-tx.json`

## Interpretation
Treat this as an incomplete run, not as a new degraded snapshot. Production freshness should be judged from the still-stale public artifacts, not from the few upstream files that moved during the failed attempt.

## Cleanup
After capturing evidence, terminate the tracked `refresh-all` process and any orphan `tsx scripts/refresh-county-month-nws-flood-alerts.ts` children so the next cron invocation does not overlap.