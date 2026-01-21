# Romanisim-Persistence-Pipeline
This project simulates images using romanisim to better understand the effects of persistence of the F146 filter on the Nancy Grace Roman Space Telescope

### What the Plots Show:
**Left plot (Flux Uncertainty):**
- Baseline ~2×10⁻²⁰ Jy (excellent)
- Spikes to ~6×10⁻²⁰ Jy in 11 files (outliers)
- Slight increasing trend (r²=0.035, not significant)

**Right plot (Magnitude Uncertainty):**
- Baseline ~10⁻³ mag (0.0005 mag)
- Spikes to ~10⁻¹ to 10⁻² in outliers
- Slight decreasing trend (r²=-0.047)


##  Technical Details

### Romancal Pipeline Steps (from your log):
```
 dq_init          - Data quality initialization
 saturation       - Detected 72,894 saturated pixels
 refpix           - Reference pixel correction
 linearity        - Linearity correction
 rampfit          - Slope fitting (Casertano weighting)
 dark_current     - Dark subtraction
 assign_wcs       - WCS assignment
 flatfield        - Flat field correction
 photom           - Photometric calibration
 source_catalog   - SKIPPED (we do our own)
 tweakreg         - SKIPPED (not needed)
```

### Manual Steps:
1. Generate L1 with romanisim
2. Replace saturated pixels in resultants
3. Run romancal pipeline → L2
4. Auto-detect source position
5. Aperture photometry
6. Persistence detection (vs file 1 reference)

---

##  Final Statistics

**Processing Speed:**
- Files per hour: ~15
- Files per day: ~370
- Total time: ~23 days

**Data Quality:**
- Detection: 100%
- SNR: 2,336 ± 40
- Mag precision: 0.0005 mag
- Outliers: 11/113 (9.7%)

**Disk Space:**
- Per file: ~100 MB temp (cleaned up)
- CSV files: Growing steadily
- Segmentation maps: Now skipped (saved 579 GB!)

---


