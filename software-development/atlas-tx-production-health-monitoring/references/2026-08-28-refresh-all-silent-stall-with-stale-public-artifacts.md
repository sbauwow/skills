# 2026-08-28 refresh-all silent stall with stale public artifacts

## What happened
A scheduled operator run started `npm run refresh:all` in `/home/stathis/atlas-tx` and the process stayed completely silent for the entire observed run.

## Observed process pattern
- Parent remained `tsx scripts/refresh-all.ts`
- Early children included `refresh-county-month-drought.ts` and `refresh-county-month-nws-flood-alerts.ts`
- After several minutes the frontier collapsed to a remaining/repeated `refresh-county-month-nws-flood-alerts.ts` child
- At ~10m37s elapsed, the run was still pinned on `refresh-county-month-nws-flood-alerts.ts` with no stdout and no advance into later steps

## Artifact movement during the stuck attempt
Some upstream inputs refreshed during the attempt:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.json`
- `data/county-month-precipitation-open-meteo.json`
- `data/county-month-streamflow-usgs.json`

But core downstream/public artifacts did not move:
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/twdb-hydrology-tx.json`
- `public/cache/surface-water-quality-tx.json`
- `public/cache/analytics/county-history.json`
- other `public/cache/analytics/*` files
- `data/county-month-nws-flood-alerts.json` also did not refresh

## Health snapshot state that remained authoritative
Because `pipeline-health.json` was not rewritten, the authoritative snapshot remained the old completed run:
- snapshot generatedAt: `2026-08-15T18:00:19.044Z`
- snapshot file mtime: `2026-08-15T18:13:43Z`
- `overallStatus`: `degraded`
- visible whole-run duration from step times: ~804.352s (~13m24s)

Recorded non-CID failures in that stale snapshot:
- `refresh-city-open-data` (`503` Dallas open data API)
- `refresh-city-open-data-curated` (`503` Austin open data API)
- `refresh-city-open-data-ranked` (downstream of Austin `503`)
- `refresh-county-month-temperature` (`429` Open-Meteo)
- `refresh-cid` failed too, but it was the known missing Search Two seed case rather than a non-CID critical step

## Interpretation
Treat this as an **incomplete stalled run**, not as a fresh degraded refresh.

Key signals:
1. The run stayed silent for >10 minutes.
2. The live child set collapsed to `refresh-county-month-nws-flood-alerts.ts` instead of advancing into later stages.
3. `pipeline-health.json` was not regenerated.
4. Core public artifacts remained stale even though some upstream inputs moved.

## Operator wording to preserve
- Lead with `ALERT` because the stale authoritative snapshot still carries non-CID critical failures and fast/intelligence public artifacts are beyond freshness thresholds.
- Separate **latest visible snapshot status** (`degraded`) from **this attempted run result** (`incomplete/stuck`).
- When asked whether artifacts are fresh enough for production use, answer **no**.

## Cleanup
After capturing evidence:
- kill the tracked `refresh-all` process
- then check separately for and terminate any orphan `tsx scripts/refresh-county-month-nws-flood-alerts.ts` child that survives the parent kill

In this run the tracked process exited with code `143` after termination, and one orphan `refresh-county-month-nws-flood-alerts.ts` child remained and had to be killed separately.