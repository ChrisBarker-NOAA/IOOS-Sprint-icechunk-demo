# ECCOFS on AWS: what's in `s3://noaa-nos-eccofs-pds`

Notes from exploring the bucket on 2026-11-17, as a candidate source for a
virtual Icechunk store. Everything here comes from the bucket listing and from
reading file headers and coordinate variables; no bulk data was downloaded.
The bucket changes daily, so counts and date ranges are a snapshot.

## What it is

The **East Coast Coastal Ocean Forecast System (ECCOFS)**, an operational NOAA
NOS ocean forecast model, published as a public NOAA Open Data Dissemination
(NODD) bucket.

- **Access:** public and anonymous: no AWS account or credentials needed.
- **Model:** ROMS 4.3 (from the `history` attribute).
- **Grid:** curvilinear, about 3 km (grid file named
  `grid_eccofs_3km_08_b7_w2.nc`), 1443 × 1667 rho points, 50 terrain-following
  (s-coordinate) levels.
- **Domain:** longitude −101.9° to −36.1°, latitude −3.4° to 53.4°: the US
  East Coast and Gulf of Mexico, the Caribbean, and a large part of the western
  North Atlantic.
- **Conventions:** `CF-1.4, SGRID-0.3`. Each file carries `lon_*`/`lat_*` for
  the rho, u, v and psi points of the staggered (Arakawa C) grid.
- **Time:** `ocean_time`, in `seconds since 2011-01-01 00:00:00`.
- **Forecast structure:** one forecast cycle per day, each running **8 days**
  ahead, so cycles overlap: a given date appears in up to 8 cycles.
- **Data assimilation:** the `obs/` prefix holds CMEMS sea surface height and
  temperature observations, which suggests the system assimilates data. Not
  confirmed from documentation.

The bucket's `index.html` is a generic S3 file browser, not product
documentation. Official ECCOFS documentation was not looked up yet.

## Layout

3,772 files, about 17.3 TB, under seven prefixes:

| Prefix | Contents | Files per cycle | Format | Total size | Dates covered |
|---|---|---|---|---|---|
| `avg/` | Full 3-D daily means | 8 × 7.6 GB | NetCDF4/HDF5 | 9.0 TB | 2026-06-08 → 2026-11-13 |
| `his/` | Full 3-D snapshots ("history") | 9 × 4.6 GB | NetCDF3, 64-bit offset | 6.2 TB | 2026-06-08 → 2026-11-13 |
| `qck/` | "Quick" files: surface fields and depth slices | 8 × ~1.6 GB | NetCDF3, 64-bit offset | 2.0 TB | 2026-06-08 → 2026-11-13 |
| `sta/` | Time series at stations | 1 × 15–160 MB | NetCDF3, 64-bit offset | 0.02 TB | 2026-07-27 → 2026-11-13 |
| `frc/` | Atmospheric forcing, one file per variable | 9 × 0.2–1.3 GB | NetCDF4/HDF5 | 0.17 TB | 2026-10-24 → 2026-11-13 |
| `riv/` | River inputs | 1 × 0.75 MB | NetCDF4/HDF5 | < 0.01 TB | 2026-10-24 → 2026-11-13 |
| `obs/` | CMEMS SSH and SST observations | 2 × 10–50 MB | NetCDF4/HDF5 | < 0.01 TB | 2026-10-24 → 2026-11-13 |

### File names

```
<prefix>/eccofs_<prefix>_YYYYMMDD_NNNN.nc     avg, his, qck (since late July 2026)
<prefix>/eccofs_<prefix>_YYYYMMDD.nc          sta, riv, older qck
frc/eccofs_frc_<variable>_YYYYMMDD.nc         Pair, Qair, Tair, Uwind, Vwind,
                                              lwrad, lwrad_down, rain, swrad
obs/eccofs_obs_ssh_sst_cmems_v2_YYYYMMDD[_3km].nc
```

`YYYYMMDD` is the forecast cycle date. `NNNN` counts forecast days from 0001.
Times confirmed by reading `ocean_time` for the 2026-11-12 cycle:

| File | `ocean_time` |
|---|---|
| `his/eccofs_his_20261112_0001.nc` | 2026-11-12 00:00 |
| `his/eccofs_his_20261112_0002.nc` | 2026-11-13 00:00 |
| `his/eccofs_his_20261112_0009.nc` | 2026-11-20 00:00 |
| `avg/eccofs_avg_20261112_0001.nc` | 2026-11-12 12:00 (mean of 11-12) |
| `avg/eccofs_avg_20261112_0008.nc` | 2026-11-19 12:00 |
| `qck/eccofs_qck_20261112_0001.nc` | 2026-11-12 00:00 to 11-13 00:00, 3-hourly (9 steps) |
| `qck/eccofs_qck_20261112_0008.nc` | 2026-11-19 03:00 to 11-20 00:00, 3-hourly (8 steps) |

So for one cycle: `his/` has 9 daily snapshots (00Z, day 0 to day 8), `avg/`
has 8 daily means (centred at 12Z), and `qck/` has 3-hourly output across the
same 8 days.

## The main products

### `avg/`: daily means (NetCDF4/HDF5)

One time step per file. 115 variables, including:

- 3-D: `temp`, `salt`, `rho`, `u`, `v`, `w`, `pvorticity`, `rvorticity`,
  `AKv`, `AKt`, `AKs`
- 2-D: `zeta`, `ubar`, `vbar`, `shflux`, `ssflux`, `latent`, `sensible`,
  `lwrad`, `swrad`, `evaporation`, `rain`, `sustr`, `svstr`, `ohc`, `mld`
- static: `h`, `f`, `pm`, `pn`, `angle`, `mask_rho`/`_u`/`_v`/`_psi`, lon/lat

Storage: `float32`, **uncompressed**, HDF5-chunked. `temp` is chunked
`[1, 50, 145, 167]` (one time step, all levels, about a 10 × 10 grid of tiles).

### `his/`: snapshots (NetCDF3)

One time step per file. 109 variables, much like `avg/` without the vorticity,
`ohc`, `mld` and some flux terms, plus `Pair`, `EminusP`, `bustr`, `bvstr`, and
`z_slice`-related fields. `float32`. NetCDF3 64-bit-offset format, so no
compression and no chunking.

### `qck/`: surface and slices (NetCDF3)

Several time steps per file, 3-hourly. Fields: `zeta`, `ubar_eastward`,
`vbar_northward`, `u_sur_eastward`, `v_sur_northward`, `temp_sur`, `salt_sur`,
and `u_eastward_slice`, `v_northward_slice`, `temp_slice`, `salt_slice` on 3
depth slices (`z_slice`). Velocities are already rotated to east/north.

### `sta/`: stations (NetCDF3)

One file per cycle: 330 stations × 193 time steps.

## Quirks

- **Missing cycles:** 148 cycles between 2026-06-08 and 2026-11-13, which span
  159 days, so about 11 are missing.
- **Delay:** files appear 3–4 days after their cycle date (the 2026-11-12 files
  were written 2026-11-16).
- **Naming change:** `qck/` was one file per cycle (`eccofs_qck_YYYYMMDD.nc`,
  about 12 GB, plus some `_fwd.nc` files) until 2026-07-25, then split into 8
  per cycle.
- **Short retention for some prefixes:** `frc/`, `riv/` and `obs/` cover only
  the last ~3 weeks, and `sta/` starts 2026-07-27. This looks like a rolling
  window, but it's not confirmed. Files that may be deleted are a risk for a
  virtual store, because its references would break.
- **Mixed formats:** `avg/`, `frc/`, `riv/` and `obs/` are HDF5; `his/`, `qck/`
  and `sta/` are NetCDF3.

## What this means for a virtual Icechunk store

- **`avg/` is the easiest place to start.** HDF5 with real chunks maps directly
  onto Zarr chunk references, and a reader can fetch one tile without reading
  the whole field.
- **NetCDF3 products are harder.** VirtualiZarr reads them with a different
  parser. Each variable is stored contiguously, so without splitting it into
  smaller references, reading any part means fetching a whole 3-D field
  (hundreds of MB).
- **Overlapping forecasts need a decision.** Concatenating all files along time
  gives duplicate timestamps. The options are a "best estimate" series (for
  example forecast day 1, `_0001`, from every cycle), or keeping both a cycle
  dimension and a forecast-lead dimension (an FMRC-style layout). The
  `virtual-icechunk` skill does not yet cover the second.
- **Gaps** in the cycles show up as gaps in a best-estimate time series.
- **No compression** in either format, so the source is large. A virtual store
  keeps it in place and adds only metadata.

## How to look at it yourself

List the bucket:

```bash
aws s3 ls --no-sign-request s3://noaa-nos-eccofs-pds/
aws s3 ls --no-sign-request s3://noaa-nos-eccofs-pds/avg/ | tail
```

Open one file's header over HTTPS without downloading it (netCDF-C byte-range
mode; works for both the NetCDF3 and the HDF5 files):

```python
import xarray as xr

url = "https://noaa-nos-eccofs-pds.s3.amazonaws.com/avg/eccofs_avg_20261112_0001.nc#mode=bytes"
ds = xr.open_dataset(url, engine="netcdf4")
print(ds)
```

Or through `s3fs` anonymously:

```python
import s3fs

fs = s3fs.S3FileSystem(anon=True)
fs.ls("noaa-nos-eccofs-pds/his")[:5]
```

## Not yet checked

- Official ECCOFS documentation (product description, retention policy, update
  schedule).
- Whether the `his/` day-0 snapshot is an analysis (after assimilation) or a
  forecast.
- What the `qck/` `_fwd.nc` files were.
- Variable attributes in detail (units, `_FillValue`, `coordinates`), and
  whether static variables are identical across files.
- Whether VirtualiZarr can parse these files and which versions work.
