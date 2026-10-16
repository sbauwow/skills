# Freshness policy and step coverage notes

Derived during the 2026-05-21 Atlas TX production watchdog run.

## Why this reference exists
`public/cache/pipeline-health.json` can look superficially valid while still omitting expected ingest steps. Future health checks should compare the snapshot against the current ingest catalog before declaring the pipeline fresh.

## Authoritative files
- `src/lib/atlas-ingest-orchestrator.ts` — current task catalog and family assignments
- `scripts/refresh-all.ts` — pipeline-health schema and overall-status logic

## Family to lane mapping used in ops reports
- `water-core` → `context`
- `catalog` → `fast`
- `weather` → `fast`
- `roadmap-open-data` → `fast`
- `analytics` → `intelligence`
- `compliance` → `intelligence`
- `community` → `intelligence`

## Freshness thresholds
- fast: older than 12h = alert
- intelligence: older than 24h = alert
- context: older than 21d = alert

## Concrete incomplete-snapshot example
Observed snapshot:
- `generatedAt`: `2026-05-10T03:00:00.000Z`
- `overallStatus`: `degraded`
- only 6 step entries present

Current catalog expected 13 steps:
1. `refresh-twdb-hydrology`
2. `refresh-surface-water-quality`
3. `refresh-city-open-data`
4. `refresh-city-open-data-curated`
5. `refresh-city-open-data-ranked`
6. `refresh-analytics-history`
7. `refresh-county-month-precipitation`
8. `refresh-county-month-streamflow`
9. `refresh-county-month-drought`
10. `refresh-county-month-temperature`
11. `refresh-county-month-nws-flood-alerts`
12. `refresh-roadmap-open-data`
13. `refresh-cid`

Missing from the snapshot during that run:
- `refresh-analytics-history`
- `refresh-county-month-precipitation`
- `refresh-county-month-streamflow`
- `refresh-county-month-drought`
- `refresh-county-month-temperature`
- `refresh-county-month-nws-flood-alerts`
- `refresh-roadmap-open-data`

## Interpretation note
In `scripts/refresh-all.ts`, `overallStatus` becomes `degraded` if any failed step is `refresh-cid`; otherwise failed steps promote the run to `failed`. This means a report can be degraded because CID failed even when the site endpoint itself is healthy.
