# 2026-08-09 refresh-all long silent completion

Use this note when judging whether a silent `npm run refresh:all` is hung versus merely slow.

## Observed run shape
- Command: `npm run refresh:all`
- Repo: `/home/stathis/atlas-tx`
- Parent process produced **no stdout at all** during the run.
- Runtime from step timestamps in the emitted `public/cache/pipeline-health.json`: **892.051s** (~14m52s).
- The watchdog observed the process still running at ~12m+ and at that point it looked similar to prior stall cases.
- Unlike the true stall cases, this run eventually exited cleanly and rewrote the production health snapshot plus downstream public artifacts.

## Mid-run evidence that it was still advancing
- Early process tree included `refresh-county-month-nws-flood-alerts.ts` children.
- Around ~13:10 local time, `data/county-month-nws-flood-alerts.json` refreshed.
- Later, the remaining active child shifted to `refresh-county-month-precipitation.ts`.
- Shortly afterward, downstream public artifacts refreshed:
  - `public/cache/city-open-data-ranked-tx.json`
  - `public/cache/roadmap-open-data-botnet.json`
  - `public/cache/surface-water-quality-tx.json`
  - `public/cache/twdb-hydrology-tx.json`
  - `public/cache/pipeline-health.json`

## Final snapshot state
- `overallStatus`: `degraded`
- Non-ok steps:
  - `refresh-county-month-precipitation` — failed with Open-Meteo archive `429`
  - `refresh-cid` — failed because Search Two refresh needs a seed (`organization`, `permit number`, or `person-name`)
- This is a **completed degraded run**, not an incomplete run.

## Operator takeaway
Do **not** declare a hang solely because `refresh:all` is silent for >10–12 minutes. Before killing anything:
1. Check whether child processes are moving forward to later steps.
2. Check whether upstream data files are still updating.
3. Most importantly, check whether `public/cache/pipeline-health.json` gets regenerated.

A true incomplete-run alert still requires the old signature: no new health snapshot plus only partial/upstream refreshes.