# 2026-08-26 refresh-all silent stall with stale public artifacts

## Summary
A scheduled watchdog run launched `npm run refresh:all` in `/home/stathis/atlas-tx`. The process stayed completely silent for about 13 minutes, never rewrote `public/cache/pipeline-health.json`, and remained pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children rather than advancing to later steps.

## Evidence
- Background process `npm run refresh:all` remained `running` with no stdout at ~192s, ~387s, ~583s, and ~780s uptime.
- Live process tree snapshots showed:
  - parent `tsx scripts/refresh-all.ts`
  - repeated child `tsx scripts/refresh-county-month-nws-flood-alerts.ts`
  - earlier transient `tsx scripts/refresh-county-month-temperature.ts`
- During the attempted run, only some upstream inputs moved:
  - `public/cache/city-open-data-tx.json` refreshed to the current run time
  - `data/county-month-drought-usdm.json` refreshed
  - `data/county-month-temperature-open-meteo.partial.json` kept advancing
- Core downstream/public artifacts did **not** move:
  - `public/cache/pipeline-health.json` stayed on the 2026-08-15 snapshot
  - `public/cache/city-open-data-ranked-tx.json` stayed at 2026-08-23
  - `public/cache/roadmap-open-data-botnet.json` stayed at 2026-08-23
  - `public/cache/analytics/*.json` stayed at 2026-08-15
  - `public/cache/surface-water-quality-tx.json` stayed at 2026-08-15
  - `public/cache/twdb-hydrology-tx.json` stayed at 2026-08-23
- The stale `pipeline-health.json` remained complete (13/13 steps) and still reported non-CID critical failures from the old completed snapshot:
  - `refresh-city-open-data`
  - `refresh-city-open-data-curated`
  - `refresh-city-open-data-ranked`
  - `refresh-county-month-temperature`
  - `refresh-cid`

## Interpretation
Treat this as an incomplete/stalled refresh, not as a new degraded refresh. The decisive signal was not mere silence duration, but the combination of:
1. no `pipeline-health.json` rewrite,
2. no frontier advance into later steps,
3. repeated NWS flood-alert child processes, and
4. only partial upstream refreshes while production-facing public artifacts stayed stale.

## Operator guidance
- Lead the report with `ALERT` because the authoritative snapshot is stale and still carries non-CID critical failures.
- Separate upstream input freshness from production-facing public artifact freshness.
- State explicitly that core public artifacts are **not** fresh enough for production use.
- After capturing evidence, terminate the tracked `refresh-all` process and any orphan `refresh-county-month-nws-flood-alerts.ts` children so later cron runs do not overlap.
