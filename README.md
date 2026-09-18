# Moraine dam surface change from Sentinel-1 InSAR

Analysis code for quantifying surface deformation and mapping buried ice within glacial lake moraine dams in the Nepal Himalaya, using stacked Sentinel-1 interferograms and seasonal InSAR coherence change.

This work builds on the InSAR framework of Brencher et al. (2026) with one principal change: surface velocity is estimated by **interferogram stacking** (Zebker et al., 1997) to capture peak late-summer velocity, rather than by full time-series inversion. Stacking mitigates the data gaps caused by seasonal snow and long intervals between Sentinel-1 acquisitions, at the cost of collapsing the temporal dimension.

## Method summary
### Data

Ascending and descending Sentinel-1 burst SLCs from 1 January 2017 to 15 May 2024 over each lake. Pre-2017 data are excluded because of long gaps before Sentinel-1B commissioning. Acquisitions are IW swath mode, VV polarization, 14.1 m azimuth by 2.3 m range. Layover and shadow masks come from the OPERA RTC Sentinel-1 static layers; topography is the 2021 Copernicus GLO-30 DEM.

### Interferometric processing

Interferograms and coherence maps were produced with the HyP3 ISCE2 plugin (v0.9.3) on HyP3, wrapping ISCE 2.6.3, via the `insar_tops_burst` workflow. Each burst SLC is paired with the three subsequent acquisitions, typically giving 12, 24 and 36-day temporal baselines (occasionally 6, 18, 30). Processing used five looks in range and one in azimuth, geocoded to 20 m
square pixels.

### Buried ice mapping

Warm-season melt of buried ice produces surface change that reduces InSAR coherence. Seasonal coherence change is therefore used as a proxy for buried ice. Delineation uses short (12-day) baselines, which show the largest seasonal coherence change over debris-covered ice. Median coherence across all 6–36 day baselines over the moving areas defines two windows per site:

- **Low-coherence (warm, snow-free)** — mean duration 60 days, DOY 228–288.
- **High-coherence (cold, snow-free)** — DOY 0–100 for the 7 sites with
  minimal snow, DOY 120–180 for the 16 sites with significant cold-season
  snow.

Seasonal coherence change is the difference between median coherence in the high- and low-coherence windows. Buried-ice pixels are identified by Gaussian mixture modeling of the seasonal coherence-change distribution: pixels without buried ice should be approximately Gaussian about zero, while buried ice forms a distinct positive population. Single- and two-Gaussian models are fit per dam (scikit-learn, maximum likelihood) and compared by BIC. Where two components are better supported, the threshold is the smallest coherence-change value at which the buried-ice population becomes more probable. The median threshold across all dams, 0.04, is then applied regionally. A single common value keeps sites consistent and allows detection of small (<5 pixel) ice areas that cannot themselves support a two-component fit.

Sensitivity to that choice is assessed by recomputing extent across thresholds 0.01–0.30 in steps of 0.01.

### Surface velocity

Velocity characterizes the warm, low-snow period. Interferograms with temporal baselines ≤12 days, formed from acquisitions between DOY 200 and 300, are masked for layover and shadow, and the mean displacement over the reference area is subtracted to suppress local atmospheric effects. Pixels below 0.6 coherence are discarded to limit unwrapping errors. LOS velocity
is displacement over temporal baseline; the per-pixel median is retained where at least five valid observations exist.

Ascending and descending LOS velocities are combined using per-burst azimuth and incidence angles to solve for east/west and vertical components. The acquisition geometry does not constrain north/south motion.

Zonal statistics: mean absolute velocity for LOS and east/west components; mean signed velocity for the vertical component, since subsidence is the expected signal; 95th percentile of absolute LOS velocity and 5th percentile of vertical velocity to characterize the fastest-moving areas without outlier sensitivity.

### Uncertainty

Buried-ice error is estimated as the count of pixels misclassified as ice within stable areas, assumed ice-free. Velocity error is the mean and standard deviation of apparent velocity in stable areas. Assuming no true motion, the mean is bias and the standard deviation is dispersion.

## Repository layout

```
dams/<site>/md_processing.ipynb   per-site processing (23 sites, manuscript)
exploratory/<site>/               sites examined but not in the manuscript
mapping/polygons/                 hand-digitized analysis polygons
all_dams_analysis/                cross-site synthesis and figures
environment.yml                   conda environment
```

Sites are named by common name where one exists (`imja`, `barun`, `thulagi`), and by ICIMOD glacial lake ID otherwise (`gakal_gl_0008`, `kotam_gl_0111`, `kodud_gl_0205`).

### `mapping/polygons/`

Hand-digitized per site as `<site>_<layer>.shp`, using multiple satellite basemaps (Bing, Google Satellite, ESRI World Imagery; accessed May–June 2024) together with the GLO-30 DEM.

| Layer            | Definition                                                 |
| ---------------- | ---------------------------------------------------------- |
| `md`             | Moraine dam: all non-bedrock material separating the lake from the downstream valley — i.e. material whose removal would allow the lake to drain. |
| `moving`         | Contiguous area within the dam where coherent LOS velocity exceeds 1 cm/yr. |
| `stable`         | Flat, high-coherence, near-zero-velocity terrain near the dam; used to quantify uncertainty. |
| `reference`      | As `stable`, but its mean displacement is subtracted to correct local atmospheric effects. |
| `bank_movement`  | Deformation on lake or outflow banks.                       |
| `landslide`      | Mapped landslide, where present.                            |

Reference and stable areas lie 193–2460 m from the dam centroid (mean 811 m). Not every site has every layer — sites with no detected motion have no `moving` polygon, and the notebooks handle that. `all_dams.shp` is a combined dam-extent layer for overview maps.

### `all_dams_analysis/`

| Notebook                       | Purpose                                              |
| ------------------------------ | ---------------------------------------------------- |
| `download_shadow_layover.ipynb`| Fetches OPERA RTC layover/shadow masks per burst; writes per-site validity masks. |
| `moving_area_statistics.ipynb` | Main aggregation: areas, coherence, velocity zonal statistics per site. |
| `velocity_statistics.ipynb`    | Velocity summary tables and distribution figures.    |
| `ice_extent.ipynb`             | Buried ice area per dam.                             |
| `ice_mapping_sensitivity.ipynb`| GMM threshold estimation per dam, and the 0.01–0.30 threshold sensitivity sweep. |
| `imja_comparison.ipynb`        | Compares Imja velocities against earlier time-series and DEM-differencing results. |

`md_processing_decisions.csv` records the per-site processing choices: burst IDs, the seasonal windows used for the coherence comparison, whether pixel offsets were needed, and free-text notes on data quality. The synthesis notebooks read its first nine columns.

## Input data

The analysis consumes Sentinel-1 burst interferograms and pixel-offset maps produced with [fufiters](https://github.com/relativeorbit/fufiters) ([10.5281/zenodo.17466768](https://doi.org/10.5281/zenodo.17466768)), which runs a fork of ASF's `hyp3-isce2` over a given burst via GitHub Actions.

These products are not redistributed here. To reproduce the analysis, regenerate them with `fufiters` for the burst IDs in `md_processing_decisions.csv` (given as `asc_burst` and `des_burst`, in `<track>_<burst>_<subswath>` form) and arrange them as:

```
data/
  data_igrams/<burst_id>/...     interferograms
```

## Environment

```bash
conda env create -f environment.yml
conda activate mintpy
```

## Running the analysis

1. **Per site:** run `dams/<site>/md_processing.ipynb`. Each writes its outputs into its own directory:

   | Output                                | Contents                          |
   | ------------------------------------- | --------------------------------- |
   | `<site>_{asc,des,combined}_mean_coherence.tif` | Mean coherence           |
   | `<site>_seasonal_coherence_change.tif`| High minus low season coherence   |
   | `<site>_ice_map.tif`                  | Buried ice mask, thresholded at 0.04 |
   | `<site>_{asc,des}_median_velocity.tif`| LOS median velocity               |
   | `<site>_{asc,des}_velocity_nmad.tif`  | Velocity NMAD                     |
   | `<site>_{ud,ew,combined}_median_velocity.tif` | Decomposed velocity       |

2. **Validity masks:** run `all_dams_analysis/download_shadow_layover.ipynb` to write `<site>_{asc,des,combined}_valid.tif`. Note the circular dependency: that notebook reads `<site>_asc_median_velocity.tif` to match grids, while the per-site notebooks read the validity masks to apply layover and shadow masking. A site therefore needs two passes through step 1.

3. **Synthesis:** run the `all_dams_analysis/` notebooks. `moving_area_statistics.ipynb` writes `md_df.csv`, an enriched table used by some downstream cells.

## Data availability

- **Derived data products from this study:**
  [10.5281/zenodo.22837426](https://doi.org/10.5281/zenodo.22837426)
- **Sentinel-1 SLC data:** freely available from the Alaska Satellite
  Facility DAAC, <https://asf.alaska.edu>. Per-site burst identifiers are in `md_processing_decisions.csv`.
- **Layover and shadow masks:** OPERA RTC Sentinel-1 static layers,
  [10.5067/SNWG/OPERA_L2_RTC-S1-STATIC_V1](https://doi.org/10.5067/SNWG/OPERA_L2_RTC-S1-STATIC_V1)
- **Copernicus GLO-30 DEM:**
  [10.5270/ESA-c5d3d65](https://doi.org/10.5270/ESA-c5d3d65)
- **Glacial lake inventory:** Mool et al. (2011).

## References

ASF HyP3 Team (2023). HyP3 ISCE2 plugin, v0.9.3.

Brencher, G., Henderson, S. T., and Shean, D. E. (2026). Quantifying degradation of the Imja Lake moraine dam with fused InSAR and SAR feature tracking time series. *The Cryosphere*, 20, 67–86. <https://doi.org/10.5194/tc-20-67-2026>

European Space Agency (2021). Copernicus GLO-30 Digital Elevation Model.

Hogenson, K. et al. (2020). Hybrid Pluggable Processing Pipeline (HyP3).

Mool, P. K. et al. (2011). Glacial lake inventory. ICIMOD.

OPERA (2023). Radiometric Terrain Corrected Sentinel-1 products.

Pedregosa, F. et al. (2011). Scikit-learn: machine learning in Python. *JMLR*, 12, 2825–2830.

Rosen, P. A. et al. (2012). The InSAR Scientific Computing Environment (ISCE), v2.6.3.

Schwarz, G. (1978). Estimating the dimension of a model. *Annals of Statistics*, 6, 461–464.

Zebker, H. A., Rosen, P. A., and Hensley, S. (1997). Atmospheric effects in interferometric synthetic aperture radar surface deformation and topographic maps. *JGR: Solid Earth*, 102, 7547–7563.
