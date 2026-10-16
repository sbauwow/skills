# 2026-08-10 refresh-all silent stall with repeated NWS children

## Trigger pattern
- `npm run refresh:all` runs with **no stdout at all** for about 10 minutes.
- `public/cache/pipeline-health.json` does **not** update and remains on the prior completed snapshot.
- Process inspection shows the parent `refresh-all.ts` still alive, but the active children are repeated `refresh-county-month-nws-flood-alerts.ts` processes instead of the frontier advancing to later steps.

## Concrete evidence from the run
- Tracked background process uptime reached ~610s before manual termination.
- Live process set near termination:
  - `tsx scripts/refresh-all.ts`
  - repeated `tsx scripts/refresh-county-month-nws-flood-alerts.ts` children
- `pipeline-health.json` stayed at:
  - `generatedAt: 2026-08-09T18:00:27.604Z`
  - `overallStatus: degraded`
- The stale snapshot still recorded:
  - `refresh-county-month-precipitation` failed due to Open-Meteo `429`
  - `refresh-cid` failed due to missing Search Two seed input

## Mixed artifact freshness seen during the incomplete attempt
Artifacts that moved during the hung attempt:
- `public/cache/city-open-data-tx.json`
- `data/county-month-drought-usdm.json`
- `data/county-month-temperature-open-meteo.partial.json`
- `data/county-month-precipitation-open-meteo.json`

Core public artifacts that did **not** move and therefore remained stale for production:
- `public/cache/pipeline-health.json`
- `public/cache/city-open-data-ranked-tx.json`
- `public/cache/roadmap-open-data-botnet.json`
- `public/cache/surface-water-quality-tx.json`
- `public/cache/twdb-hydrology-tx.json`

## Interpretation rule
This is the **contrasting case** to the 2026-08-09 long-silent successful completion.

Do **not** classify every long-silent refresh as hung. Instead:
1. Check whether `pipeline-health.json` gets regenerated.
2. Check whether child processes advance beyond NWS flood alerts into later steps.
3. If the run stays silent, the step frontier does not advance, and only upstream/intermediate files move while core public artifacts stay stale, treat it as an **incomplete run**.
4. Report that production-facing artifacts are **not fresh enough for use**.
5. Kill the tracked background job and any orphan `tsx scripts/refresh-*` children after collecting evidence.
