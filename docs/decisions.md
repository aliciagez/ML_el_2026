# Decisions log

## 2026-09-30 — M0 skeleton
- Python 3.12 (broadest compatibility for LightGBM, StatsForecast and TimesFM/PyTorch).
- Package name `pricefc`, uv-managed with `uv.lock` committed (D7).
- Local MLflow: SQLite backend (`mlflow.db`) plus local artifacts (`mlruns/`) until the shared server exists (D9).
- Config: YAML in `configs/`, validated with pydantic; the resolved config is hashed (`config_hash` tag).
- Heavy dependencies (LightGBM, StatsForecast, Optuna, Prefect, TimesFM) are added in the milestone that first needs them.

## 2026-09-30 — M1 ingestion (partial: ENTSO-E token pending)
- **Raw storage**: local `data/raw/` for now (D8 still open). Layout and manifests follow spec 5.3;
  snapshot files are written read-only and a snapshot directory is never reused.
- **Invalid snapshots are kept**, marked `validation.passed=false` in the manifest; readers
  (`load_latest`) skip them. The raw record of what the API returned is preserved, and the CLI
  exits non-zero so downstream flows stop.
- **Open-Meteo model pinned to `ecmwf_ifs`**. `best_match` and `*_seamless` switch underlying
  models over time (and `best_match` lacks 100 m wind in Previous Runs), which breaks
  reproducibility. `ecmwf_ifs` is continuous in the Historical Forecast API from before 2021 and
  has Previous Runs from 2025-10-01 (day-1 lead) / 2025-10-02 (day-2 lead), so training
  (stitched) and backtest (true-lead) use the same model. Tag: `open-meteo:<endpoint>:ecmwf_ifs`.
- **Train/eval weather mismatch (intended)**: training uses the stitched historical-forecast
  series (slightly optimistic); backtests must use Previous Runs. The true-lead window
  (2025-10-02 onwards) coincides with the 12-month backtest window.
- **Previous Runs archive gaps**: whole days are missing between about 2026-03-26 and
  2026-05-06, differently per variable and lead. Day-1 and day-2 gaps mostly do not overlap,
  but `temperature_2m` is missing in both for 366 hours (and 100 m wind for ~263 hours).
  Tolerated in validation via `max_null_frac: 0.10` (per-endpoint config) with per-column
  `null_counts` recorded. M2 must handle this explicitly (lead fallback day1 -> day2, then flag
  or exclude affected origins); never silently impute.
- **Leakage note for M2**: `previous_day1` values for early D+1 hours may come from a run whose
  data is only available shortly before the 09:00 origin. Define `available_at` from model run
  time + publication delay; if unclear, use `previous_day2` for strictness.
- **ENTSO-E**: entsoe-py 0.8.1. All series converted to UTC; per-row `resolution` inferred from
  spacing (handles the 2025-10-01 switch to PT15M). Chunks are [start, end) to avoid the
  library's inclusive-end overlap. Tests use a fake client; no recorded ENTSO-E fixtures yet
  (need a token). Hydro reservoirs are pulled per country (`SE`) [VERIFY].
- **Live forecast archive** started 2026-09-30 (as-issued vintages, spec 11).
- **First SE3 weather backfill has no commit SHA**: the 20 snapshots pulled on 2026-09-30 before
  the first commit record `git_sha: unknown, git_dirty: true`. They are kept, but superseded by a
  re-pull made from a committed tree, and dataset builds should use the re-pulled snapshots.
