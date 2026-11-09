# Student manual — TD4: Time series: collection, cleaning, smoothing, and honest projection

**Before you start:** create a file called `REFLECTIONS.md` in your repo now. Every
question in this document (marked **Reflection**) gets a short answer (2-4 sentences
— a real judgment, not a definition) written into that file, under a heading matching
the stage. A consolidated list of every question is at the end of this document, so
you can check nothing was skipped. `REFLECTIONS.md` is a real deliverable, part of
your submission, exactly like the code.

## What you're building

```
td-time-series/
  scripts/
    fetch_series.py                 # real: pulls 90 days of real hourly CO2 from Open-Meteo
    analyze_series.py               # clean, smooth, project
  data/
    series.csv                      # generated + committed by the Action -- real daily CO2 (ppm)
  docs/                             # GitHub Pages source
    index.html
    data/
      series_analysis.json           # generated + committed by the Action
  .github/workflows/
    update_series.yml
  README.md
  REFLECTIONS.md
```

**Everything below happens in a web browser on github.com. No local install
required.**

## Stage 0 — Framing: how far can you honestly project a noisy series forward?

*Connects to CM3 (Modern Data Acquisition) and CM4 (Automated Pipelines).*

The central question of this TD: given a stream of noisy real values over time, how
do you turn it into something you can act on, and how far can you honestly project
it forward?

An earlier version of this exercise used a synthetic (fake, generated) series,
reasoning that real accumulated history wouldn't exist yet this early in the term.
That reasoning turned out to be wrong the moment someone actually checked: Open-Meteo's
Air Quality API returns up to ~90 days of **real** historical data in a single call,
so there was never a real reason to fake the data at all — the acquisition
constraint that justified the synthetic approach didn't actually exist.

> **Reflection 1.** The synthetic-data plan was reasonable-sounding and wrong. Before
> reading further: what would you have had to do to *check* whether real historical
> CO2 data was available, instead of just assuming it wasn't? Name one concrete
> action. (This is CM3's acquisition-planning discipline in miniature: an assumption
> about what data is available is itself a claim that needs checking, not something
> to design around unverified.)

## Stage 1 — Collection: pulling a real environmental variable from a public API

*Connects to CM3 (acquisition patterns, provenance).*

### Step 1 — Create the repository

Public repo (GitHub Pages requires it), e.g. `td-time-series`.

### Step 2 — Allow the workflow to write to the repo

**Settings → Actions → General → Workflow permissions → "Read and write
permissions" → Save.** The Action you add later commits its own output back to the
repo — without this, that commit silently fails.

### The data source

[Open-Meteo's Air Quality API](https://open-meteo.com/en/docs/air-quality-api) is
free, keyless, no registration — same convention as this course's other Open-Meteo
use. It reports `carbon_dioxide` (ppm) hourly, from a reanalysis-plus-forecast model
(CAMS), not a single physical sensor — a distinction that matters later, in Stage 2.

> **Reflection 2.** Before running the script below: the location is fixed to Le Mans
> (48.0061, 0.1996). Based on what you know about seasons and plant growth, do you
> expect atmospheric CO2 there to be *rising*, *falling*, or *flat* over the last 90
> days from today? Write your prediction down before you look at your own output —
> you'll check it in Stage 3.

### Step 3 — Add the fetch script (`scripts/fetch_series.py`)

```python
import csv, os, time
import requests

# Open-Meteo Air Quality API -- free, keyless, no registration, real hourly
# atmospheric CO2 concentration (ppm) from a reanalysis+forecast model, not a
# single physical sensor. past_days gives real history in ONE call (up to at
# least 90 days, confirmed by real testing) -- no need to wait weeks for a
# scheduled pipeline to accumulate data, unlike this course's other real-time-
# history projects (TD2, TP2).
LAT, LON = 48.0061, 0.1996  # Le Mans
PAST_DAYS = 90
URL = "https://air-quality-api.open-meteo.com/v1/air-quality"

def fetch_hourly_co2(retries=4, backoff=5):
    params = {
        "latitude": LAT, "longitude": LON,
        "hourly": "carbon_dioxide",
        "past_days": PAST_DAYS,
        "forecast_days": 0,
        "timezone": "auto",
    }
    last_error = None
    for attempt in range(1, retries + 1):
        try:
            resp = requests.get(URL, params=params, timeout=(15, 30))
            resp.raise_for_status()
            return resp.json()
        except requests.exceptions.RequestException as e:
            last_error = e
            print(f"  fetch attempt {attempt}/{retries} failed: {e}")
            if attempt < retries:
                time.sleep(backoff * attempt)
    raise RuntimeError(f"Open-Meteo Air Quality API unreachable after {retries} attempts") from last_error

def aggregate_to_daily(times, values):
    # Real hourly readings -> one row per calendar day (mean of that day's
    # available hours). A day found with fewer than 24 real (non-null) hours
    # is flagged, not silently averaged over whatever happened to be present --
    # same "say what you did" discipline as the rest of this course's pipelines.
    by_day = {}
    for t, v in zip(times, values):
        day = t[:10]  # "YYYY-MM-DDTHH:MM" -> "YYYY-MM-DD"
        by_day.setdefault(day, []).append(v)

    rows = []
    for day in sorted(by_day):
        hours = by_day[day]
        real_hours = [h for h in hours if h is not None]
        rows.append({
            "date": day,
            "co2_ppm": round(sum(real_hours) / len(real_hours), 1) if real_hours else "",
            "n_hours_observed": len(real_hours),
        })
    return rows

def main():
    print(f"Fetching {PAST_DAYS} days of real hourly CO2 (ppm) for Le Mans "
          f"({LAT}, {LON}) from Open-Meteo Air Quality API...")
    data = fetch_hourly_co2()
    times = data["hourly"]["time"]
    values = data["hourly"]["carbon_dioxide"]
    print(f"Received {len(times)} real hourly readings "
          f"({times[0]} to {times[-1]})")

    rows = aggregate_to_daily(times, values)
    incomplete = [r for r in rows if r["n_hours_observed"] < 24]
    print(f"Aggregated to {len(rows)} daily values -- "
          f"{len(incomplete)} day(s) with fewer than 24 real hourly readings")
    for r in incomplete:
        print(f"  incomplete day: {r['date']} ({r['n_hours_observed']}/24 hours)")

    os.makedirs("data", exist_ok=True)
    with open("data/series.csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=["date", "co2_ppm", "n_hours_observed"])
        w.writeheader()
        w.writerows(rows)
    print(f"Wrote data/series.csv ({len(rows)} days)")

if __name__ == "__main__":
    main()
```

**Checkpoint:** you don't run this script standalone yet — it runs as part of the
Action in Stage 4. For now, just have the file committed.

**A note on the numbers in this document.** Nowhere below do we quote exact CO2
values, an exact trend slope, or an exact date range. That's deliberate, not an
omission: this script hits a **live** API, so your 90-day window is *today's* 90
days, not the same window a previous run saw — your real numbers will be genuinely
different from anyone else's, including your instructor's. Expect roughly 90 rows,
CO2 readings somewhere in the low-to-mid 400s ppm, and a trend slope of a few tenths
of a ppm per day, in either direction. A different number than a classmate's is not a
sign you did something wrong.

## Stage 2 — Cleaning: what does "clean" even mean for a time series?

*Connects to CM3 (provenance, data quality) and CM4 (what breaks when nobody is
watching).*

The next script does three things: loads what was fetched, **reindexes it to every
calendar day** and flags any that are missing (rather than silently pretending they
aren't), then smooths and projects. Read `clean_series()` before running anything.

> **Reflection 3.** Open-Meteo's air-quality numbers come from a reanalysis *model*,
> not a single physical sensor — meaning gaps are unlikely (the model backfills every
> hour). A raw sensor feed (say, a physical CO2 monitor bolted to a building) would
> behave completely differently. If your own run reports zero gaps, does that mean
> the gap-detection code in `clean_series()` is unnecessary and could be deleted? Argue
> your answer using the distinction between "this data source happened to be clean
> this one time" and "this data source is guaranteed to always be clean."

### Step 4 — Add the analysis script (`scripts/analyze_series.py`)

```python
import csv, json, time
from datetime import date, timedelta

WINDOW = 7
PROJECT_STEPS = 7

def load_series(path="data/series.csv"):
    with open(path) as f:
        rows = []
        for r in csv.DictReader(f):
            if r["co2_ppm"] == "":
                continue  # a day with zero real hourly readings -- nothing to average
            rows.append({"date": r["date"], "value": float(r["co2_ppm"])})
    return rows

def clean_series(rows):
    # Reindex to every calendar day between the first and last real date,
    # marking gaps explicitly rather than silently interpolating -- a real
    # pipeline should say what it did, not hide that a day was missing.
    by_date = {r["date"]: r["value"] for r in rows}
    start = date.fromisoformat(rows[0]["date"])
    end = date.fromisoformat(rows[-1]["date"])
    n_days = (end - start).days + 1

    cleaned = []
    gaps = []
    for i in range(n_days):
        d = (start + timedelta(days=i)).isoformat()
        if d in by_date:
            cleaned.append({"day": i, "date": d, "value": by_date[d]})
        else:
            gaps.append(i)

    gap_ranges = []
    if gaps:
        seg_start = prev = gaps[0]
        for g in gaps[1:]:
            if g == prev + 1:
                prev = g
            else:
                gap_ranges.append((seg_start, prev))
                seg_start = prev = g
        gap_ranges.append((seg_start, prev))
    return cleaned, gap_ranges, start

def rolling_mean(values, window=WINDOW):
    out = []
    for i in range(len(values)):
        lo = max(0, i - window + 1)
        chunk = values[lo:i + 1]
        out.append(round(sum(chunk) / len(chunk), 2))
    return out

def exponential_smoothing(values, alpha=0.3):
    out = [values[0]]
    for v in values[1:]:
        out.append(round(alpha * v + (1 - alpha) * out[-1], 2))
    return out

def linear_trend_projection(days, values, start, steps=PROJECT_STEPS):
    # Simple least-squares line over the observed (day, value) pairs --
    # deliberately the simplest honest baseline, not a sophisticated model.
    n = len(days)
    mean_x = sum(days) / n
    mean_y = sum(values) / n
    num = sum((x - mean_x) * (y - mean_y) for x, y in zip(days, values))
    den = sum((x - mean_x) ** 2 for x in days)
    slope = num / den if den else 0.0
    intercept = mean_y - slope * mean_x
    last_day = days[-1]
    projection = []
    for i in range(1, steps + 1):
        day_idx = last_day + i
        projection.append({
            "day": day_idx,
            "date": (start + timedelta(days=day_idx)).isoformat(),
            "value": round(intercept + slope * day_idx, 2),
        })
    return projection, slope

def main():
    raw = load_series()
    cleaned, gap_ranges, start = clean_series(raw)

    print(f"Loaded {len(raw)} real daily CO2 readings "
          f"({raw[0]['date']} to {raw[-1]['date']}).")
    if gap_ranges:
        for seg_start, seg_end in gap_ranges:
            span = seg_end - seg_start + 1
            d1 = (start + timedelta(days=seg_start)).isoformat()
            d2 = (start + timedelta(days=seg_end)).isoformat()
            print(f"  Gap detected: {d1} to {d2} ({span} missing day(s)) -- "
                  f"flagged, not silently filled.")
    else:
        print("  No gaps detected -- this real pull is a complete daily series "
              "(Open-Meteo's air-quality model backfills every hour, unlike raw "
              "sensor telemetry, which can and does drop readings -- the "
              "gap-detection logic stays in for that reason, see the manual).")

    days = [c["day"] for c in cleaned]
    values = [c["value"] for c in cleaned]

    smoothed_ma = rolling_mean(values)
    smoothed_exp = exponential_smoothing(values)
    projection, slope = linear_trend_projection(days, values, start)

    print(f"\nLast 5 raw values (ppm):      {values[-5:]}")
    print(f"Last 5 rolling-mean (ppm):    {smoothed_ma[-5:]}")
    print(f"Last 5 exp-smoothed (ppm):    {smoothed_exp[-5:]}")
    print(f"\nLinear trend slope: {slope:+.3f} ppm/day")
    print(f"Projection (next {PROJECT_STEPS} days, ppm): {[p['value'] for p in projection]}")

    honest_note = (
        f"Projection covers {PROJECT_STEPS} days ahead from {len(cleaned)} days of "
        f"real CO2 history ({len(gap_ranges)} known gap(s) in the source data). "
        "This is a straight line through one seasonal window of real atmospheric "
        "data -- CO2 has a well-known yearly cycle (lower in the Northern "
        "Hemisphere's growing season, higher in winter), so a trend measured over "
        "90 summer/autumn days should NOT be extrapolated across season boundaries. "
        "Trust this projection only a few days out."
    )
    print(f"\nHonesty note: {honest_note}")

    output = {
        "generated_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "unit": "ppm",
        "raw": [{"day": c["day"], "date": c["date"], "value": c["value"]} for c in cleaned],
        "gaps": [
            {"start_day": s, "end_day": e,
             "start_date": (start + timedelta(days=s)).isoformat(),
             "end_date": (start + timedelta(days=e)).isoformat()}
            for s, e in gap_ranges
        ],
        "rolling_mean": [{"day": d, "value": v} for d, v in zip(days, smoothed_ma)],
        "exp_smoothed": [{"day": d, "value": v} for d, v in zip(days, smoothed_exp)],
        "projection": projection,
        "trend_slope_per_day": round(slope, 4),
        "honest_note": honest_note,
    }
    with open("docs/data/series_analysis.json", "w") as f:
        json.dump(output, f, indent=2)
    print("\nWrote docs/data/series_analysis.json")

if __name__ == "__main__":
    main()
```

You'll run this together with the fetch script via the Action in Stage 4 — read on
before running anything, since Stage 3 covers what the rest of this same script does.

## Stage 3 — Modeling: rolling mean, exponential smoothing, and a projection

*Connects to CM2 (KPI Design and Multi-Objective Decision Making) — a model's output
is a construct built from choices, not a neutral fact.*

The same script above computes three different views of the cleaned series:

- **Rolling mean** — the plain average of the last `WINDOW` (7) days.
- **Exponential smoothing** — a weighted average where recent points count more,
  controlled by `alpha` (0.3 here — higher means more weight on the very latest
  value, lower means smoother but slower to react).
- **Linear trend projection** — the simplest possible honest forecast: a straight
  line through the observed points, extended `PROJECT_STEPS` (7) days forward.

> **Reflection 4.** Look at your own script output: the last 5 rolling-mean values
> versus the last 5 exponentially-smoothed values. They're close but not identical.
> In your own words, describe one real situation where you'd prefer the rolling mean
> over exponential smoothing, and one where you'd prefer the reverse — base this on
> how each one reacts to a sudden recent change versus a slow trend, not on the
> formulas themselves.

## Stage 4 — Storage: committing a rolling window, on a schedule

*Connects to CM3 (Git for data) and CM4 (scheduled execution vs. one-off).*

### Step 5 — Add a placeholder (`docs/data/series_analysis.json`)

**Add file → Create new file** → `docs/data/series_analysis.json` → paste:

```json
{
  "generated_at": "not yet generated",
  "unit": "ppm",
  "raw": [],
  "gaps": [],
  "rolling_mean": [],
  "exp_smoothed": [],
  "projection": [],
  "trend_slope_per_day": 0,
  "honest_note": ""
}
```

### Step 6 — Add the workflow (`.github/workflows/update_series.yml`)

```yaml
name: Update analysis

"on":
  workflow_dispatch: {}
  schedule:
    - cron: "0 6 * * 1"   # weekly, Monday -- fresh 90-day rolling window each run

permissions:
  contents: write

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - run: pip install requests
      - run: python scripts/fetch_series.py
      - run: python scripts/analyze_series.py
      - name: Commit updated data
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/series.csv docs/data/series_analysis.json
          git commit -m "Update CO2 series [automated]" || echo "Nothing to commit"
          git push
```

Weekly, not one-off: unlike a static file, `fetch_series.py` pulls a fresh real
90-day window every time it runs, so a schedule keeps the data genuinely current
instead of freezing it at whatever the first run happened to capture.

> **Reflection 5.** This pipeline has exactly one step that can fail *unattended*, at
> 6am on a Monday with nobody watching: the API call. `fetch_hourly_co2`'s retry loop
> exists for that reason. Suppose the retry loop exhausts all 4 attempts and the whole
> job fails. What actually happens to `data/series.csv` and `docs/data/series_analysis.json`
> — do they go stale with last week's numbers, or does the dashboard break outright?
> Look at the workflow file and the scripts to answer, don't guess.

### Step 7 — Turn on GitHub Pages

**Settings → Pages → Source: "Deploy from a branch" → Branch `main`, folder `/docs`
→ Save.**

### Step 8 — Run the pipeline

**Actions tab → "Update analysis" → Run workflow → Run workflow.** Confirm a green
checkmark, then open `data/series.csv` and `docs/data/series_analysis.json` in your
repo — both should contain real numbers with a fresh `generated_at` timestamp, not
placeholders.

## Stage 5 — Visualization: building the honesty into the dashboard, not just the console

*Connects to CM2 (multi-objective/honest KPIs) and CM6 (Responsible and Sustainable
BI) — a dashboard that shows a projection without also showing its limits is making a
claim it can't back up.*

### Step 9 — Add the dashboard (`docs/index.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>CO2 time series lab — cleaning, smoothing, projection</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
<style>
  body { font-family: system-ui, sans-serif; max-width: 900px; margin: 2rem auto; padding: 0 1rem; }
  .kpis { display: flex; gap: 1rem; flex-wrap: wrap; margin: 1rem 0; }
  .kpi { flex: 1; min-width: 160px; padding: 0.8rem 1rem; border: 1px solid #ccc; border-radius: 8px; }
  .kpi .value { font-size: 1.5rem; font-weight: bold; color: #0d7c9e; }
  .kpi .label { font-size: 0.8rem; color: #666; }
  #honesty { padding: 0.8rem 1rem; border: 2px solid #c46e0a; border-radius: 8px; background: #faf3e8; margin: 1rem 0; }
  canvas { margin-top: 1.5rem; }
  #updated { color: #666; font-size: 0.85rem; }
</style>
</head>
<body>
  <h1>CO2 time series lab</h1>
  <p>Real atmospheric CO2 (ppm) for Le Mans, from Open-Meteo's Air Quality API — raw values, a rolling mean, exponential smoothing, and a linear-trend projection, all computed by <code>scripts/analyze_series.py</code> from <code>data/series.csv</code>.</p>
  <div class="kpis" id="kpis">Loading...</div>
  <div id="honesty"></div>
  <canvas id="seriesChart" height="320"></canvas>
  <p id="updated"></p>

<script>
fetch('data/series_analysis.json')
  .then(r => r.json())
  .then(data => {
    const raw = data.raw || [];
    if (raw.length === 0) {
      document.getElementById('kpis').innerHTML = '<p>No analysis yet -- run "Update analysis" from the Actions tab.</p>';
      return;
    }
    const rollingMean = data.rolling_mean || [];
    const expSmoothed = data.exp_smoothed || [];
    const projection = data.projection || [];
    const gaps = data.gaps || [];
    const unit = data.unit || '';

    document.getElementById('kpis').innerHTML = `
      <div class="kpi"><div class="value">${raw.length}</div><div class="label">Observed days</div></div>
      <div class="kpi"><div class="value">${gaps.length}</div><div class="label">Detected gap(s)</div></div>
      <div class="kpi"><div class="value">${data.trend_slope_per_day >= 0 ? '+' : ''}${data.trend_slope_per_day} ${unit}</div><div class="label">Trend per day</div></div>
      <div class="kpi"><div class="value">${raw[raw.length - 1].value} ${unit}</div><div class="label">Most recent reading</div></div>
    `;

    document.getElementById('honesty').innerHTML = `<strong>Honesty note:</strong> ${data.honest_note}`;

    // One shared x-axis (real calendar dates) covering every day that appears in
    // ANY series (raw, rolling mean, or projection) -- each dataset is null
    // outside its own range so Chart.js draws each line only where it actually
    // has data, rather than a misleading flat line at zero.
    const byDay = {};
    raw.forEach(r => { byDay[r.day] = r.date; });
    projection.forEach(p => { byDay[p.day] = p.date; });
    const allDayIndices = raw.map(r => r.day).concat(projection.map(p => p.day));
    const allLabels = allDayIndices.map(d => byDay[d]);

    function seriesFor(points) {
      const byDayValue = Object.fromEntries(points.map(p => [p.day, p.value]));
      return allDayIndices.map(d => (d in byDayValue ? byDayValue[d] : null));
    }

    const datasets = [
      { label: `Observed (${unit})`, data: seriesFor(raw), borderColor: '#0d7c9e', backgroundColor: 'transparent', pointRadius: 2, borderWidth: 2 },
      { label: 'Rolling mean (7-day)', data: seriesFor(rollingMean), borderColor: '#2e8b46', backgroundColor: 'transparent', pointRadius: 0, borderWidth: 2 },
      { label: 'Exponential smoothing', data: seriesFor(expSmoothed), borderColor: '#8b2ea0', backgroundColor: 'transparent', pointRadius: 0, borderWidth: 1, borderDash: [2, 2] },
      { label: 'Projection (low confidence)', data: seriesFor(projection), borderColor: '#c46e0a', backgroundColor: 'transparent', pointRadius: 3, borderWidth: 2, borderDash: [6, 4] },
    ];

    new Chart(document.getElementById('seriesChart'), {
      type: 'line',
      data: { labels: allLabels, datasets },
      options: {
        plugins: {
          title: { display: true, text: `Real atmospheric CO2 (${unit}), Le Mans — raw → cleaned → smoothed → projected (dashed = beyond observed data)` },
          tooltip: { mode: 'index', intersect: false },
        },
        scales: {
          x: { title: { display: true, text: 'Date' }, ticks: { maxTicksLimit: 12 } },
          y: { title: { display: true, text: `CO2 (${unit})` } },
        },
        interaction: { mode: 'nearest', axis: 'x', intersect: false },
      },
    });

    document.getElementById('updated').textContent =
      `Data last analyzed ${data.generated_at || '(no timestamp -- see script output)'}` +
      (gaps.length ? ` -- gap(s) detected: ${gaps.map(g => `${g.start_date} to ${g.end_date}`).join(', ')}` : ' -- no gaps in this pull.');
  })
  .catch(err => {
    document.getElementById('kpis').innerHTML = '<p>Could not load analysis (' + err + ').</p>';
  });
</script>
</body>
</html>
```

### Step 10 — Open the live dashboard

Visit your Pages URL. You should see four KPI cards, the honesty note, and a line
chart with real calendar dates on the x-axis and four series: solid observed points,
a smoothed rolling-mean line, a dotted exponential-smoothing line, and a dashed
low-confidence projection extending past the real data.

Look specifically at the dashed projection line and the honesty note next to it.

> **Reflection 6.** The honesty note says CO2 follows a yearly cycle and warns
> against extrapolating a 90-day trend across a season boundary. Based on your own
> trend slope and your own 7-day projection values: how many days into the future
> would *you* personally still trust this specific projection before you'd stop
> believing the straight line, and why that number and not a much bigger or smaller
> one? There's no single correct answer — the point is that you can defend the number
> you pick using what you know about the underlying process, not just eyeball it.

## Stage 6 — On your own: a different location, or a different variable

*No new CM concept here — this is you operating the pipeline yourself, not following
a recipe.*

Two independent extensions — do either or both:

1. **Change `LAT, LON`** in `fetch_series.py` to a different real place (your
   hometown, a place you're curious about) and compare: is the trend direction and
   magnitude similar to Le Mans', or different? A genuinely different climate/season
   timing could show a different pattern.
2. **Change the `hourly` parameter** to a different Open-Meteo Air Quality variable
   (check [the docs](https://open-meteo.com/en/docs/air-quality-api) for the current
   list — `pm2_5`, `pm10`, `nitrogen_dioxide`, `ozone`, and others are available the
   same way `carbon_dioxide` is). Note that `aggregate_to_daily` and the rest of the
   pipeline don't care which variable they're fed — only the field name and unit need
   updating, the cleaning/smoothing/projection logic is unchanged.

> **Reflection 7.** You changed one line (a location or a variable) and the entire
> pipeline — cleaning, smoothing, projection, dashboard — kept working with no other
> changes needed. Why was that possible? Point to the specific reason in
> `analyze_series.py`'s own functions (what do they NOT mention, that let them stay
> generic?).

**What "done" looks like:**

- [ ] A working dashboard for your chosen location and/or variable, real data, real
      trend.
- [ ] Reflection 7 answered in `REFLECTIONS.md`.

## Reflection checklist — what `REFLECTIONS.md` should contain

1. What concrete action would have caught the wrong "no real data available"
   assumption, before it shaped a design decision? (Stage 0)
2. Your prediction (rising/falling/flat) for Le Mans CO2, made before running the
   fetch script. (Stage 1)
3. Does zero detected gaps in your own run mean the gap-detection code is
   unnecessary? Argue it. (Stage 2)
4. One situation favoring rolling mean, one favoring exponential smoothing, based on
   your own output. (Stage 3)
5. What actually happens to your data files if the scheduled Action's API call fails
   every retry? (Stage 4)
6. How many days into the future would you personally trust your own projection, and
   why that number? (Stage 5)
7. Why did the pipeline keep working after you changed the location/variable with no
   other code changes? (Stage 6)

## Final delivery checklist

- [ ] Repository public, workflow permissions read/write.
- [ ] All files present: `scripts/fetch_series.py`, `scripts/analyze_series.py`,
      `data/series.csv`, `docs/data/series_analysis.json`, `docs/index.html`,
      `.github/workflows/update_series.yml`, `REFLECTIONS.md`.
- [ ] At least one green run visible in the Actions tab.
- [ ] GitHub Pages live, dashboard shows the four-series chart, real dates, and the
      honesty note.
- [ ] Stage 6 extension attempted (different location or variable).
- [ ] All 7 reflections answered in `REFLECTIONS.md`.

Submit: the repository URL, the live Pages URL, and `REFLECTIONS.md`.
