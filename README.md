# CO2 time series

**[Open the dashboard](https://omar-el-moujahid.github.io/td-time-series/)**

A small automated pipeline that collects 90 days of atmospheric CO2 data, checks for missing days, smooths the series and makes a short projection.

## Pipeline

1. `scripts/fetch_series.py` gets hourly CO2 data from the Open-Meteo Air Quality API and saves daily averages in `data/series.csv`.
2. `scripts/analyze_series.py` flags missing days, computes a 7-day rolling mean and exponential smoothing, and projects 7 days ahead with a straight line.
3. `.github/workflows/update_series.yml` runs both scripts every Monday and commits the results.
4. `docs/index.html` shows the results on GitHub Pages.

## Run the pipeline

Open the **Actions** tab, choose **Update analysis**, then click **Run workflow**.

## Limits

The data comes from a model, not from a physical sensor. CO2 also follows a yearly cycle, so the straight-line projection is only useful for a few days.

## Reflections

See [REFLECTIONS.md](REFLECTIONS.md).
