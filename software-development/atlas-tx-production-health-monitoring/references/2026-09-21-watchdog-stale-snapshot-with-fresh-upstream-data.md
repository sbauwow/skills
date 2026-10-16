# 2026-09-21 watchdog: stale health snapshot with fresh upstream data

## Finding
- Site: `https://atlastexas.org/api/health` → HTTP 200, `{ok:true, dataset_count:21}`. Reachable.
- `public/cache/pipeline-health.json`: `generatedAt` 2026-09-01T10:49:48.770Z, mtime 2026-09-01 06:13:05 CDT, `overallStatus: degraded`. Snapshot complete: 13/13 catalog stepIds present (keyed by `stepId`, joined to `task("<id>", "<family>")` calls in `src/lib/atlas-ingest-orchestrator.ts`).
- Snapshot age at check time (2026-09-21T03:07Z): 471h ≈ 19.6 days since latest step `endedAt` (2026-09-01T11:13:05Z).

## Mixed state (crashed run, not a stall)
- Fresh within ~0–1.5h of the check: `data/county-month-precipitation-open-meteo.json` (1h), `data/county-month-streamflow-usgs.json` (0h), `data/county-month-nws-flood-alerts.json` (0h).
- Stale: `pipeline-health.json` (19.6d), `city-open-data-ranked-tx.json` (28d), `city-open-data-curated-tx.json` (28d), `twdb-hydrology-tx.json` (28d), `surface-water-quality-tx.json` (36d), `data/county-month-temperature-open-meteo.json` (23d) + a `.partial.json` from 2026-09-01.
- No `tsx scripts/refresh-*` or `refresh-all` processes alive → the run that refreshed the upstream weather files aborted/finished without emitting a new health snapshot. Nothing to clean up.

## Policy verdict (fast 12h / intelligence 24h / context 21d)
- Fast lane: ALERT — 19.6d vs 12h.
- Intelligence lane: ALERT — 19.6d vs 24h (`refresh-cid` failed: missing seed).
- Context lane: ALERT on real artifacts — `twdb-hydrology-tx.json` 28d, `surface-water-quality-tx.json` 36d vs 21d (both steps failed 2026-09-01 with DNS `EAI_AGAIN`). The snapshot's own `endedAt` (19.6d) was still under 21d — on-disk mtimes of failed steps are the true freshness signal.
- `degraded` persisted ≥19.6 days → "degraded for too long" → report prefixed with ALERT.

## Failure causes in the stale snapshot
Non-CID critical failures: `refresh-twdb-hydrology` + `refresh-surface-water-quality` (DNS EAI_AGAIN to twdb.texas.gov / tceq.texas.gov), `refresh-city-open-data-curated` + `refresh-city-open-data-ranked` (fetch failed), `refresh-county-month-precipitation`/`-temperature` (Open-Meteo EAI_AGAIN), `refresh-county-month-streamflow` (USGS EAI_AGAIN), `refresh-county-month-nws-flood-alerts` (fema.gov EAI_AGAIN), `refresh-cid` (missing seed input — known intentional degraded-downgrade).
