# 2026-08-06 refresh-all timeout with stale health snapshot

Observed watchdog pattern while running `npm run refresh:all` from `/home/stathis/atlas-tx`.

## What happened
- Top-level command timed out after `600s` with no useful log output beyond startup.
- During the attempted run, some upstream artifacts refreshed:
  - `public/cache/city-open-data-tx.json`
  - `data/county-month-drought-usdm.json`
  - `data/county-month-temperature-open-meteo.partial.json`
- Key production-facing artifacts did **not** refresh:
  - `public/cache/pipeline-health.json`
  - `public/cache/city-open-data-ranked-tx.json`
  - `public/cache/roadmap-open-data-botnet.json`
- `public/cache/pipeline-health.json` remained on the old `2026-05-25` snapshot with `overallStatus: degraded`.
- The stale snapshot still showed non-CID failures in:
  - `refresh-county-month-precipitation`
  - `refresh-county-month-temperature`
- The snapshot also showed CID failure in `refresh-cid`, but the operator conclusion should still key separately on the non-CID failures.

## Important interpretation
- This is an **incomplete run**, not a successful degraded refresh.
- Fresh upstream `data/` files or a freshly rewritten `city-open-data-tx.json` do **not** make the production state healthy when `pipeline-health.json` and fast-lane public artifacts stay stale.
- A timed-out run may leave a transient child refresh process visible during inspection and then self-exit by the time cleanup happens. Even if no orphan processes remain on the second check, still report the run as incomplete if the health snapshot was not regenerated.
- Core public artifacts were **not** fresh enough for production use because the health snapshot, ranked catalog, and roadmap snapshot were still ~73 days old.

## Useful facts from this incident
- Stale snapshot `generatedAt`: `2026-05-25T18:00:15.430Z`
- Visible whole-run duration inside the stale snapshot: about `591.768s`
- Recent-but-insufficient artifacts during the timed-out attempt:
  - `city-open-data-tx.json` ~8 minutes old at inspection time
  - `county-month-drought-usdm.json` ~8 minutes old at inspection time
  - `county-month-temperature-open-meteo.partial.json` ~7 minutes old at inspection time
- Other public artifacts may be fresher from earlier independent runs (for example TWDB hydrology and surface-water-quality at ~24h old) without changing the incomplete verdict for the just-run `refresh:all`.
