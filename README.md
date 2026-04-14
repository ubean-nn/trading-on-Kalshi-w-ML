# Forecasting TSA Volume for Kalshi Markets

_This report is censored and anonymized to protect our proprietary data and pipeline. It shows the research process, the validation standard, and the kinds of signal families we are building around without exposing the features themselves, model configuration, execution thresholds, or order-sizing rules._

<details>
<summary><i>Public-safe scope</i></summary>

The project is not just feature engineering. The full pipeline starts with a Kalshi market, maps that market back to daily TSA passenger throughput, adds external demand and disruption signals, trains forecasting models, evaluates them through walk-forward backtests, and then feeds the forecast output into downstream trading logic. This MD currently focuses on the public research layer: the Kalshi problem framing, the TSA target variable, the Google Trends signal study, and the statistical gates we use before a candidate feature is allowed to influence the model.

What this report intentionally omits:

- Exact search terms, feature names, data sources, and feature preprocessing.
- Model hyperparameters, architecture, feature weights, and ensemble configurations.
- Live trading thresholds, sizing constants, account constraints, and execution rules.
- Raw vendor or market data that cannot be redistributed.

</details>

---

## What is Kalshi?

<p align="center">
  <img src="./media/kalshi-img.png" width="400" alt="Kalshi" />
</p>

Kalshi is a prediction market for all sorts of things. Instead of buying shares of a company, people buy or sell contracts tied to a specific real-world outcome. A contract can be read as a market-implied probability: if people think an event is more likely, the "Yes" side becomes more expensive; if they think it is less likely, it becomes cheaper.

That makes this project a little different from just a normal forecasting assignment. We have to also estimate whether the market's view of a future travel outcome is too high, too low, or fairly priced after accounting for uncertainty.

The market we focus on is tied to U.S. airport checkpoint traffic. In simple terms, the market asks whether TSA passenger volume for a given week will be over or under certain thresholds.

<details>
<summary><i>Technical note: how event contracts change the modeling objective</i></summary>

A point forecast is only one part of the problem. To evaluate a market, we also need uncertainty: a forecast distribution, a way to compare that distribution to market-implied probabilities, and a backtest that checks whether the decision rule would have worked without future information.

This report discusses that pipeline at a high level only. Pricing conversion, live order book handling, risk limits, and sizing rules are intentionally redacted.

</details>

<details>
<summary><i>Technical note: daily target vs. weekly contract</i></summary>

The modeling layer tracks two related metrics:

- Daily error, which diagnoses whether the model captures the within-week travel pattern.
- Weekly-average error, which is closer to the Kalshi contract settlement surface.

This is also why backtesting is walk-forward by week. A model can look acceptable on daily averages but still be misaligned with the contract if it misses the weekly aggregate, or if it uses information that would not have been available at the decision time.

</details>

---

## Our Current Pipeline

```mermaid
graph TD;

    %% TRAINING PIPELINE
    subgraph Model Training & Calibration
        A[Ingest Historical Data: TSA & Public Signals] --> B(Feature Engineering);
        B --> C[Train Forecasting Model];
        C --> D[Walk-Forward Evaluation];
        D --> E[Calibrate Rolling Error & STD];
        E --> F[(Save Model Weights)];
    end

    %% AUTOMATED INFERENCE
    subgraph Daily Inference Engine
        G((Cron Job Trigger)) --> H[Fetch Latest Daily Data];
        H --> I{Data Valid?};
        I -- No --> J[Alert: Data Failure];
        I -- Yes --> K[Load Model Weights];
        F -.-> K;
        K --> L[Generate Daily Forecast];
        L --> M[Process into Quantiles per Contract];
        M --> N[Push Output to Discord Bot];
    end

    %% TRADING & EXECUTION
    subgraph Trading Logic & Execution
        O[Fetch Live Kalshi Contract Prices] --> P[Convert Price to Implied Probability];
        M -.-> Q[Compare Model Quantile vs Implied Probability];
        P -.-> Q;
        Q --> R{Positive Edge Detected?};
        R -- No --> S[Hold Position];
        R -- Yes --> T[Kelly Criterion Order Sizing];
        T --> U[Apply Risk Circuit Breakers];
        U --> V[Execute Kalshi Order];
    end
```

---

## The Target

Like any machine learning/data science project, the first step is to understand the data. In this case, our target is TSA passenger throughput.

It's public, daily, and very seasonal, but there is still a lot of week-to-week variation for outside signals to matter.

The U.S. Transportation Security Administration publishes daily passenger throughput: the number of people screened at airport security checkpoints nationwide.

![TSA EDA Overview](charts/tsa_eda_overview.png)

The structure of the data at a glance is very seasonal. TSA volume has a clear day-of-week pattern, with Sundays consistently busier than Tuesdays as an example. There is also an annual cycle: summer and holiday periods regularly produce much higher throughput than January and February. You can also visualzie the dip in travelling right around the COVID-19 pandemic.

![Monthly Volume Distribution](charts/tsa_monthly_distribution.png)

The monthly distribution is where the forecasting problem becomes interesting. Even after controlling for the month, the range inside a given month can be wide. So a calendar-only model will capture the broad seasonal shape, but it will still miss many week-specific deviations. Those deviations are where external signals, holiday structure, and other highly engineered features can add value.

<p align="center">
It's also the most challenging part of the problem...
</p>
<p align="center">
  <img src="./media/benito-crying-bad-bunny.gif" width="150" alt="bad bunny having to do feature engineering" />
</p>

---

## Google Trends

Google Trends, a data source surrounding google search volume, can act as an early demand signal: people often search for travel-related things before they actually fly. We tested a broad set of travel-adjacent searches, filtered out the candidates that were mostly just seasonal noise, and kept only the signals that survived our validation checks.

We found certain Google Trends terms to be useful, but only after being heavily screened. A lot of travel searches move with TSA volume simply because both rise around the same times of year. The useful signals are the ones that still carry information after we remove the obvious calendar effects.

![Trends Exploration Overview](charts/trends_exploration_overview.png)

The broad screen gave us a quick way to rank candidates and separate "this looks related" from "this might actually help the model."

![Final Validated Signal Summary](charts/trend_correlation_final.png)

The final summary is intentionally anonymized. It shows which signals made it through the validation process without exposing the underlying query list or exact feature recipes.

<details>
<summary><i>Expand Google Trends Analysis</i></summary>

Raw correlation is misleading here. Two series can both peak around July, Thanksgiving, or Christmas and look correlated even if one is not useful for forecasting the other. To avoid that, we evaluated candidate signals after removing major calendar structure from both the TSA series and each trend signal.

The validation pass emphasized:

- Residual correlation instead of raw correlation.
- Lead/lag scans to test whether a signal leads TSA, moves with it, or trails it.
- HAC/Newey-West style significance checks for serially correlated daily data.
- Benjamini-Hochberg FDR correction across the signal-by-lag testing grid.
- Directional stability after outlier handling.
- Pearson/Spearman agreement as a sanity check against a few extreme observations driving the result.

![Trends Exploration Heatmap](charts/trends_exploration_heatmap.png)

The heatmap was the first real filter. It helped us identify which candidates had a stable relationship across nearby lead/lag alignments and which ones only looked good at one isolated point.

![Distributed Lag Profiles](charts/lag_profiles_core_signals.png)

After the broad screen, we moved to a smaller core study. The distributed lag profiles made timing explicit: does the candidate lead TSA volume, move at the same time, or trail it?

![Pearson vs. Spearman Agreement](charts/pearson_spearman_comparison.png)

Pearson and Spearman agreement gave a second robustness check. When both point in the same direction around the same lag, the signal is less likely to be driven by a few extreme observations.

Passing this study does not automatically make a candidate a production feature. A candidate still needs to be evaluated for forecast-time availability, missing-data behavior, interaction with calendar and lag features, and out-of-sample impact in model backtests.

</details>

---

## Weather

Under construction...

**TL;DR.** 

...

At a high level, weather is a different kind of signal from search data. Google Trends is mostly a demand/planning proxy; weather is more of a disruption proxy. The final version should explain what was tested, what survived, and how the surviving weather features fit into the broader forecasting pipeline.

<details>
<summary><i>Expand Weather Analysis</i></summary>

Under construction...

</details>

---

## Model Performance

![Model Performance Leaderboard](charts/model_performance_leaderboard.png)

This is the running model leaderboard for our team. Lower Daily MAE is better and MAPE and Weekly MAE are supporting diagnostics when they are available. These models span the work done by the team for about one and a half semesters.

> [!NOTE] TALON is still in development and being auditted for possible overfitting

We also developed an internal backtester to test each model. The models we use for backtesting are completely seperate and trained on a truncated data set. This ensures theres no leakage or overfitting when preforming inference on the test set. We also made sure to account for all platform fees, real world liquidty constraints, and built it all ontop of historical Kalshi candle data for all the realism.

We test mutliple different strategies that each all handle risk and position sizing differently. At a high level, we use a custom Kelly criterion model around the predicted probabilities, which we get from our autocorrelation-scaled variance model—which dynamically updates our daily standard deviation🤓— to determine position sizing. For clarification, each line on the graph below represent a different trading strategy across our top preforming models.

_Again, our buying strategies are anonymized to protect our edge._

![Bankroll Race](charts/bankroll_race.gif)

Not only did we backtest on historical data, but we also did live testing with paper trading.

<div align="center">

| Date             | Portfolio Value | Cumulative PnL |
| :--------------- | :-------------- | :------------- |
| February 1, 2026 | $12.00          | $8.00          |
| Feb 8, 2026      | $20.00          | $11.00         |
| Feb 15, 2026     | $23.00          | $12.52         |
| Mar 8, 2026      | $24.52          | $18.56         |
| Mar 15, 2026     | $30.56          | $24.32         |
| Mar 22, 2026     | $36.32          | $30.08         |
| Mar 29, 2026     | $42.08          | $30.30         |
| Mar 30, 2026     | $42.30          | $30.30         |

</div>

So we found that our model was able to generate a positive return over the course of the backtest as well as real world paper trading. With portfolio growth of **748%** on the backtest and **252.5%** on two months of live paper trading. As of today, March 9th 2026, we have deployed our model to Kalshi and are live trading with real money.

---
