# Handoff

Rolling session state for picking up work on this repo. Last updated
2026-11-17.

## Goal

A sample, for the IOOS sprint, of building a **virtual Icechunk store** from
NetCDF/HDF5 files already in object storage: an Icechunk repository that holds
only Zarr metadata and references to the original files, left where they are.

## Where things stand

- Repo created 2026-11-17: public, `ChrisBarker-NOAA/IOOS-Sprint-icechunk-demo`,
  cloned at `~/IOOS-Sprint-icechunk-demo`. Only a README on `main` so far.
- No store, notebook or code yet. Nothing has been decided about the build.
- Candidate source explored: ECCOFS on AWS (below).

## How we work

- A new branch for each task; work comes in through a pull request.
- Before merging a PR, summarize its changes and wait for Chris's approval.
- After a merge, ask whether to delete the branch.

(These are also in `~/.claude/CLAUDE.md` for Claude Code.)

## Tools on the hub

- Agent skills from
  [ocean-icechunks-skills](https://github.com/ocean-icechunks/ocean-icechunks-skills),
  cloned at `~/ocean-icechunks-skills` and linked into `~/.claude/skills`,
  `~/.agents/skills` and `~/.gemini/config/skills`:
  - `virtual-icechunk`: research → plan → smoke-test notebook → production
    build → docs, stopping for review between steps.
  - `archive-to-cloud`: copying files to object storage first. Not needed
    here; ECCOFS is already on S3.
- `gh` is logged in as ChrisBarker-NOAA.
- Python environment has `xarray` and `s3fs` (2026.9.0), `netCDF4`, `h5netcdf`
  and the `aws` CLI. VirtualiZarr and Icechunk versions not yet checked.

## Candidate data: ECCOFS, `s3://noaa-nos-eccofs-pds`

East Coast Coastal Ocean Forecast System, NOAA NOS. Public NODD bucket, read
anonymously (`--no-sign-request`, `anon=True`). Explored 2026-11-17 from the
listing and file headers only; no data downloaded.

- ROMS 4.3, curvilinear ~3 km grid, 1443 × 1667, 50 s-levels. Lon −102 to
  −36, lat −3 to 53. CF-1.4, SGRID-0.3, lon/lat for rho/u/v/psi points.
- One cycle per day, each an **8-day forecast**, so cycles overlap.
- 3,772 files, ~17.3 TB:

| Prefix | Contents | Per cycle | Format | Total | Dates |
|---|---|---|---|---|---|
| `his/` | full 3-D snapshot, daily at 00Z, days 0–8 | 9 × 4.6 GB | NetCDF3 64-bit offset | 6.2 TB | 2026-06-08 → 11-13 |
| `avg/` | full 3-D daily mean at 12Z | 8 × 7.6 GB | HDF5, uncompressed, chunks `[1,50,145,167]` | 9.0 TB | 2026-06-08 → 11-13 |
| `qck/` | surface + 3 depth slices, 3-hourly | 8 × ~1.6 GB | NetCDF3 | 2.0 TB | 2026-06-08 → 11-13 |
| `sta/` | 330 stations, 193 steps | 1 × 15–160 MB | NetCDF3 | 0.02 TB | from 2026-07-27 |
| `frc/` | atmospheric forcing | 9 × 0.2–1.3 GB | HDF5 | 0.17 TB | last ~3 weeks |
| `riv/`, `obs/` | rivers; CMEMS SSH/SST obs | small | HDF5 | small | last ~3 weeks |

File names: `<prefix>/eccofs_<prefix>_YYYYMMDD_NNNN.nc`, where `YYYYMMDD` is the
cycle date and `NNNN` the forecast day (e.g. `avg/..._20261112_0001.nc` is the
daily mean for 2026-11-12, `_0008` for 11-19).

Quirks seen:

- ~11 cycles missing between 06-08 and 11-13 (148 cycles in 159 days).
- Files land 3–4 days after their cycle date.
- `qck/` went from one file per cycle to 8 per cycle in late July 2026.
- `frc/`, `riv/`, `obs/` hold only ~3 weeks: probably a rolling window. Check
  before referencing them, since references to deleted files break.
- The bucket's `index.html` is a generic S3 browser, not documentation. Product
  documentation not yet found.

## Proposed next step (not yet agreed)

Build a virtual store from **`avg/`**, taking forecast day 1 (`_0001`) from each
cycle: a daily "best estimate" series from June to now, from chunked HDF5 that
virtualizes cleanly. Start with the `virtual-icechunk` skill's research-and-plan
step, then a smoke test on a few days.

## Open questions for Chris

- Is ECCOFS the dataset for the demo, and `avg/` the right product?
- One forecast day per cycle (a best-estimate series), or keep all forecast
  days with a cycle dimension? Overlapping forecasts (FMRC) are outside what the
  `virtual-icechunk` skill currently covers.
- Where should the Icechunk store live: local disk for the demo, or a bucket?
  If a bucket, which one and with what credentials?
- Who the audience is at the sprint, which decides notebook vs. script.
