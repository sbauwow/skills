# 2026-08-23 refresh-all silent stall with stale public artifacts

## Summary
A scheduled watchdog run started `npm run refresh:all` in `/home/stathis/atlas-tx`. The process stayed completely silent for about 10m37s and never rewrote `public/cache/pipeline-health.json`. The live child-process pattern stayed pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children rather than advancing into later steps. The run was killed and orphan refresh children were cleaned up.

## Evidence
- Parent/background job: `npm run refresh:all`
- Runtime before kill: about 629s uptime on the tracked process
- Repeated live children during inspection:
  - `tsx scripts/refresh-all.ts`
  - repeated `tsx scripts/refresh-county-month-nws-flood-alerts.ts`
- No stdout was emitted by the tracked process during the observed run.

## Artifact state observed
### Did refresh recently during/around the attempt
- `public/cache/city-open-data-tx.json` — refreshed on 2026-08-23 09:02 UTC
- `data/county-month-drought-usdm.json` — refreshed on 2026-08-23 09:02 UTC
- `data/county-month-streamflow-usgs.json` — refreshed on 2026-08-23 08:23 UTC
- `data/county-month-precipitation-open-meteo.json` — fresh the same day (2026-08-23 03:37 UTC)

### Did not refresh
- `public/cache/pipeline-health.json` — still 2026-08-15 18:13 UTC
- `public/cache/city-open-data-ranked-tx.json` — still 2026-08-16 09:32 UTC
- `public/cache/roadmap-open-data-botnet.json` — still 2026-08-16 09:32 UTC
- `public/cache/twdb-hydrology-tx.json` — still 2026-08-16 09:26 UTC
- `public/cache/surface-water-quality-tx.json` — still 2026-08-15 18:12 UTC
- `data/county-month-nws-flood-alerts.json` — did not refresh during the attempt
- `data/county-month-temperature-open-meteo.json` — stale; `.partial.json` also not newly advanced during this observed attempt

## Interpretation
Treat this as an incomplete/stalled refresh, not as a fresh degraded snapshot. The key signal is that some upstream inputs moved while the production health snapshot and core downstream public artifacts did not. Production freshness must therefore be judged from the stale public artifacts, not from the partially refreshed inputs.

## Reporting guidance
- Lead with `ALERT`.
- Say the attempted refresh stalled silently and the production health snapshot was not regenerated.
- Separate upstream input freshness from production-facing public artifact freshness.
- Say core public artifacts were **not** refreshed recently enough for production use.
- If cleanup was performed, note that the stuck parent/orphan refresh children were killed after evidence capture.
