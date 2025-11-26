# v3_1_4_0 — Skin Surface Tracking 

## Overview
- Purpose: Detect the skin surface in OCT video frames and export surface depth per time sample for each a-line.
- Output CSV: `skin_displacement_estimation.csv` saved in each acquisition folder (`1_process/OCT/v3_1_4_0.py:178-181`).
- Auto brushing detection: Periods of brushing are auto-detected and marked as `NaN` in the output to avoid bias (`1_process/OCT/v3_1_4_0.py:81-95`, `1_process/OCT/v3_1_4_0.py:141-144`).

## Inputs & Outputs
- Input folders: Found automatically where `morph.pkl` exists in the database path (`1_process/OCT/v3_1_4_0.py:18-23`).
- Output file: `skin_displacement_estimation.csv` containing columns `aline_id0`, `aline_id1`, … with surface depth per time sample (`1_process/OCT/v3_1_4_0.py:66-67`, `1_process/OCT/v3_1_4_0.py:178-181`).
- Optional figures: Per a-line PNG showing raw, binary, and filtered images with the detected surface overlay (`1_process/OCT/v3_1_4_0.py:148-176`).

## How It Works
- Load OCT morph data and build dB video (`1_process/OCT/v3_1_4_0.py:71-77`).
- Detect brushing window using signal variance; smooth, threshold, and create a sample mask (`1_process/OCT/v3_1_4_0.py:81-95`, `1_process/OCT/v3_1_4_0.py:124-129`).
- Preprocess per a-line:
  - Crop shallow depths by `depth_offset=15` (`1_process/OCT/v3_1_4_0.py:78-100`).
  - Suppress low intensities with `mean + 0.5×std` (`1_process/OCT/v3_1_4_0.py:101-105`).
  - Normalize to `uint8` and median blur (`1_process/OCT/v3_1_4_0.py:105-107`).
  - Binarize by global mean (`1_process/OCT/v3_1_4_0.py:108`).
  - Morphological close to connect edges (`1_process/OCT/v3_1_4_0.py:109-111`).
  - Connected components; keep components with area ≥ 1% of max (`1_process/OCT/v3_1_4_0.py:113-121`).
- Extract surface depth per column:
  - Take first non-zero pixel as surface; if missing, carry forward previous value; use `depth_offset` for the first column (`1_process/OCT/v3_1_4_0.py:130-140`).
  - Mask values to `NaN` during brushing to exclude disturbed frames (`1_process/OCT/v3_1_4_0.py:141-144`).
- Save results to CSV; optionally save visualization figures (`1_process/OCT/v3_1_4_0.py:148-176`, `1_process/OCT/v3_1_4_0.py:178-181`).

## Key Parameters
- `datatype`: Data group to process; defaults to `OCT_HAIR-DEFLECTION` (`1_process/OCT/v3_1_4_0.py:39`).
- `depth_offset`: Pixels to skip from top of image; default `15` (`1_process/OCT/v3_1_4_0.py:78`).
- `auto_detect_brushing`: Enables brushing detection and masking; default `True` (`1_process/OCT/v3_1_4_0.py:46-49`).
- `variance_threshold`: Sensitivity for brushing detection; higher detects more periods; default `2.0` (`1_process/OCT/v3_1_4_0.py:47-49`).
- `MAX_ACQS`: Limits number of acquisitions processed in a run; default `12` (`1_process/OCT/v3_1_4_0.py:44`).
- `save_figure`: When `True`, saves per a-line diagnostic figures (`1_process/OCT/v3_1_4_0.py:43`, `1_process/OCT/v3_1_4_0.py:148-176`).

## Brushing Detection Methods
- Automatic detection:
  - Enable with `auto_detect_brushing = True` and control sensitivity via `variance_threshold` (higher → more samples flagged).
  - Internally, the script computes variance across a-lines and depths per sample, smooths the trace, thresholds by `median * variance_threshold`, and adds a small buffer around detected intervals. Those samples are masked as `NaN` in the CSV.
- Custom per-acquisition periods:
  - Define fixed intervals in samples using `brushing_periods = { acq_index: (start, end), ... }`.
  - Use when you know the brushing window a priori or want deterministic masking.
  - Note: the dictionary is defined in the script, but the current loop applies auto-detection. To use custom periods, override the detected `brushing_start`/`brushing_end` when a key exists:

```python
custom = brushing_periods.get(acq_id - 1)
if custom is not None:
    brushing_start, brushing_end = custom
```

This will mask exactly the specified sample range for that acquisition.

## Run It
- Open a terminal in the project root and run:
  - `python 1_process/OCT/v3_1_4_0.py`
- The script locates eligible acquisitions, processes each a-line, and writes `skin_displacement_estimation.csv` in the same folder.

## Interpreting the CSV
- Each column `aline_idX` contains the detected skin surface depth (pixels) over time for a-line `X`.
- Samples within detected brushing windows are `NaN` to indicate disturbance and exclude them from downstream analysis.
- Depths include the `depth_offset` applied; they reference pixel indices in the original depth axis.

## Tips
- To process more or fewer acquisitions, adjust `MAX_ACQS` (`1_process/OCT/v3_1_4_0.py:44`).
- If too many samples are masked as brushing, reduce `variance_threshold` (`1_process/OCT/v3_1_4_0.py:47-49`).
- If the surface looks trimmed near the top, lower `depth_offset` (`1_process/OCT/v3_1_4_0.py:78`).
- Use saved figures to quickly validate surface detection quality (`1_process/OCT/v3_1_4_0.py:148-176`).