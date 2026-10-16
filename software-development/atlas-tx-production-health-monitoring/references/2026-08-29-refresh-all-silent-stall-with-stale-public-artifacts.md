# 2026-08-29 refresh-all silent stall with stale public artifacts

## Summary
A cron-triggered `npm run refresh:all` attempt in `/home/stathis/atlas-tx` stayed completely silent, never regenerated `public/cache/pipeline-health.json`, and remained pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children. The run was treated as incomplete and explicitly cleaned up.

## Observed run window
- Start context: 2026-08-29 13:00 local time (`America/Chicago` offset `-05:00` from `date -Is` output).
- After ~68s: tracked `refresh-all` still running with no output; public snapshot mtimes unchanged.
- After ~204s: live children collapsed to repeated `refresh-county-month-nws-flood-alerts.ts` workers; `public/cache/city-open-data-tx.json` and `data/county-month-drought-usdm.json` had advanced, but `pipeline-health.json` and downstream public artifacts had not.
- After ~634s (~10m34s): same repeated NWS-child pattern persisted, still no `pipeline-health.json` rewrite, so the run was classified as a silent stall rather than a long-silent successful refresh.
- Cleanup: killed the tracked background `refresh-all` process, then verified no remaining `refresh-all` / `tsx scripts/refresh-*` processes were left.

## Process signature
Observed live process tree during the stall:
- `tsx scripts/refresh-all.ts`
- repeated `tsx scripts/refresh-county-month-nws-flood-alerts.ts` children
- earlier in the run, drought, city-open-data, and temperature children appeared briefly before the frontier collapsed back to NWS flood alerts only

Interpretation: this matched the known repeated-NWS-child stall pattern rather than the 2026-08-09 / second-2026-08-15 long-silent-but-eventually-completing pattern, because later-step public artifacts never began to advance and the health snapshot never regenerated.

## Artifacts that moved during the incomplete attempt
These upstream inputs/caches advanced during the attempted run:
- `public/cache/city-open-data-tx.json` → 2026-08-29 13:02:23 -0500
- `data/county-month-drought-usdm.json` → 2026-08-29 13:03:30 -0500

These were already fresh the same day or recently, but did not prove pipeline completion:
- `data/county-month-precipitation-open-meteo.json` → 2026-08-29 06:48:10 -0500
- `data/county-month-streamflow-usgs.json` → 2026-08-29 12:24:14 -0500
- `data/county-month-temperature-open-meteo.partial.json` → 2026-08-29 04:08:32 -0500

Notably stale during and after the attempt:
- `public/cache/pipeline-health.json` → still 2026-08-15 13:13:43 -0500
- `public/cache/city-open-data-ranked-tx.json` → still 2026-08-23 04:23:32 -0500
- `public/cache/roadmap-open-data-botnet.json` → still 2026-08-23 04:23:37 -0500
- `public/cache/surface-water-quality-tx.json` → still 2026-08-15 13:12:31 -0500
- `public/cache/twdb-hydrology-tx.json` → still 2026-08-23 04:16:30 -0500
- `public/cache/analytics/county-history.json` and sibling analytics artifacts → still 2026-08-15 13:12:32 -0500

## Authoritative stale snapshot at report time
Because the new run never emitted a fresh health snapshot, the old `public/cache/pipeline-health.json` remained authoritative:
- `generatedAt`: `2026-08-15T18:00:19.044Z`
- `overallStatus`: `degraded`
- expected step coverage: complete (`13/13`)
- derived run duration from step timestamps: ~804s (`13m24s`)

Recorded failed steps in that stale snapshot:
- Non-CID critical failures:
  - `refresh-city-open-data`
  - `refresh-city-open-data-curated`
  - `refresh-city-open-data-ranked`
  - `refresh-county-month-temperature`
- CID failure:
  - `refresh-cid`

## Operational conclusion
Lead the operator report with `ALERT` when this pattern appears:
- the attempted refresh is incomplete,
- the authoritative snapshot still contains non-CID critical failures, and
- fast/intelligence-lane public artifacts are well past freshness thresholds, so the production-facing artifacts are not fresh enough for use.
