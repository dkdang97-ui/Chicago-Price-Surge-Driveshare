# Chicago Rideshare 2025: Demand and Surge Pricing Analysis

An exploratory analysis of 2025 Chicago rideshare trips (Transportation Network Providers data), plus a statistical test of one question:

> **When local rideshare demand rises above its normal level, does the fare of an otherwise comparable trip also rise above its expected level?**

The project has two Jupyter notebooks:

| Notebook | Purpose | Run order |
|---|---|---|
| `Chicago_Rideshare_Surge_Analysis.ipynb` | Cleans the data, explores demand, and tests whether local demand raises fares | 1st (it saves the cleaned data) |
| `Chicago_Rideshare_Fare_Prediction.ipynb` | Predicts a trip's fare **before it starts** from a pickup location, a destination and a date and time | 2nd |

See [Fare prediction notebook](#fare-prediction-notebook) for the second one.

---

## Contents

- [Data](#data)
- [Folder layout](#folder-layout)
- [Setup](#setup)
- [How to run](#how-to-run)
- [What the notebook does](#what-the-notebook-does)
- [Method in plain language](#method-in-plain-language)
- [Settings you can change](#settings-you-can-change)
- [Outputs](#outputs)
- [Limitations](#limitations)
- [Testing](#testing)
- [References](#references)

---

## Data

| Input | Description | Source |
|---|---|---|
| `rideshare_2025_01.parquet` … `rideshare_2025_12.parquet` | One file per month of 2025 rideshare trips (about 55 million trips in total) | City of Chicago, *Transportation Network Providers – Trips (2025-)* [1] |
| `weather_cleaned.csv` | Daily precipitation (`PRCP (Inches)`) and snowfall (`SNOW (Inches)`), 2006 onward | Prepared separately |
| `chicago_community_areas.geojson` | Boundaries of Chicago's 77 community areas (downloaded automatically on first run, then cached) | City of Chicago Data Portal |

**Important properties of the trip data [1]:**
- Fares are rounded to the nearest **\$2.50**, tips to the nearest \$1.00.
- Trip start and end times are rounded to the nearest **15 minutes**.
- Census tracts are suppressed in some cases. A blank location can mean outside Chicago *or* privacy suppression.
- The platform (Uber, Lyft, etc.) is **not** identified.

---

## Folder layout

The notebook expects this structure. Change `BASE_DIR` in Section 1, or set the environment variable `RIDESHARE_BASE_DIR`, if yours is different.

```
<BASE_DIR>/
├── Code/
│   └── Chicago_Rideshare_Surge_Analysis.ipynb
├── Data/
│   ├── driveshare_data/
│   │   ├── rideshare_2025_01.parquet
│   │   ├── ...
│   │   └── rideshare_2025_12.parquet
│   ├── weather_cleaned.csv
│   ├── chicago_community_areas.geojson      # created on first run
│   └── rideshare_2025_clean.parquet         # created by the notebook
└── Output/
    └── rideshare_pickup_map.html            # created by the notebook
```

Default `BASE_DIR`: `/Users/williamdang/Documents/Project`

---

## Setup

Python 3.10+ recommended.

```bash
pip install pandas pyarrow numpy matplotlib seaborn folium branca requests pyfixest
pip install duckdb        # optional: only used for the duplicate-trip check
```

With Anaconda you can use `conda install` for most packages. `pyfixest` is installed with pip.

**Memory:** the cleaned table for 55M trips uses roughly 5 GB of RAM (columns are stored in compact types). 16 GB of RAM is comfortable.

---

## How to run

1. Put the data files in the folders shown above.
2. Open the notebook in Jupyter.
3. **Kernel → Restart & Run All.**

The notebook runs top to bottom with no manual steps. Every result in the conclusion is generated from the data when it runs.

---

## What the notebook does

| Section | What happens |
|---|---|
| **0. Guide** | Overview, plus a table of the key benchmarks and what each one is for |
| **1. Setup** | Imports, paths and every setting in one place |
| **2. Load** | Finds the 12 monthly files; loads and fills weather; combines months using compact data types; optional duplicate check |
| **3. Missing data** | Missing-value report; removes trips without a fare; flags trips with missing location |
| **4. Time and quality** | Date, hour, month, weekday, trip minutes; time-of-day categories; data-quality flags (nothing removed) |
| **5. Weather** | Attaches daily weather to each trip |
| **6. Summary tables** | Small daily tables that all charts use; saves the cleaned data to Parquet |
| **7. Exploratory analysis** | Demand by time of day, month, weekday and community area (interactive map) |
| **8. Modeling** | Local demand index, rate card, control vs. surge models, demand buckets, time-of-day comparison, sensitivity check, **weather vs. demand and price** |
| **9. Conclusion** | Auto-generated findings: demand patterns, overall surge result, **demand by pickup time category vs. price surge**, **how weather affects demand and price**, limitations |
| **10. References** | Sources cited in the notebook |

**Time-of-day categories**

| Category | Hours |
|---|---|
| Morning | 6:00 AM – 9:59 AM |
| Noon | 10:00 AM – 3:59 PM |
| Evening | 4:00 PM – 8:59 PM |
| Night | 9:00 PM – 5:59 AM |

---

## Method in plain language

1. **Local demand.** Pickups are counted for every **community area × hour × day** of 2025 (77 × 24 × 365 slots), including hours with zero trips.
2. **Normal demand.** For each slot, "normal" is the average for the same area, same weekday and same hour over the **±4 surrounding weeks**, never including the day itself.
3. **Demand index.** `actual ÷ normal`. 1.0 is normal and 1.3 is 30% busier than normal. The model uses it in steps of +10%.
4. **Noise filter.** Area-hours with fewer than **12 normal pickups** are excluded. Below that, chance alone moves the count by more than about 30% (1/√n), so the ratio is mostly noise. Busy downtown hours are far above this floor and unaffected by it.
5. **Rate card.** Fares follow a recipe: **base + rate × minutes + rate × miles**. The recipe is estimated from normal-demand trips only, then checked against published Chicago UberX rates [3].
6. **Fare premium.** Each trip's fare is compared with its rate-card fare, as a %. Surge pricing multiplies the price, so % is the natural unit. A dollar version is reported as a cross-check.
7. **Models.**
   - *Control:* fare premium ~ minutes + miles + rain + snow, plus month, weekday, hour and area adjustments.
   - *Surge:* the same model **+ demand**. Demand is the only difference, so any improvement is due to demand alone.
   - *Shape:* the same model with demand buckets (<0.8, 0.8–0.9, **0.9–1.1 normal**, 1.1–1.2, 1.2–1.5, >1.5).
   - Error bars are clustered by date and pickup area [6], estimated with `pyfixest` [7].
8. **Time-of-day comparison.** For Morning, Noon, Evening and Night, the notebook reports:
   - Trips, trips per hour, and how often hours are busier than normal.
   - The fare response per +10% demand, at a busy moment and in very busy hours.
   - A formal test of whether each category's response differs from the busiest category's.
9. **Weather.** Each day is grouped as dry, light rain, heavy rain or snow:
   - Dry is under 0.01 inch, the National Weather Service threshold for measurable precipitation.
   - Heavy rain is 0.5 inch or more.
   - Snow is 1 inch or more.

   For each group the notebook measures:
   - **Demand:** daily trips compared with dry days of the same weekday and month.
   - **Price:** the fare premium over the rate card, both overall and holding local demand fixed. The difference shows roughly how much of the weather premium comes through extra riders.
10. **Sensitivity check.** The main model is re-run with the minimum at 5, 12, 20 and 50 pickups and the cap at the 95th, 99th and 100th percentile. If the answer holds, the settings don't drive it.

**Why compare with "normal" at all?** A raw count can't tell you whether an hour is busy. 300 pickups is an ordinary Friday evening downtown but an extraordinary one in a quiet neighborhood. Comparing each area-hour with its own usual level (same place, same weekday and hour, nearby weeks, never including the day itself) makes "busy" mean the same thing everywhere in the city. The rate card plays the same role for price: each fare is compared with what that exact trip normally costs, so long and short rides can be judged on equal terms.

Each notebook section starts with a note explaining what its code does and what any benchmark in it is for.

---

## Settings you can change

All settings are in Section 1 of the notebook.

| Setting | Default | Meaning |
|---|---|---|
| `BASE_DIR` | `/Users/williamdang/Documents/Project` | Project folder (or set `RIDESHARE_BASE_DIR`) |
| `YEAR` | `2025` | Analysis year |
| `SAVE_CLEAN_DATA` | `True` | Save the cleaned table to Parquet |
| `BENCHMARK_WEEKS` | `4` | Weeks on each side used for "normal" demand |
| `MAX_RELATIVE_NOISE` | `0.30` | Max chance variation allowed; sets `MIN_EXPECTED` (0.30 → 12 pickups, 0.22 → 21, 0.15 → 45) |
| `DEMAND_INDEX_CAP_PCTL` | `99` | Cap extreme demand ratios at this percentile (100 = no cap) |
| `DEMAND_BIN_EDGES` | `0, 0.8, 0.9, 1.1, 1.2, 1.5` | Demand bucket boundaries |
| `HIGH_DEMAND_PCTL` | `90` | Defines a "busy moment" for reporting |
| `MEANINGFUL_PCT` | `1.0` | Fare change (%) at a busy moment labeled "economically meaningful" (only changes wording) |
| `BUSY_INDEX` / `VERY_BUSY_INDEX` | `1.2` / `1.5` | Demand index counted as a "busy" / "very busy" hour (same as the bucket edges) |
| `SENSITIVITY_MIN_EXPECTED` | `[5, 12, 20, 50]` | Minimums tested in the sensitivity check (≈45%, 30%, 22%, 14% chance noise) |
| `SENSITIVITY_CAP_PCTL` | `[95, 99, 100]` | Caps tested in the sensitivity check (top 5% trimmed, top 1%, no cap) |
| `SAMPLE_FRACTION` | `0.10` | Share of trips in the regression sample (~5.5M trips) |
| `RANDOM_SEED` | `42` | Fixed seed so the same sample is drawn every run |
| `WEATHER_WINDOW_DAYS` | `2` | Weather blanks filled with the average of this many days before and after |
| `QUALITY_MAX_MILES` / `QUALITY_MAX_MINUTES` | `100` / `360` | Limits for flagging suspicious trips (nothing is removed) |
| `CHUNK_ROWS` | `5,000,000` | Rows processed at a time when counting pickups (memory only) |
| `MAP_CENTER` / `MAP_ZOOM` / `MAP_COLOR_BINS` | Loop / `10` / `7` | Map display only |

Every number in the settings cell has a comment explaining where it comes from.

---

## Outputs

| Output | Where |
|---|---|
| Cleaned trip table | `Data/rideshare_2025_clean.parquet` |
| Interactive pickup map | `Output/rideshare_pickup_map.html` (opens in any browser) |
| Charts, tables and conclusion | Inside the notebook |

---

## Limitations

- **Completed trips ≠ total demand.** Searches, cancellations and riders put off by price are not observed, and prices and volume affect each other.
- **Driver supply is not observed.** Surge pricing responds to both riders and drivers.
- **Platforms are pooled.** Uber, Lyft and others have different rate cards.
- **Rounding.** Fares are rounded to \$2.50 and times to 15 minutes [1]. Single trips are imprecise, though averages are reliable.
- **Short surges are averaged out.** Surges often last only minutes; in one study about 40% lasted 5 minutes [2]. 1-hour slots measure the average premium during busy hours, not the peak multiplier.
- **Fees.** City surcharges (ground transportation, downtown congestion zone, airport and venue fees [4][5]) are probably in `additional_charges`, not `fare`.
- **Holidays** are compared with ordinary days in the "normal" benchmark.
- **Weather gaps.** Blank weather values are filled with the average of the 2 days before and after. Snow blanks with no measured day nearby (long blank runs, typical of station files that record snow only when it falls) are set to 0.

The results are evidence **consistent with** (or against) demand-related pricing. They are **not proof of any platform's proprietary surge-pricing algorithm.**

---

## Fare prediction notebook

`Chicago_Rideshare_Fare_Prediction.ipynb` builds a **pre-trip fare quote**, like the price an app shows before you book. It starts from `Data/rideshare_2025_clean.parquet`, so run the main notebook first.

**Inputs at booking time:**
- **Pickup location and destination**, entered as a Chicago community area name or number (1–77), e.g. `"Loop"`, `"O'Hare"`. Capitals, spaces and punctuation are ignored, and partial names work.
- **Date and time.**
- **Optional:** expected rain and snow, and how busy the pickup area is right now.

Actual trip minutes and miles are unknown before the trip, so the model uses:
- **Route history:** median miles and minutes of past trips between the same two areas, overall and by time of day. It's built from training months only, so the test months are never seen.
- **Straight-line distance** between the two area centers, for routes with little history.
- **Calendar:** hour, weekday, month, weekend and US federal holidays.
- **Weather** for the day.
- **Previous-hour pickups** in the pickup area, and how that compares with normal. The previous hour is used because the current hour isn't complete at booking time.

**Models compared:**
- **Route median:** the median fare of past trips between the same two areas.
- **Rate card:** base + per minute + per mile, applied to the route's typical minutes and miles.
- **Gradient boosting:** scikit-learn's `HistGradientBoostingRegressor`.

**Evaluation:** train on January–September, choose the model on October, and score once on November–December. The model only ever predicts months it hasn't seen. Measures are MAE, RMSE, MAPE, R² and the share of quotes within \$2.50.

Because fares are rounded to the nearest \$2.50, even a perfect model averages \$0.625 of error. That's the floor to judge MAE against.

**Using the quote:**

```python
quote_fare("Loop", "O'Hare", "2025-12-12 18:00")
list_areas()                       # every community area name and number
```

There are also optional dropdown menus (needs `pip install ipywidgets`). The trained model and its lookup tables are saved to `Output/fare_quote_model.joblib`.

**Main settings (Section 1):**

| Setting | Default | Meaning |
|---|---|---|
| `TRAIN_END` / `VALID_END` | Sep 30 / Oct 31 | Ends of the training and validation periods |
| `TRAIN_ROWS` / `EVAL_ROWS` | 2,000,000 / 1,000,000 | Trips used to fit and to score the models |
| `FARE_MAX_PCTL` | 99.9 | Fares above this percentile of training months are treated as errors and removed |
| `MIN_ROUTE_TRIPS` | 20 | Training trips a route needs before its history is trusted |
| `DEMAND_BASELINE_WEEKS` | 4 | Weeks of past data defining "normal" previous-hour demand |
| `GB_PARAMS` | see notebook | Gradient boosting settings, each with its reason |

**Limitations:**
- **Area-level locations:** the data has community areas, not addresses, so trips within large areas vary.
- **Weather:** actual weather stands in for a forecast.
- **Platform and ride type:** neither is identified.
- **Route distance:** a production quote engine would also use the exact route distance from a map service.

---

## Testing

The modeling pipeline was checked end to end on synthetic data built to match the real file format:
- **Planted effect.** With fares set 15% higher during injected demand bursts, the notebook detected a clear premium in every time category and all 12 sensitivity settings. The rate card was recovered almost exactly (\$3.02 + \$0.250/min + \$1.799/mile vs. \$3.00 + \$0.25/min + \$1.80/mile planted).
- **No effect.** With no premium planted, every estimate was indistinguishable from zero, and the conclusion said so.

- **Fare prediction.** On synthetic trips where distance depends on the pickup and destination areas and fares are rounded to \$2.50, the notebook ran end to end with no errors. Gradient boosting beat both simpler quotes, and distance and route history came out as the most important features.

These tests show the code finds an effect when one exists and does not invent one. They say nothing about the real Chicago result, which comes from running the notebook on the actual data.

---

## References

1. City of Chicago / Data.gov. *Transportation Network Providers – Trips (2025-).* https://catalog-beta.data.gov/dataset/transportation-network-providers-trips-2025
2. Chen, L., Mislove, A., & Wilson, C. (2015). *Peeking Beneath the Hood of Uber.* ACM Internet Measurement Conference. https://mislove.org/publications/Uber-IMC.pdf
3. Ridester. *Uber Rates and Cost* (Chicago UberX sample fare, updated Feb 2024; third-party estimate). https://www.ridester.com/uber-rates-cost/
4. Uber. *New surcharges mandated by the City of Chicago.* https://www.uber.com/us/en/blog/new-surcharges-mandated-by-the-city-of-chicago/
5. NBC Chicago. *Rideshare surcharge takes effect in parts of Chicago.* https://www.nbcchicago.com/news/local/rideshare-surcharge-takes-effect-in-parts-of-chicago-heres-where-and-what-it-costs/3872330/
6. Cameron, A. C., Gelbach, J. B., & Miller, D. L. (2011). *Robust Inference With Multiway Clustering.* Journal of Business & Economic Statistics, 29(2), 238–249.
7. PyFixest documentation. https://py-econometrics.github.io/pyfixest/


## Files included on GitHub

This repository includes the surge analysis notebook, weather data, community-area boundaries, and the generated pickup map. The monthly trip datasets and generated cleaned Parquet dataset remain local because they exceed GitHub's regular file size limit. Download the trip data from the sources listed above and place it in `Data/driveshare_data/` before running the notebook.

The fare prediction notebook mentioned above is not included in this upload.
