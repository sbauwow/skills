# 2026-08-22 refresh-all silent stall with stale public artifacts

## Summary
A scheduled watchdog run on 2026-08-22 started `npm run refresh:all`, but the orchestrator never emitted a new `public/cache/pipeline-health.json`. The run stayed silent for ~11m26s, remained pinned on `refresh-county-month-nws-flood-alerts`, and had to be terminated to avoid overlap with later cron invocations.

## Evidence
- Foreground attempt timed out at 600s with only the startup banner and a lone `DEP0205` warning.
- Background retry showed no stdout for the entire observed run.
- Process-tree evolution:
  - early: `refresh-city-open-data`, `refresh-county-month-drought`, and `refresh-county-month-nws-flood-alerts` children active
  - later: only `refresh-county-month-nws-flood-alerts.ts` remained alive under `refresh-all.ts`
- `public/cache/pipeline-health.json` mtime stayed at `2026-08-15 18:13:43 UTC`; no new health snapshot was emitted.
- `https://atlastexas.org/api/health` still returned HTTP 200 with `{"ok":true,...}`.

## Artifact movement during the failed attempt
Refreshed/upstream moved:
- `public/cache/city-open-data-tx.json` -> `2026-08-22 13:13:04`
- `data/county-month-drought-usdm.json` -> `2026-08-22 13:15:30`
- `data/county-month-streamflow-usgs.json` was already fresh at `2026-08-22 12:23:30`
- `data/county-month-precipitation-open-meteo.json` was already fresh at `2026-08-22 00:38:18`

Did not refresh / remained stale:
- `public/cache/pipeline-health.json` -> `2026-08-15 18:13:43`
- `public/cache/city-open-data-ranked-tx.json` -> `2026-08-16 09:32:26`
- `public/cache/roadmap-open-data-botnet.json` -> `2026-08-16 09:32:27`
- `public/cache/twdb-hydrology-tx.json` -> `2026-08-16 09:26:22`
- `public/cache/surface-water-quality-tx.json` -> `2026-08-15 18:12:31`
- `public/cache/analytics/county-history.json` -> `2026-08-15 18:12:32`
- `data/county-month-nws-flood-alerts.json` -> `2026-08-20 19:11:27Z` / mtime `2026-08-20 14:21:02`
- `data/county-month-temperature-open-meteo.json` -> stale from `2026-08-16`

## Interpretation
Treat this as an incomplete run, not as a fresh degraded pipeline snapshot. The distinguishing signal is that the step frontier never advanced beyond the repeated NWS flood-alert child and the health snapshot was not regenerated. Even with fresh upstream weather inputs and a healthy site endpoint, the production-facing public artifacts are not fresh enough for production use.

## Operator action
- Report `ALERT`.
- Explicitly separate site reachability from public-artifact freshness.
- Mention that the last authoritative `pipeline-health.json` remains the 2026-08-15 degraded snapshot.
- Kill the tracked background process and any remaining `tsx scripts/refresh-all.ts` / `tsx scripts/refresh-county-month-nws-flood-alerts.ts` children after collecting evidence.
