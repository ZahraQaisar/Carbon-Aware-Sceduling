# Carbon-Aware Scheduling

Forecasting how carbon-heavy Great Britain's electricity will be over the next 24 hours, and picking the cleanest 3-hour window of the day.

Term project for Data Science Tools and Techniques (MSDS).
Zahra Qaisar (26L-8024) and Kainat Bashir (26L-8026)

## What this is

The carbon intensity of the grid (grams of CO2 per kWh of electricity) changes through the day. It drops when there is a lot of wind and rises when gas is doing more of the work. If you know in advance when the grid will be cleaner, you can charge an electric car or run the washing machine then.

Given a date, our model predicts the carbon intensity for each of the 48 half-hours of that day. It only uses information that would be available at 23:30 UTC the day before: yesterday's carbon intensity and fuel mix, the pattern from last week, the weather, and the calendar. It then recommends the 3-hour window with the lowest predicted average.

We used Great Britain because the data is public at 30-minute resolution, for the years 2022 to 2025.

## What we did

Part 1 (`notebooks/part-1.ipynb`) collects and cleans the data, builds the features and sets up the baselines.

Part 2 (`notebooks/part-2.ipynb`) trains and compares Ridge, Random Forest and LightGBM, tunes them, tests a few ideas to improve them, and then evaluates everything on 2025, a year the models never saw. We split by time (train on 2022-2023, validate on 2024, refit on 2022-2024, test on 2025), because a random split would let the model learn from the future.

The final model is a blend of 80% Ridge and 20% LightGBM. We chose it using the 2024 validation year and a time-based cross-validation, before looking at the test results.

## Results

On the 2025 test days (361 days, all methods compared on the same days):

| Model | Average error (MAE, gCO2/kWh) |
|---|---|
| Final blend (Ridge + LightGBM) | 19.81 |
| Ridge | 20.55 |
| LightGBM | 19.65 |
| Random Forest | 22.65 |
| "Same as yesterday" | 38.87 |
| Average of the last 7 days | 42.08 |

- Our model has about half the error of the best simple baseline (RQ1).
- The error grows through the forecast day: 3.65 for the first half-hour and 29.69 for the last (RQ2).
- Using one-day-ahead weather forecasts instead of historical weather did not make the model worse (19.27 vs 19.81) (RQ3).
- The 3-hour window we pick is on average 5.82 gCO2/kWh worse than the best possible window, against 21.72 for "same as yesterday". It lands within an hour of the best start time on 58.7% of days, against 31.0%. (RQ4)
- The weather of the target day turned out to be by far the most useful input. Removing it pushes the error up from 20.75 to 27.07 in our test with Ridge.

![Carbon intensity over time and the train/validation/test split](figures/02_carbon_intensity_over_time_and_the_data_split.png)

![Test results on 2025](figures/10_test_results_on_2025_lower_is_better.png)

![Error for each of the 48 half-hours](figures/13_rq2_mae_for_each_of_the_48_half_hours_2025.png)

![Window regret and hit rate](figures/15_regret_lower_is_better.png)

## Limits

- The official NESO forecast has an error of 9.19, much lower than ours. It is made with a different forecast horizon, so it is not a fair comparison, but we do not beat it.
- On the 2025 test set, LightGBM on its own was slightly better than the blend. We kept the blend because we had chosen it on validation data, and switching after seeing the test results would defeat the point of a test set.
- Carbon intensity fell from 182.5 in 2022 to 129.2 in 2025, so models trained on older years tend to predict too high. Our final model is still about 7.7 gCO2/kWh too high on average.
- The weather we call "historical" also comes from a weather-model API, so it is close to the forecast data. The RQ3 result probably understates what real forecast errors would cost.
- Windy days and winter are harder, and our 80% prediction ranges do not widen on difficult days.
- We measure how clean the recommended window is (carbon intensity), not actual CO2 saved. That would also depend on how much electricity someone can move. We also predict the average carbon intensity, not the marginal one.

## Running it

1. `pip install -r requirements.txt`
2. Open `notebooks/part-2.ipynb` in Jupyter, or upload it to Kaggle. If the Kaggle dataset folder is not found, it reads the files from `data/`.
3. The notebook writes its outputs to `/kaggle/working`. On Kaggle that folder exists already; on your own computer create it first.
4. Run all cells. The 2025 weather forecast is read from `data/previous_day1_2025.csv`, or downloaded from Open-Meteo if the file is missing.
5. The last cells download today's data and forecast today. They need internet and are skipped if the download fails, so the rest of the notebook is not affected.
6. To forecast a single date, call `predict("2025-08-01")` after running the notebook.

## Repository layout

    notebooks/   part-1.ipynb, part-2.ipynb
    data/        master_clean.csv, selected_features.csv, previous_day1_2025.csv
    models/      final_model.joblib
    results/     result tables (csv)
    figures/     all plots from Part 2

## Data and credits

- Carbon Intensity API, National Energy System Operator (NESO), CC BY 4.0: https://api.carbonintensity.org.uk
- Open-Meteo weather APIs (Historical Forecast, Previous Runs, Forecast), CC BY 4.0: https://open-meteo.com