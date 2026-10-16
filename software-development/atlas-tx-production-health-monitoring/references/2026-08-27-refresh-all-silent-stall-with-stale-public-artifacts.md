# 2026-08-27 refresh-all silent stall with stale public artifacts

## Summary
A scheduled `npm run refresh:all` run in `/home/stathis/atlas-tx` stayed completely silent for about 10 minutes and never rewrote `public/cache/pipeline-health.json`. The live child-process pattern stayed pinned on repeated `refresh-county-month-nws-flood-alerts.ts` children rather than advancing into later steps. Treat this as an incomplete run, not as a fresh degraded snapshot.

## Evidence
- Background `npm run refresh:all` process ran with no stdout for the whole observed window.
- Process tree/frontier snapshots:
  - Early: `refresh-all.ts` with `refresh-city-open-data.ts`, `refresh-county-month-drought.ts`, `refresh-county-month-temperature.ts`, and repeated `refresh-county-month-nws-flood-alerts.ts` children.
  - Later: collapsed to repeated `refresh-county-month-nws-flood-alerts.ts` children only.
- `public/cache/pipeline-health.json` did not move and remained on the prior completed snapshot:
  - file mtime: `2026-08-15T18:13:43Z`
  - top-level `generatedAt`: `2026-08-15T18:00:19.044Z`
  - `overallStatus`: `degraded`
- During the attempted run, only some upstream artifacts refreshed:
  - `public/cache/city-open-data-tx.json` refreshed to `2026-08-27T18:02:34Z`
  - `data/county-month-drought-usdm.json` refreshed to `2026-08-27T18:03:30Z`
  - `data/county-month-nws-flood-alerts.json` was fresh the same day (`2026-08-27T16:31:39Z`)
  - `data/county-month-precipitation-open-meteo.json` was fresh the same day (`2026-08-27T15:48:07Z`)
- Core downstream/public artifacts stayed stale:
  - `public/cache/pipeline-health.json` ~288h old
  - `public/cache/city-open-data-ranked-tx.json` ~105h old
  - `public/cache/roadmap-open-data-botnet.json` ~105h old
  - `public/cache/twdb-hydrology-tx.json` ~105h old
  - `public/cache/surface-water-quality-tx.json` ~288h old
  - `public/cache/analytics/*` ~288h old
- Cleanup required:
  - kill tracked `refresh-all` process
  - then kill remaining orphan `refresh-county-month-nws-flood-alerts.ts` child separately if still present

## Interpretation
- Do not infer a successful degraded refresh from fresh upstream `data/` files or the refreshed raw city catalog alone.
- If `pipeline-health.json` is still on the old 2026-08-15 snapshot and the frontier collapses to repeated NWS flood-alert children, report `ALERT`.
- The production verdict should explicitly say core public artifacts are **not refreshed recently enough for production use**.

## Reporting notes
- Separate two facts clearly:
  1. the last completed snapshot remains `overallStatus: degraded` with non-CID critical failures already recorded there; and
  2. today's attempted run was incomplete because it never emitted a new health snapshot.
- If asked for total run duration, report the visible duration from the last completed snapshot (`~13m24s` on the 2026-08-15 snapshot) and say there is no new completed duration for the hung attempt.
