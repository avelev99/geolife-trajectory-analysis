# Trajectory Insights Pipeline — Report

Version: 1.0  
Status: Analytical prototype with validation, robustness, and reporting scaffolds.

## Executive Summary
- Mobility Structure: Cleaned trajectories yield consistent trip distributions; median distances and durations fall in expected urban ranges with plausible speeds.
- Stay-Point Robustness: Stay-point detection remains stable across eps ∈ [100, 200] meters and min_samples ∈ [3, 8], indicating limited sensitivity in core hotspots.
- Trip Segmentation Sensitivity: Larger gap thresholds reduce trip counts primarily by merging short/adjacent trips; medium/long trip statistics remain comparatively stable.
- Cross-User Stability: KS tests across user folds show no major distributional divergence in distance or speed, suggesting results generalize across observed users.
- Temporal Generalization: Quarterly drift is observable in median distance and speed; seasonality or sampling changes may partially explain the variation.
- Decision Implications: Default parameters are adequate for first-pass insights; periodic retraining/revalidation is advised when user mix or seasonality shifts.

Links to generated summaries and figures:
- Sensitivities:
  - CSV: `outputs/reports/sensitivity_staypoints.csv`, `outputs/reports/sensitivity_trips.csv`
  - Figures: `outputs/figures/sensitivity_staypoints.png`, `outputs/figures/sensitivity_trips.png`
- Validation:
  - CSV: `outputs/reports/cross_user_validation.csv`, `outputs/reports/temporal_generalization.csv`, `outputs/reports/temporal_generalization_ks.csv`
  - Figures: `outputs/figures/cross_user_validation.png`, `outputs/figures/temporal_generalization.png`

## Data Understanding & Dictionary
Data sources are raw GPS trajectories transformed into cleaned point-level tables and trip summaries. Time is UTC-normalized where applicable; coordinates in degrees (WGS84).

Key artifacts (cleaned/processed):
- 01_trajectories_cleaned.parquet — point-level records per user, chronologically sorted.
- 02_trips.parquet — per-trip summaries with distance, duration, and speed.
- 03_stay_points.parquet — clustered stay-points (if produced in your run).
- 04_user_features.parquet — per-user features for segmentation.
- 05_frequent_paths.parquet — recurring route summaries.
- 06_trip_modes_heuristic.parquet — heuristic mode inference outputs.
- 07_user_segments.parquet — user clusters from segmentation.

Limitations:
- Sampling variability across users/time, potential GPS noise and gaps.
- Sparse label coverage; mode inference is heuristic/exploratory.
- Trajectories reflect observed behavior; unobserved trips absent.

Concise Data Dictionary (cleaned artifacts)
- Points (01_trajectories_cleaned.parquet)
  - user_id (str): anonymized identifier
  - timestamp (datetime, UTC): observation time
  - lat, lon (float, deg): coordinates
  - alt_m (float, m): altitude (if available/derived)
  - dist_m (float, m): distance to previous point within user
  - dt_s (float, s): time delta to previous point
  - speed_kmh (float, km/h): instantaneous speed
  - trip_seq (int): trip sequence per user
  - trip_id (str): "{user_id}-{trip_seq}"
- Trips (02_trips.parquet)
  - user_id (str)
  - trip_seq (int)
  - trip_id (str)
  - start_time, end_time (datetime, UTC)
  - points (int): count of points in trip
  - distance_m (float, m)
  - duration_s (float, s)
  - avg_speed_kmh (float, km/h)
- Stays (03_stay_points.parquet)
  - user_id (str)
  - stay_id (str)
  - center_lat, center_lon (float, deg)
  - start_time, end_time (datetime, UTC)
  - dwell_minutes (float, min)
  - point_count (int)
- Features (04_user_features.parquet)
  - user_id (str)
  - total_trips (int)
  - median_trip_distance_km (float, km)
  - median_trip_duration_min (float, min)
  - radius_of_gyration_km (float, km)
  - fraction_time_home (float, 0–1)
  - fraction_time_work (float, 0–1)
  - routine_index (float, 0–1)
  - average_daily_distance_km (float, km)
- Additional artifacts follow analogous naming with self-descriptive fields.

## Methods
Pipeline components:
1. Ingestion: Parse raw trajectory files, standardize schema and timestamps.
2. Preprocessing: Compute track-to-track distances (haversine), dt, speed; filter outliers above a configurable speed threshold.
3. Trip Segmentation: Split sequences when time gaps exceed threshold; summarize per-trip distances, durations, and speeds; drop short trips under min distance.
4. Stay-Point Detection: DBSCAN on locally projected XY coordinates; per-cluster dwell time and centers; infer home/work via temporal coverage proxies.
5. Feature Engineering: User-level mobility features (radius of gyration, routine index, trip stats).
6. Routes/Hotspots: Gridded hotspot maps and route frequency summaries.
7. Mode Inference (exploratory): Heuristic signals on speed/stop patterns.
8. Segmentation: KMeans on standardized features; qualitative interpretation of segments.
9. Modeling (exploratory): Basic experiments on mode prediction or segment explainability.

Parameter choices (examples; configurable in [`configs/config.yaml`](./configs/config.yaml)):
- Preprocessing: trip_segmentation_threshold_min=30, min_trip_distance_m=100, max_speed_kmh=200.
- Stay-Point DBSCAN: eps_meters=150, min_samples=5.
- Segmentation: KMeans k=4, random_state=42.

## Results
- EDA: Points concentrated around urban corridors; reasonable daily coverage with gaps.
- Hotspots/Stops: Dense clusters at home/work-like locations; hotspot grid heatmaps align with known centers.
- Temporal Patterns: Weekday peaks in trip counts; weekend longer-distance leisure tails suggested by distributions.
- Segmentation Interpretation: Clusters separate by typical distance/duration and routine entropy; “commuter-like” vs “errands/short-trip” profiles emerge.
- Modeling Performance (Exploratory): Heuristic mode signals show plausible separation but require labels for rigorous evaluation.

## Validation & Robustness
Sensitivity analyses:
- Stay-Points: See `outputs/reports/sensitivity_staypoints.csv` and `outputs/figures/sensitivity_staypoints.png`. Stability indicates parameter tolerance around defaults.
- Trip Segmentation: See `outputs/reports/sensitivity_trips.csv` and `outputs/figures/sensitivity_trips.png`. Trip counts drop as gap threshold grows.

Cross-User Checks:
- See `outputs/reports/cross_user_validation.csv` and `outputs/figures/cross_user_validation.png`. KS statistics suggest distributional stability across folds for key metrics.

Temporal Generalization:
- See `outputs/reports/temporal_generalization.csv` and `outputs/reports/temporal_generalization_ks.csv`, plus `outputs/figures/temporal_generalization.png`. Some seasonal drift observed; monitor and re-tune thresholds if drift grows.

Uncertainty & Biases:
- Sampling bias by device usage and user mix.
- GPS noise and urban canyon effects.
- Heuristic mode inference can misclassify; requires labels/ground truth for calibration.

## Ethics & Privacy
- Trajectory data is sensitive and can reveal home/work, routines, and social patterns.
- Anonymization: Keep `user_id` pseudonymous; avoid re-identification through spatial joins with public datasets.
- Minimization: Retain only necessary features; aggregate where possible (e.g., grid-level counts).
- Access Controls: Store processed artifacts with restricted access; avoid public release of raw points.
- Transparency: Document data use, retention periods, and opt-out mechanisms when applicable.

## What I’d Do Next
- Add formal cross-validation for mode inference with labeled ground truth.
- Introduce advanced stay detection (time-aware clustering, Hidden Markov Models for modes).
- Improve route canonicalization and map-matching for better path consistency.
- Deploy a parameter dashboard to run sensitivity in interactive UI.
- Extend fairness auditing: evaluate distributional stability across demographic or spatial strata (if ethically and legally available).

## Reproducibility
- Configuration: [`configs/config.yaml`](./configs/config.yaml)
- Snapshot: `python src/export_config_snapshot.py` → `outputs/reports/config_snapshot.json`
- Notebook Order:
  1. [`notebooks/1.0-Data-Exploration.ipynb`](./notebooks/1.0-Data-Exploration.ipynb)
  2. [`notebooks/2.0-Stay-Point-Detection.ipynb`](./notebooks/2.0-Stay-Point-Detection.ipynb)
  3. [`notebooks/3.0-User-Segmentation-Analysis.ipynb`](./notebooks/3.0-User-Segmentation-Analysis.ipynb)
  4. [`notebooks/4.0-Mode-Modeling.ipynb`](./notebooks/4.0-Mode-Modeling.ipynb)
  5. [`notebooks/5.0-Validation-and-Robustness.ipynb`](./notebooks/5.0-Validation-and-Robustness.ipynb)

Artifacts reference:
- Sensitivity CSVs: `outputs/reports/sensitivity_staypoints.csv`, `outputs/reports/sensitivity_trips.csv`
- Validation CSVs: `outputs/reports/cross_user_validation.csv`, `outputs/reports/temporal_generalization.csv`, `outputs/reports/temporal_generalization_ks.csv`
- Figures: `outputs/figures/*.png`
