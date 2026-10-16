# 2026-09-21 evening watchdog: manual off-peak temperature retry also 429'd

Context: third consecutive daily run with `refresh-county-month-temperature` failing on Open-Meteo 429 (after the ~11:05Z run and the ~18:02Z evening run documented in the sibling references). Per the escalation rule, a manual off-peak single-step retry was attempted instead of waiting for the next cron.

## Evidence
- `https://atlastexas.org/api/health` → HTTP 200, `{ok:true, dataset_count:21}` at 21:06:22Z.
- Snapshot regenerated 2026-09-21 18:50:46Z (file mtime; `generatedAt` 18:02:01Z marks run start). 13/13 catalog coverage, `overallStatus: degraded`.
- Completed run that evening: context lane repaired (TWDB + surface-water-quality + analytics all ok, ~2h old), ranked catalog and roadmap ok.
- Failed steps: temperature (429, artifact `data/county-month-temperature-open-meteo.json` 579h ≈ 24.1d), precipitation (429, artifact 17.4h — prior same-day run keeps it near threshold), NWS flood alerts (failed after 2559s, artifact still fresh at 7h), CID (missing seed input, chronic).
- Manual retry at ~21:09Z: `npm run refresh:county-month-temperature` → exit 1, same `Open-Meteo temperature archive request failed (429)` for the 2020-01-01..2025-12-31 archive request. Artifact unchanged.

## Lesson
- A manual off-peak retry the SAME DAY can still be rate-limited: the Open-Meteo 429 for this step persisted across morning cron (~11:05Z), evening cron (~18:02Z), and a ~21:09Z manual attempt. Escalating to a manual retry is not guaranteed recovery; treat a same-day 429 as quota/rate-limit persistence, not transient noise.
- Next remediation tier when the manual retry also 429s: reduce request weight (split the 6-year date range into yearly chunks with inter-chunk sleeps) or move to a different daily window — do not keep re-firing the identical full-range request, and do not modify the fetch script from the watchdog itself; hand the finding to the operator.
