# 2026-08-15 refresh-all silent success with non-CID failures

## Summary
A scheduled `npm run refresh:all` run in `/home/stathis/atlas-tx` stayed completely silent for the full observed runtime, but **completed successfully** after about **13m24s** and rewrote `public/cache/pipeline-health.json`.

Final snapshot state:
- process exit: `0`
- `overallStatus`: `degraded`
- `pipeline-health.json` mtime: `2026-08-15 13:13:43 -05:00`
- visible whole-run duration: about `804s` (derived from earliest `startedAt` to latest `endedAt`)

## Important interpretation
This run is a useful counterexample to the earlier 2026-08-15 silent-stall pattern.

During the first ~9 minutes, the process looked stall-like:
- no stdout at all
- repeated `refresh-county-month-nws-flood-alerts.ts` children early in the run
- older public artifacts had not yet moved

But later checks showed the step frontier **advanced** into later tasks:
- `refresh-county-month-streamflow`
- `refresh-county-month-precipitation`
- `refresh-roadmap-open-data`
- `refresh-cid`

And the downstream public artifacts were rewritten:
- `public/cache/pipeline-health.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/twdb-hydrology-tx.json`
- `public/cache/surface-water-quality-tx.json`

## Failure set in the completed snapshot
Non-CID failures:
- `refresh-city-open-data` — Dallas Open Data fetch `503`
- `refresh-city-open-data-curated` — Austin Open Data fetch `503`
- `refresh-city-open-data-ranked` — Austin Open Data fetch `503`
- `refresh-county-month-temperature` — Open-Meteo archive `429`

CID failure:
- `refresh-cid` — missing seed input (`organization`, `permit number`, or `person-name`)

## Freshness consequence
Even with a fresh `pipeline-health.json`, production should still be treated as **not fresh enough for use** if key fast-lane public artifacts remain stale. In this run:
- `public/cache/city-open-data-curated-tx.json` remained about 6 days old
- `public/cache/city-open-data-ranked-tx.json` remained about 6 days old

So the right operator conclusion is:
- completed run, not an incomplete/stuck run
- `overallStatus: degraded`
- **ALERT** because non-CID critical steps failed
- production-facing city-open-data derived artifacts are stale beyond the fast-lane window

## Additional schema note
In this snapshot, top-level `generatedAt` reflected the **start** of the run (`2026-08-15T18:00:19.044Z`), not completion time. For recency, prefer:
1. file mtime of `public/cache/pipeline-health.json`, and/or
2. latest `steps[].endedAt`

Do not interpret `generatedAt` alone as the completion timestamp.