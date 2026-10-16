# 2026-08-24 refresh-all silent stall with stale public artifacts

## Summary
A scheduled `npm run refresh:all` attempt on 2026-08-24 looked like the established silent-stall pattern, not a long-silent success. The run stayed completely silent for about 10m35s, never regenerated `public/cache/pipeline-health.json`, and remained pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children until it was killed.

## Evidence
- Background process started at roughly 2026-08-24 13:00 local time.
- At ~66s: active children included `refresh-city-open-data.ts`, `refresh-county-month-drought.ts`, `refresh-county-month-temperature.ts`, and repeated `refresh-county-month-nws-flood-alerts.ts`.
- At ~206s: the frontier had already collapsed to `refresh-county-month-temperature.ts` plus repeated `refresh-county-month-nws-flood-alerts.ts`.
- At ~300s: only repeated `refresh-county-month-nws-flood-alerts.ts` children remained.
- At ~501s and ~635s: still only repeated `refresh-county-month-nws-flood-alerts.ts` children; no stdout; no `pipeline-health.json` rewrite.
- After killing the tracked background process, one orphan `refresh-county-month-nws-flood-alerts.ts` child still remained and had to be killed separately.

## Artifact movement during the stalled attempt
These upstream artifacts advanced during the failed run:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.partial.json`

These production-facing artifacts did **not** advance:
- `public/cache/pipeline-health.json` (still mtime 2026-08-15T18:13:43Z)
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/analytics/county-history.json`
- `public/cache/analytics/county-movers.json`
- `public/cache/analytics/pressure-risk-scatter.json`
- `public/cache/analytics/source-freshness.json`
- `public/cache/twdb-hydrology-tx.json`
- `public/cache/surface-water-quality-tx.json`

## Snapshot interpretation
The authoritative `public/cache/pipeline-health.json` remained the old 2026-08-15 completed snapshot:
- `overallStatus: degraded`
- visible whole-run duration: ~804.352s
- failed steps in that snapshot included non-CID critical failures:
  - `refresh-city-open-data`
  - `refresh-city-open-data-curated`
  - `refresh-city-open-data-ranked`
  - `refresh-county-month-temperature`
  - `refresh-cid`

## Operational rule
When this exact pattern appears:
1. Do **not** treat the attempted run as a fresh degraded refresh.
2. Report `ALERT` if the stale snapshot still records non-CID critical failures or if fast/intelligence public artifacts are beyond freshness thresholds.
3. Separate partially refreshed upstream `data/` or raw catalog artifacts from stale production-facing public artifacts.
4. Say explicitly that the core public artifacts are **not fresh enough for production use** when `pipeline-health.json`, ranked catalog, roadmap, or analytics outputs remain stale.
5. Kill both the tracked `refresh-all` process and any orphan `refresh-county-month-nws-flood-alerts.ts` child left behind.
