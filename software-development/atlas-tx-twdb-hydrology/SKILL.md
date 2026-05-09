---
name: atlas-tx-twdb-hydrology
description: Refresh and use Atlas TX TWDB hydrology context layers (major aquifers, river basins, HUC8), including the bbox-based cache format, refresh script, and county-summary wiring.
version: 0.1.0
author: Hermes Agent
license: MIT
---

# Atlas TX TWDB Hydrology Workflow

Use this when working in the Atlas TX repo on TWDB hydrology context layers, especially:
- `src/lib/datasets/twdb-hydrology.ts`
- `scripts/refresh-twdb-hydrology.ts`
- county-summary / county-breakdown hydrology context wiring
- tests around TWDB hydrology cache refresh and normalization

## What this workflow does

Atlas TX now has a first-pass TWDB hydrology ingestion slice for three statewide layers:
- `twdb-major-aquifers`
- `twdb-river-basins`
- `twdb-huc8`

The workflow supports:
1. refreshing the committed cache from live TWDB zip downloads
2. normalizing DBF attribute rows + SHP record bounding boxes into a compact JSON snapshot
3. exposing county hydrology context using county-centroid overlap against cached feature bounding boxes

## Important design decision

The current cache is intentionally compact.
It stores:
- layer id/name
- normalized key fields
- feature bounding box
- geometry type
- source URL

It does NOT store full polygon geometry.

That means:
- refresh is lighter
- committed snapshot stays manageable
- county hydrology context is approximate
- map rendering / true spatial overlay still needs a future full-geometry slice

## Current files involved

Data / refresh:
- `src/lib/datasets/twdb-hydrology.ts`
- `scripts/refresh-twdb-hydrology.ts`
- `public/cache/twdb-hydrology-tx.json`

Tests:
- `tests/twdb-hydrology.test.ts`
- `tests/refresh-twdb-hydrology.test.ts`
- `tests/package-scripts.test.ts`
- `tests/atlas-county-explorer.test.ts`

Docs/contracts:
- `docs/contracts/dataset-registry.md`
- `docs/STATE.md`

Registry:
- `src/lib/mvp-datasets.ts`

## Runtime commands

Refresh the cache from live TWDB downloads:
```bash
npm run refresh:twdb-hydrology
```

Targeted tests:
```bash
npm test -- --run tests/package-scripts.test.ts tests/refresh-twdb-hydrology.test.ts tests/twdb-hydrology.test.ts tests/atlas-county-explorer.test.ts
```

Full verification:
```bash
npm test
npm run build
```

## Refresh-script behavior

The refresh script:
- downloads the three TWDB zip archives
- uses system `unzip`
- reads DBF rows directly in TypeScript/Node
- reads SHP record bounding boxes directly from the binary file
- checks DBF row count matches SHP record count
- normalizes via `normalizeTwdbHydrologyRawSnapshot(...)`
- writes `public/cache/twdb-hydrology-tx.json`

Current source URLs:
- `https://www.twdb.texas.gov/mapping/gisdata/doc/major_aquifers.zip`
- `https://www.twdb.texas.gov/mapping/gisdata/doc/Major_River_Basins_Shapefile.zip`
- `https://www.twdb.texas.gov/mapping/gisdata/doc/USGS_HUC_8_Shapefile.zip`

## Snapshot shape

The normalized row shape is:
```ts
export type TwdbHydrologyRow = {
  layerId: "twdb-major-aquifers" | "twdb-river-basins" | "twdb-huc8";
  layerName: string;
  primaryCode: string | null;
  name: string | null;
  basin: string | null;
  region: string | null;
  subregion: string | null;
  bbox: [number, number, number, number];
  geometryType: "polygon" | "other";
  sourceUrl: string;
};
```

Layer-specific mapping:
- `twdb-major-aquifers` → `AQUIFER`, `AQ_NAME`
- `twdb-river-basins` → `basin_num`, `basin_name`
- `twdb-huc8` → `HUC_8`, `SUBBASIN`, `BASIN`, `REGION`, `SUBREGION`

## County-summary wiring

`src/lib/atlas-county-explorer.ts` now accepts:
- `hydrologyRowsLoader?: () => Promise<TwdbHydrologyRow[]>`

Default behavior loads from the committed snapshot.

County breakdowns now return `hydrologyContext` with:
- `countyCentroid`
- `layerHits`
- `matches`
- `caveat`

Current matching logic is:
- take county centroid
- check whether centroid falls inside each cached row bbox
- count matches by layer
- return matched rows as context

This is explicitly approximate and should be described as such.

## TDD path that worked

1. Add a failing script-registration test in `tests/package-scripts.test.ts`
2. Add the npm script and re-run
3. Add a failing refresh-script test in `tests/refresh-twdb-hydrology.test.ts`
4. Implement the minimum refresh script to pass
5. Add a failing county-breakdown hydrology-context test in `tests/atlas-county-explorer.test.ts`
6. Implement the minimum centroid/bbox matching to pass
7. Re-run targeted tests
8. Re-run full suite and build

## Pitfalls

- Do not claim polygon-level spatial accuracy; this slice uses bbox + centroid approximation.
- Do not skip the DBF/SHP row-count consistency check in the refresh script.
- The refresh script currently depends on system `unzip`; if that changes, document it.
- Some rows legitimately have `name: null` after normalization; do not assume every row has a label.
- For HUC8 rows, source data may have imperfect labels/spelling; preserve source-derived values unless you intentionally add a cleanup layer.

## Good next step

If Atlas TX needs map rendering or stronger overlay logic, the next step is:
- add a full-geometry export path, probably for `twdb-huc8` first
- then replace centroid/bbox approximation with true polygon intersection logic

## Verification checklist

Before shipping changes in this area:
- `npm run refresh:twdb-hydrology` succeeds
- `tests/package-scripts.test.ts` passes
- `tests/refresh-twdb-hydrology.test.ts` passes
- `tests/twdb-hydrology.test.ts` passes
- `tests/atlas-county-explorer.test.ts` passes
- full `npm test` passes
- `npm run build` passes
- `docs/contracts/dataset-registry.md` reflects current behavior
