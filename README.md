# Moraine dam surface change from Sentinel-1 InSAR

Analysis code for measuring surface deformation and mapping buried ice on
glacial-lake moraine dams in the Nepal Himalaya, using stacked Sentinel-1
interferograms and seasonal coherence change.

> **Status:** supports a manuscript in preparation. Citation and archived
> release DOI will be added on submission.
> <!-- TODO: manuscript citation -->
> <!-- TODO: Zenodo DOI for the tagged release -->
> <!-- TODO: license -->

## What the analysis does

For each moraine dam, ascending and descending Sentinel-1 burst
interferograms are clipped to the dam, then used two ways:

1. **Buried ice mapping.** Mean coherence is computed over a high-coherence
   (cold, snow-free) window and a low-coherence (warm) window. The seasonal
   difference is a proxy for buried ice: ice-cored ground loses coherence in
   summer as the ice deforms and the active layer thaws. Pixels above a
   threshold are mapped as buried ice.
2. **Surface velocity.** Short-baseline (12-day) warm-season interferograms
   above a coherence threshold are stacked to a median line-of-sight
   velocity per look direction, referenced to a stable off-dam polygon.
   Ascending and descending LOS velocities are then decomposed into
   vertical (up/down) and horizontal (east/west) components using the
   per-pixel look vectors.

Results across sites are aggregated in `all_dams_analysis/`.

## Repository layout

```
dams/<site>/md_processing.ipynb   per-site processing (23 sites, manuscript)
exploratory/<site>/               sites examined but not in the manuscript
mapping/polygons/                 hand-digitized analysis polygons
all_dams_analysis/                cross-site synthesis and figures
environment.yml                   conda environment
```

Sites are named by common name where one exists (`imja`, `barun`,
`thulagi`), and by ICIMOD glacial lake ID otherwise (`gakal_gl_0008`,
`kotam_gl_0111`, `kodud_gl_0205`).

### `mapping/polygons/`

Hand-digitized per site, as `<site>_<layer>.shp`:

| Layer            | Meaning                                                   |
| ---------------- | --------------------------------------------------------- |
| `md`             | Moraine dam extent. The primary analysis footprint.       |
| `moving`         | Subset of the dam showing coherent deformation.           |
| `stable`         | Off-dam terrain assumed stable; used to characterize noise.|
| `reference`      | Reference area; its median velocity is removed as a datum. |
| `bank_movement`  | Deformation on lake or outflow banks.                      |
| `landslide`      | Mapped landslide, where present.                           |

Not every site has every layer — sites with no detected motion have no
`moving` polygon, and the notebooks handle that. `all_dams.shp` is a
combined dam-extent layer for overview maps.

### `all_dams_analysis/`

| Notebook                       | Purpose                                              |
| ------------------------------ | ---------------------------------------------------- |
| `download_shadow_layover.ipynb`| Fetches OPERA RTC layover/shadow masks per burst and writes per-site validity masks. |
| `moving_area_statistics.ipynb` | Main aggregation: areas, coherence, velocity statistics per site. |
| `velocity_statistics.ipynb`    | Velocity summary tables and distribution figures.    |
| `ice_extent.ipynb`             | Buried ice area per dam.                             |
| `ice_mapping_sensitivity.ipynb`| Sensitivity of mapped ice area to the coherence-change threshold, including a Gaussian-mixture threshold estimate. |
| `imja_comparison.ipynb`        | Compares Imja velocities against the earlier time-series and DEM-differencing results (see below). |

`md_processing_decisions.csv` records the per-site processing choices:
burst IDs, the seasonal windows used for the coherence comparison, whether
pixel offsets were needed, and free-text notes on data quality. The
synthesis notebooks read the first nine columns of this file.

## Input data

The analysis consumes Sentinel-1 burst interferograms and pixel-offset
maps produced with **[fufiters](https://github.com/relativeorbit/fufiters)**
([10.5281/zenodo.17466768](https://doi.org/10.5281/zenodo.17466768)), which
runs a fork of ASF's `hyp3-isce2` over a given burst via GitHub Actions.

These products are **not redistributed with this repository** — the full
set is several hundred GB. To reproduce the analysis, regenerate them with
`fufiters` for the burst IDs listed in `md_processing_decisions.csv`, and
arrange them as:

```
data/
  data_igrams/<burst_id>/...     interferograms
  data_offsets/<burst_id>/...    pixel offsets
```

`data/` is gitignored. Notebooks reference it relative to their own
location, so no path editing is needed once the directory exists.

Burst IDs are given per site in `md_processing_decisions.csv` as
`asc_burst` and `des_burst`, in `<track>_<burst>_<subswath>` form.

## Environment

```bash
conda env create -f environment.yml
conda activate mintpy
```

The notebooks derive the PROJ data directory from the active environment,
so no environment-specific path editing is required.

## Running the analysis

Order matters — the synthesis notebooks read rasters written by the
per-site notebooks.

1. **Per site:** run `dams/<site>/md_processing.ipynb`. Each writes its
   outputs into its own directory:

   | Output                                | Contents                          |
   | ------------------------------------- | --------------------------------- |
   | `<site>_{asc,des,combined}_mean_coherence.tif` | Mean coherence         |
   | `<site>_seasonal_coherence_change.tif`| High minus low season coherence   |
   | `<site>_ice_map.tif`                  | Thresholded buried ice mask       |
   | `<site>_{asc,des}_median_velocity.tif`| LOS median velocity               |
   | `<site>_{asc,des}_velocity_nmad.tif`  | Velocity NMAD                     |
   | `<site>_{ud,ew,combined}_median_velocity.tif` | Decomposed velocity       |

2. **Validity masks:** run `all_dams_analysis/download_shadow_layover.ipynb`
   to write `<site>_{asc,des,combined}_valid.tif`. The per-site notebooks
   read these when masking layover and shadow, so sites may need a second
   pass through step 1.

3. **Synthesis:** run the `all_dams_analysis/` notebooks.
   `moving_area_statistics.ipynb` writes `md_df.csv`, an enriched table
   used by some downstream cells.

Rasters (`*.tif`), MintPy run directories and `*.h5` stacks are gitignored;
they are regenerated by the notebooks.

## Known issues and caveats

These are recorded honestly rather than silently patched, since some affect
how results should be read.

- **`water_mask.shp` is missing.** Seventeen per-site notebooks load
  `mapping/polygons/water_mask.shp` in their polygon cell. That file was
  never committed and is not available. The loaded `water_gdf` is not used
  anywhere downstream, so the line is dead code — but it will raise on a
  clean checkout and must be commented out before running. Six notebooks
  already have it commented.

- **The buried-ice threshold is not consistent between notebooks.** All 23
  per-site notebooks threshold seasonal coherence change at `>= 0.03` when
  writing `<site>_ice_map.tif`, and `moving_area_statistics.ipynb`
  recomputes ice area at `> 0.03`. But `ice_extent.ipynb` recomputes it at
  `> 0.04`. Buried ice area therefore differs depending on which notebook
  it is read from. `ice_mapping_sensitivity.ipynb` sweeps the threshold and
  highlights 0.04, and its Gaussian-mixture estimate gives a median of
  about 0.037 across sites — so both values are defensible, but one should
  be chosen and applied consistently before the numbers are quoted.

- **Two variants of the interferogram loader.** The `hyp3_to_xarray` cell
  exists in two forms across sites. The later form wraps the product glob
  in `try/except: continue`; the earlier form does not. The difference is
  only reached when a burst directory is missing a requested product — in
  which case the later form silently drops that acquisition from the stack
  rather than failing. Sites using the later variant: barun, dig,
  east_hongu_2, gakal_gl_0008, hongu_1, kekyap, kodud_gl_0205, lumding,
  muli_tal, sabai, tallo_kekyap.

- **`imja_comparison.ipynb` reads outside this repository.** It compares
  against velocity and DEM-differencing rasters from the earlier Imja
  study (Brencher, Henderson and Shean, *The Cryosphere*, 20, 67–86, 2026,
  [10.5194/tc-20-67-2026](https://doi.org/10.5194/tc-20-67-2026)). Those
  paths are absolute and will need to be repointed.

- **Exploratory notebooks are not cleaned.** The four
  `exploratory/*/md_processing_full.ipynb` notebooks still contain
  machine-specific absolute paths. They are kept for provenance, are not
  part of the manuscript, and are not expected to run as-is.

- **Coherence windows are per site.** The seasonal windows in
  `md_processing_decisions.csv` were chosen by inspecting each site's
  coherence time series, not by a uniform rule. Sites with long snow-cover
  periods have correspondingly shifted windows; see the `notes` column.
