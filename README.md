# Roman WFI Persistence Study Pipeline

A Jupyter notebook pipeline for simulating and analyzing detector persistence effects in the Nancy Grace Roman Space Telescope Wide Field Instrument (WFI), using the F146 filter and MA Table 1002.

The pipeline simulates a time series of exposures of a single point source, runs each through the `romancal` calibration pipeline, performs aperture photometry, and tracks pixel-level flux variations across all resultants to characterize persistence behavior.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Installation](#installation)
- [Directory Structure](#directory-structure)
- [Configuration](#configuration)
- [Pipeline Steps](#pipeline-steps)
- [Output Files](#output-files)
- [Analysis Cells](#analysis-cells)
- [Key Functions and Classes](#key-functions-and-classes)
- [Photometric Calibration](#photometric-calibration)
- [Known Issues and Notes](#known-issues-and-notes)

---

## Overview

This pipeline:

1. Creates a synthetic star catalog at a fixed sky position
2. Simulates Roman WFI L1 exposures using `romanisim`, with persistence carried forward between exposures
3. Optionally replaces saturated pixels before passing files to `romancal`
4. Runs each L1 file through the `romancal` `ExposurePipeline` to produce L2 calibrated images
5. Performs aperture photometry on each L2 image
6. Tracks DN values for an ensemble of PSF-wing pixels across all resultants and exposures
7. Produces flux matrices, photometric light curves, persistence metrics, and variance analyses

The observation setup is:
- **Duration**: 0.9 days (~108 exposures at 12-minute cadence)
- **Filter**: F146
- **Detector**: WFI01 (SCA 1)
- **MA Table**: 1002 (custom 5-resultant read pattern)
- **Star position**: RA = 79.922971°, Dec = 30.034266° (sensor center)

---

## Requirements

### Python packages

```
romanisim
romancal
roman_datamodels
asdf
galsim
crds
astropy
photutils
numpy
pandas
matplotlib
scipy
```

### Environment

- A working CRDS cache with a valid Roman context (e.g. `roman_0171.pmap`)
- Set the following environment variables before running:

```bash
export CRDS_PATH=/path/to/your/crds_cache
export CRDS_SERVER_URL=https://roman-crds.stsci.edu
export CRDS_CONTEXT=roman_0171.pmap
```

---

## Installation

It is recommended to use the same conda environment used for `romanisim` development. Refer to the [romanisim installation guide](https://romanisim.readthedocs.io) for setup instructions.

---

## Directory Structure

After running the pipeline, the following directories are created automatically:

```
star_mag_18.00/
├── star18.00.ecsv                  # Source catalog
├── checkpoint.json                 # Resume checkpoint
├── analysis_results/
│   ├── l1_metrics_Mag_18.00.csv
│   ├── Photometry_Mag_18.00.csv
│   ├── persistence_Mag_18.00.csv
│   ├── multi_pixel_resultant0_unsat.csv
│   ├── multi_pixel_resultant1_unsat.csv
│   ├── ...
│   ├── flux_matrix_resultant0_MAG_18.00.csv
│   ├── ...
│   └── [plots and summary CSVs]
└── temp_l1_processing/
    └── [temporary L1/L2 ASDF files, cleaned up each iteration]
```

---

## Configuration

All user-facing parameters are set in **Cell 3** and **Cell 7**. Edit these before running.

### Cell 3 — Observation Parameters

| Variable | Default | Description |
|---|---|---|
| `MAG` | `18` | Star magnitude for simulation |
| `USE_SATURATION_REMOVAL` | `True` | Replace saturated pixels in L1 before pipeline |
| `RA`, `DEC` | `79.922971, 30.034266` | Sky position of the star |
| `OBSERVATION_PERIOD_DAYS` | `0.9` | Total observation window |
| `CADENCE_MINUTES` | `12` | Time between exposures |
| `N_ENSEMBLE_PIXELS` | `1600` | Number of PSF-wing pixels to track (40×40 box) |
| `ENSEMBLE_MAX_RADIUS` | `30` | Half-width of ensemble pixel box (pixels) |

### Cell 7 — Global Configuration

| Variable | Description |
|---|---|
| `RESULTANT_TIMES` | Dict mapping resultant index → cumulative time in seconds |
| `RESULTS_DIR` | Output directory for analysis CSVs and plots |
| `TEMP_L1_DIR` | Scratch directory for intermediate ASDF files |
| `ZERO_POINT_AB` | AB magnitude zero point for F146 (28.08) |
| `COUNTS_TO_JY` | DN → Jansky conversion factor |
| `photfnu` | Alternative flux conversion constant (MJy/sr × pixel area) |

### Cell 15 — Optional Intermediate Outputs

These are disabled by default for speed:

```python
EXTRACT_JUMPS = False        # Save cosmic ray detection stats
EXTRACT_WCS = False          # Save per-source RA/Dec
EXTRACT_RAMP_QUALITY = False # Save ramp fit chi-squared
EXTRACT_DQ_FLAGS = False     # Save DQ flag percentages
```

---

## Pipeline Steps

Run cells in order. The main processing loop is in **Cell 35**.

### Step 1 — Set Variables (Cell 3)
Set `MAG`, cadence, observation window, and ensemble configuration.

### Step 2 — Create Catalog (Cell 5)
Generates a single-star ECSV catalog at the sensor center in the format expected by `romanisim`.

### Step 3 — Load Functions (Cells 9–31)
Defines all helper functions and the `RomanL1ResultantAnalyzer` class. Must be run before Step 4.

### Step 4 — Pipeline Setup (Cell 33)
Creates output directories and initializes all CSV files with correct column headers.

### Step 5 — Run Main Loop (Cell 35)
Processes all `N_EXPOSURES` files. For each exposure:

1. **Generate L1** — calls `romanisim` with persistence state from the previous L1 file
2. **Saturation handling** — if `USE_SATURATION_REMOVAL=True`, identifies saturated pixels (from CRDS reference or fallback threshold of 55,000 DN) and replaces them with `NaN` before pipeline processing
3. **Run romancal** — passes the cleaned L1 through `ExposurePipeline` to produce an L2 calibrated image
4. **Photometry** — aperture photometry (r=10 px aperture, r_in=15/r_out=25 px annulus background) on the L2 image using the fixed star position
5. **Pixel tracking** — records DN values for all ensemble pixels across all resultants to per-resultant CSV files
6. **Persistence detection** — compares each L2 image to a reference image to flag persistence-affected pixels
7. **Cleanup** — deletes intermediate ASDF files, keeping only the current L1 (for next iteration's persistence) and current L2 (for persistence comparison)
8. **Checkpoint** — saves progress after every file for safe resumption

#### Resuming an interrupted run
The pipeline automatically resumes from where it left off. On restart, simply re-run Cell 35 — it reads `checkpoint.json` and starts from `last_completed + 1`.

---

## Output Files

| File | Description |
|---|---|
| `l1_metrics_Mag_X.csv` | Per-exposure L1 metrics: saturation fraction, slopes, timing |
| `Photometry_Mag_X.csv` | Per-exposure aperture photometry: flux, magnitude, SNR, errors |
| `persistence_Mag_X.csv` | Per-exposure persistence metrics: affected pixel count, strength |
| `multi_pixel_resultant{N}_{mode}.csv` | Per-pixel DN values at resultant N for all exposures |
| `flux_matrix_resultant{N}_MAG_X.csv` | Wide-format flux matrix: rows = exposures, columns = pixels |
| `ensemble_analysis_all_resultants_summary.csv` | RMS precision per resultant |
| `pixel_statistics.csv` | Per-pixel RMS variation in mmag |
| `outlier_pixels.csv` | Pixels with variation >2× their spatial neighbors |

---

## Analysis Cells

After the main loop completes, the following analysis cells can be run independently:

| Cell | Description |
|---|---|
| 37 | Load result CSVs into `l1_df`, `phot_df`, `pers_df` — **run this first** |
| 38 | Summary plot: slopes, photometry, persistence vs time |
| 40–41 | Uncertainty analysis: flux and magnitude error vs image step |
| 43 | Persistence detail plots: fraction, strength, max, pixel count vs time |
| 44 | L2 image visualization: full frame, cutout, radial profile |
| 46 | Ensemble magnitude comparison: saturated vs unsaturated |
| 47 | Individual pixel magnitude time series |
| 49 | Pixel-by-pixel Δmag analysis and outlier detection |
| 50 | Ensemble fractional variation across all resultants |
| 52 | Flux matrix creation for all resultants |

---

## Key Functions and Classes

### `RomanL1ResultantAnalyzer` (Cell 19)
Loads a Roman L1 ASDF file and extracts ramp data, MA table structure, and timing.

Key attributes after loading:
- `analyzer.ramp_data` — shape `(n_resultants, 4088, 4088)`
- `analyzer.resultant_times` — cumulative time per resultant in seconds
- `analyzer.exposure_metadata` — dict with detector, MA table, timing info

### `run_romanisim_simulationapi` (Cell 17)
Generates a Roman L1 ASDF file using the `romanisim` Python API.

```python
l1_path = run_romanisim_simulationapi(
    catalog_file='star_mag_18.00/star18.00.ecsv',
    filter_name='F146',
    sca=1,
    pointing=(80.0, 30.0),
    date='2026-10-15T00:00:00',
    output_directory='./temp',
    output_suffix='_1',
    previous_file='./temp/roman_sim_F146_WFI01_L1_0.asdf',  # for persistence
)
```


### `track_multiple_pixels_all_resultants` (Cell 35)
Records DN values for a list of pixels across all resultants and appends to the per-resultant CSV files.

---

## Photometric Calibration

Magnitudes are computed using the AB zero point method:

```
m_AB = ZERO_POINT_AB - 2.5 × log10(aperture_sum_DN)
```

where `ZERO_POINT_AB = 28.08` for the F146 filter.

Flux in Janskys uses an alternative path through `photfnu`:

```
flux_Jy = (DN / resultant_time) × photfnu
```

where `photfnu = photmjsr × 1e6 × pixel_area = 0.240393 × 1e6 × 2.8083e-13`.

The two paths are used in different parts of the notebook:
- **Aperture photometry** (main pipeline) uses `ZERO_POINT_AB`
- **Ensemble/pixel tracking analysis** uses `photfnu`

---

## Known Issues and Notes

- **Saturation threshold**: The pipeline first attempts to retrieve the per-pixel saturation threshold from CRDS. If CRDS lookup fails, it falls back to 55,000 DN.
- **MA Table 1002**: This is a custom read pattern not in the standard `romanisim` table set. It is injected directly into the metadata before simulation.
- **Processing time**: At ~3.5 minutes per exposure, the full 108-exposure run takes approximately 6 hours. The checkpoint system allows safe interruption and resumption.
