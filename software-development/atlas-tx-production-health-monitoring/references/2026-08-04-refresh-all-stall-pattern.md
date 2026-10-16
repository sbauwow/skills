# 2026-08-04 refresh-all stall pattern

Observed during cron-driven production health monitoring in `/home/stathis/atlas-tx`.

## Symptom
- `npm run refresh:all` ran silently for about 6m38s and never emitted a new `public/cache/pipeline-health.json`.
- The top-level process tree showed `refresh-all.ts` still alive while child refreshers remained active.
- Early in the run, live children included both:
  - `refresh-county-month-nws-flood-alerts.ts`
  - `refresh-county-month-temperature.ts`
- Later, the run collapsed to repeated / remaining `refresh-county-month-nws-flood-alerts.ts` children while the parent still hung.

## Mixed artifact freshness captured during the stall
Recently refreshed upstream / intermediate artifacts:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.json`
- `data/county-month-nws-flood-alerts.json`
- `data/county-month-precipitation-open-meteo.json` was also recent during the check window

Still stale production-facing artifacts:
- `public/cache/pipeline-health.json` (~1695h old at inspection time)
- `public/cache/city-open-data-ranked-tx.json` (~1695h old)
- `public/cache/roadmap-open-data-botnet.json` (~1695h old)
- `public/cache/surface-water-quality-tx.json` (~336h old)

## Operational interpretation
Treat this as an incomplete refresh, not a successful run with a degraded snapshot. The key signal is that some upstream data files moved, but the committed health snapshot and multiple downstream public artifacts did not.

## Reporting guidance
- Say explicitly that `refresh:all` did not complete.
- If the stale snapshot still contains non-CID critical failures, begin the operator report with `ALERT`.
- Distinguish:
  1. the stale snapshot's recorded failures, from
  2. the newly attempted run hanging before a replacement snapshot was emitted.
- Conclude that artifacts are not fresh enough for production use when core public cache outputs remain stale even if a few upstream files refreshed.

## Cleanup guidance
After capturing evidence, terminate the timed-out `refresh-all.ts` parent and leftover `tsx scripts/refresh-*` children so later cron invocations do not overlap.
