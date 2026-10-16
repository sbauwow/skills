# 2026-08-16 refresh-all silent stall with stale fast-lane public artifacts

## Signature
- `npm run refresh:all` ran completely silent for about 11m26s.
- Live process tree stayed pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children instead of advancing into later steps.
- The run never rewrote `public/cache/pipeline-health.json`.

## Mixed refresh evidence
Artifacts that advanced during the attempted run:
- `public/cache/city-open-data-tx.json` → 2026-08-16T04:02:31
- `data/county-month-drought-usdm.json` → 2026-08-16T04:02:38
- `data/county-month-temperature-open-meteo.json` → 2026-08-16T04:03:30

Artifacts that did not advance and remained production-stale:
- `public/cache/pipeline-health.json` stayed on the 2026-08-15 degraded snapshot
- `public/cache/city-open-data-ranked-tx.json` stayed on 2026-08-09
- `public/cache/roadmap-open-data-botnet.json` stayed on 2026-08-15

## Snapshot interpretation
The stale `pipeline-health.json` still showed:
- `overallStatus: degraded`
- non-CID failed steps: `refresh-city-open-data`, `refresh-city-open-data-curated`, `refresh-city-open-data-ranked`, `refresh-county-month-temperature`
- CID also failed, but the non-CID failures are enough to raise an operator alert
- visible completed-run duration in the stale snapshot was ~804s (~13m24s), which belongs to the prior completed run rather than the stalled 2026-08-16 attempt

## Reporting rule
Treat this as an incomplete run, not as a fresh degraded run. In the report:
- lead with `ALERT`
- say the attempted refresh never regenerated the health snapshot
- separate upstream/input refreshes from production-facing public artifact freshness
- conclude that core public artifacts are not fresh enough for production use because fast-lane files exceeded the 12h threshold

## Cleanup rule
After capturing evidence, terminate the tracked `refresh-all` process and any orphan `tsx scripts/refresh-*` children so the next cron invocation does not overlap with the stuck run.
