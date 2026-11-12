# energy-forecasting-dl

**OPIM 5509 (Introduction to Deep Learning) final project.** Your own Keras model forecasts **Connecticut's electricity
load for the next 24 hours, every hour**, on Google Cloud. A class leaderboard scores it live. Built for Dr. Dave's
course at UConn. Every student forks this repo into their **own** energy project on Google Cloud.

(OPIM 5512 uses a different repo, [`energy-forecasting`](https://github.com/drdave-teaching/energy-forecasting), with
scikit-learn models. The two courses never share a repo, a function or a leaderboard.)

## What runs every hour (Eastern time)

| Minute | Cloud Function | Source | Writes to `gs://<bucket>/energy/` |
|---|---|---|---|
| :05 | `scraper-weather-obs` | Iowa State Mesonet, Bradley Airport (BDL) reports | `datasets/bdl_weather_obs_master.csv` (one row per UTC hour) |
| :05 | `scraper-weather-forecast` | [forecast.weather.gov](https://forecast.weather.gov/MapClick.php?lat=41.9389&lon=-72.6832&FcstType=digital), Bradley | `datasets/bdl_weather_forecast_master.csv` (every snapshot: this hour + the next 24, with the NWS issue time) |
| :15 | `scraper-isone` | ISO New England web services, Connecticut zone (4004) | `datasets/ct_load_hourly_master.csv` (official metered, ~2.5 days late), `datasets/ct_load_5min_hourly_master.csv` (live, averaged per hour), `datasets/isone_ct_forecast_master.csv` (ISO-NE's own forecast) |
| :20 | `forecast-24h-dl` | your Keras model in `models/model.keras` + `models/model_card.json` | `forecasts/latest.csv` (24 rows: `target_hour_utc`, `predicted_low_mw`, `predicted_load_mw`, `predicted_high_mw`), `forecasts/history/`, `datasets/forecasts_master.csv`, `forecasts/run_summary.json` |

`forecast-24h-dl` needs the other three: the live load from `scraper-isone` and the weather forecast. `scraper-weather-obs`
is optional: nothing reads it yet, but it keeps a record of the real weather (one of the "Your turn" ideas in FP1 uses it).
All times are stored in **UTC** (`*_utc` columns = the hour the value begins), so daylight saving never duplicates or
drops a row.

## Set up (once)
Follow the **[OPIM 5509 Energy Project Setup Guide](https://colab.research.google.com/github/drdave-teaching/OPIM5509-notebooks/blob/main/FinalProject/Energy_Project_Setup_5509.ipynb)**:
a new Google Cloud project, **fork this repo**, Workload Identity Federation, 6 repository **variables** (`PROJECT_ID`,
`REGION`, `BUCKET_NAME`, `RUNTIME_SA`, `DEPLOYER_SA`, `WORKLOAD_IDENTITY_PROVIDER`) and 2 repository **secrets** from your
own free ISO Express account (`ISO_NE_USERNAME`, `ISO_NE_PASSWORD`). Without the secrets, `scraper-isone` deploys but
reports `"status": "skipped"`.

Then **Actions → Run workflow** on the four deploy workflows. Get course updates with **Sync fork → Update branch**, then
run "Deploy forecast-24h-dl" again.

## Train and ship a model
Train in Colab with the final project notebooks:
[FP1, a basic LSTM](https://github.com/drdave-teaching/OPIM5509-notebooks/blob/main/FinalProject/FP1_Basic_LSTM_Forecast.ipynb),
then [FP2, an LSTM that gives a range](https://github.com/drdave-teaching/OPIM5509-notebooks/blob/main/FinalProject/FP2_Probabilistic_LSTM.ipynb).
Each one ends by uploading `models/model.keras` and `models/model_card.json` to your bucket. A copy of every model you
ship is kept in `models/history/`.

Every hour `forecast-24h-dl` loads your model and builds its input with the **same function as the notebook**:
`make_window` for a one-input model (FP1: one 192-row table, the past week + the next 24 hours) or `make_inputs` for a
two-input model (FP2: past and future). The weather comes from the newest HRRR runs in Dr. Dave's class file that reach a
full day ahead, or the NWS forecast if that file is missing or stale. The card's `range_method` picks the range:

| `"range_method"` | Model | Low, expected and high |
|---|---|---|
| `"none"` | FP1 (one number per hour) | expected value only, newest weather run |
| `"lstm_hrrr_runs"` | FP1 or FP2 | the 10th, 50th and 90th percentile of the forecasts made with each of the newest `hrrr_runs` runs (default 3) |
| `"lstm_quantiles"` | FP2 (20 percentiles per hour) | the model's own 10th, 50th and 90th percentiles, newest weather run |
| `"lstm_quantiles_pooled"` | FP2 | every run's 20 percentiles pooled into one pile per hour |

`"range_widen": 1.3` (optional) stretches the range 30% around the expected value. When the NWS fallback is used,
`range_method` gets `_nws` added to it.

Every run also writes `forecasts/run_summary.json`: the function's name, the Keras version and your network's layers.
The OPIM 5509 leaderboard reads it and only scores forecasts made by `forecast-24h-dl` with a Keras network.

## Optional: your own public dashboard (no points)
[`dashboard/`](dashboard/) is a [Streamlit](https://streamlit.io) app that reads your bucket and shows your next 24 hours,
every model you shipped, your past forecasts against the real load, and your score. One workflow puts it on Google Cloud
Run with no keys. See [dashboard/README.md](dashboard/README.md).

## Data credits
Weather: Iowa State Mesonet (mesonet.agron.iastate.edu), the National Weather Service and NOAA's HRRR model. Load and
forecasts: ISO New England (iso-ne.com, ISO Express web services). Each student uses their own ISO Express account.
