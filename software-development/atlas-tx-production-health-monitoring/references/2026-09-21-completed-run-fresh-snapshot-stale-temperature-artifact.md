# 2026-09-21 — completed run, fresh snapshot, but stale fast-lane temperature artifact

Watchdog check at ~11:05 UTC found the opposite of the morning's crashed-run state:

- Site `/api/health`: HTTP 200, `ok:true`, dataset_count 21.
- `pipeline-health.json` regenerated at 09:11:58Z (mtime), `generatedAt` 09:01:13Z, `overallStatus: degraded`, 13/13 catalog steps covered.
- All step `endedAt` values were from the same morning run (<2h old) — by snapshot recency alone every lane looks fresh.

## The breach

`refresh-county-month-temperature` → `failed`, Open-Meteo archive 429, but its real artifact
`data/county-month-temperature-open-meteo.json` had mtime 569h (~23.7 days) old, i.e. last
successfully written ~2026-08-28. Weather → fast lane → 12h threshold: 23.7d >> 12h, so the fast
lane is stale **on real data** even though the step's `endedAt` was minutes old. Its
`.partial.json` was 480h (~20d) old. `refresh-county-month-precipitation` also failed with 429 in
this run, but its artifact was 7h old (written by an earlier same-day run) — under the 12h
threshold, so no alert for it yet, only a warning.

## Lessons

1. A brand-new snapshot with full step coverage can still hide a weeks-old fast-lane artifact.
   For every `failed` step, `stat` the `outputPath` (and `.partial.json`) and age it against the
   lane threshold; that — not `endedAt` — is what breaches policy.
2. A failed step does not automatically mean a stale artifact: check whether an earlier run the
   same day wrote it (precipitation was 7h old despite its own 429).
3. Persistent 429 rate-limiting on Open-Meteo archive API can keep one dataset stale for weeks
   while every scheduled run otherwise completes; the snapshot stays `degraded` with the same
   non-CID failure each time.
4. CID failed again for missing seed input (known intentional-degraded condition) — reported, not
   alerted on.