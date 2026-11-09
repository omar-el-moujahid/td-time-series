# Reflections

## Reflection 1: Stage 0, checking the data assumption

I would make one test call to the Open-Meteo Air Quality API with `past_days=90` and count the rows in the response. It takes a few minutes, and it would show that real history is available. An assumption about data is a claim, so it should be tested before it shapes the design.

## Reflection 2: Stage 1, my prediction

I predict that CO2 in Le Mans will be **rising** over the last 90 days. Plants absorb a lot of CO2 in summer, so the level is usually low around August. In autumn, plants absorb less, so the level should go up.

Result after the run: the slope was **[+/-0.000] ppm/day**, so my prediction was **[correct / wrong]**.

## Reflection 3: Stage 2, zero gaps

No, the gap-detection code is still needed, even if my run found **[0]** gaps. Zero gaps only shows that the data was clean this one time. It does not guarantee that it will always be clean, because the API could change or the source could become a physical sensor that drops readings. Without this check, a missing day could stay hidden.

## Reflection 4: Stage 3, rolling mean or exponential smoothing

In my output, the last rolling-mean value was **[000.00]** and the last exponential-smoothing value was **[000.00]**. I would prefer the rolling mean for a slow trend, such as a weekly CO2 report, because all seven days count equally and one unusual day has less effect. I would prefer exponential smoothing when a sudden change matters, such as a pollution alert, because recent values have more weight and it reacts faster.

## Reflection 5: Stage 4, when the API fails

The data files go stale, but the dashboard does not break. If all 4 attempts fail, `fetch_series.py` raises an error, the job stops, and the analysis and commit steps do not run. The files from the last successful run stay in the repo, so the page still works with old data. The risk is that the page shows no warning: only the old "last analyzed" date and the red run in the Actions tab show the problem.

## Reflection 6: Stage 5, how far I trust the projection

I would trust my projection for about **7 days**. My slope is **[+/-0.000] ppm/day**, and the projected values go from **[000.00]** to **[000.00]** ppm. Over one week, the slow seasonal trend probably does not change much. After a few weeks, the straight line becomes less reliable because the CO2 cycle is curved and winter changes the trend.

## Reflection 7: Stage 6, why the pipeline kept working

I changed the location to **[city]** (`LAT, LON` in `fetch_series.py`), and the trend was **[+/-0.000] ppm/day**, compared with **[+/-0.000] ppm/day** for Le Mans. The pipeline kept working because `clean_series()`, `rolling_mean()`, `exponential_smoothing()` and `linear_trend_projection()` only receive dates or numbers. They do not mention CO2, ppm or a place. These names only appear in the fetch code, the CSV column, the labels and the notes.
