# Agentic AI-Based Dynamic Tariff Optimization for EV Charging Networks

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg?logo=python&logoColor=white)](https://www.python.org/)
[![XGBoost](https://img.shields.io/badge/ML-XGBoost%20%7C%20LightGBM%20%7C%20RandomForest-brightgreen)](https://xgboost.readthedocs.io/)
[![Jupyter](https://img.shields.io/badge/Notebooks-Jupyter-orange.svg?logo=jupyter&logoColor=white)](https://jupyter.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

An end-to-end **multi-agent AI pipeline** for dynamic tariff optimization across EV charging networks, developed for the **Society of Business Open Project 2026**. The system ingests real-world charging session logs (ACN-Data commuter sessions + ST-EVCDP public grid logs), forecasts localized demand and congestion, dynamically prices electricity per kWh, and self-improves its pricing policy through a closed-loop feedback architecture.

---

## Table of Contents
1. [Core Accomplishments](#1-core-accomplishments)
2. [Multi-Agent Architecture](#2-multi-agent-architecture)
3. [Model Performance Results](#3-model-performance-results)
4. [Self-Improving Feedback Loop Results](#4-self-improving-feedback-loop-results)
5. [Folder Structure](#5-folder-structure)
6. [Preprocessing & Feature Engineering Assumptions](#6-preprocessing--feature-engineering-assumptions)
7. [Setup and Execution Instructions](#7-setup-and-execution-instructions)

---

## 1. Core Accomplishments

| Metric | Result |
| :--- | :--- |
| **Best Model R² Score** | **0.9953** (RandomForest, out-of-sample temporal test) |
| **Best Model RMSE** | **0.0121** |
| **Off-Peak Energy Uplift** | **~9.00%** increase in off-peak energy volume |
| **Pricing Efficiency Score (PES) Gain** | **12.136 → 12.655** over 6 self-improving episodes |
| **Daily Revenue Growth** | **₹35.88M → ₹38.68M** (Episode 1 → Episode 6) |
| **Surge Threshold Calibrated by Agent** | **0.80 → 0.70** (auto-adapted over 6 episodes) |

---

## 2. Multi-Agent Architecture

The pipeline consists of **three specialized agents** interacting in a closed loop:

```
Raw Data (ACN-Data + ST-EVCDP)
          │
          ▼
 ┌─────────────────────────┐
 │  Data Preprocessing     │  — Spatial join, ToU cost, queue proxy
 └────────────┬────────────┘
              │
              ▼
 ┌─────────────────────────┐
 │  1. Demand Prediction   │  — XGBoost / RandomForest
 │       Agent             │    Forecasts utilization & congestion P
 └────────────┬────────────┘
              │  congestion_probability, predicted_utilization
              ▼
 ┌─────────────────────────┐
 │  2. Tariff Pricing      │  — Surge 1.5x if util > threshold
 │       Agent             │    Discount 0.7x if util < 0.30
 └────────────┬────────────┘
              │  dynamic_tariffs.csv
              ▼
 ┌─────────────────────────┐
 │  3. Monitoring &        │  — Computes PES, CRR, off-peak uplift
 │     Learning Agent      │    Adapts surge_threshold if PES drops
 └────────────┬────────────┘
              │  updated policy parameters
              └──────────────────────────► back to Agent 1 (next episode)
```

**Agent Responsibilities:**

1. **Demand Prediction Agent** — Trains an ensemble regressor on historical grid-level features (occupancy density, queue length proxy, ToU cost, hour-of-day, day-of-week) to output `predicted_utilization` and a binary `congestion_probability` (threshold: utilization > 80%).

2. **Tariff Pricing Agent** — Applies a causal demand elasticity model to determine per-kWh pricing:
   - **Surge** (up to 1.5× base rate): triggered when `congestion_probability` is high
   - **Off-Peak Discount** (0.7× base rate): triggered during low utilization windows

3. **Monitoring & Learning Agent** — Evaluates day-by-day episodes and computes:
   - **Pricing Efficiency Score (PES)**: composite measure of load balance and revenue quality
   - **Customer Response Rate (CRR)**: demand shift response to pricing signals
   - **Off-Peak Uplift %**: percentage increase in off-peak energy consumed
   - Automatically lowers `surge_threshold` by 0.02 if PES declines, tightening control

---

## 3. Model Performance Results

Four gradient-boosted and ensemble models were benchmarked on out-of-sample temporal test data to forecast charger utilization rates. Results from [`results/model_metrics.csv`](results/model_metrics.csv):

| Model | RMSE ↓ | MAE ↓ | R² Score ↑ |
| :--- | :---: | :---: | :---: |
| **RandomForest** ✅ | **0.01212** | **0.00463** | **0.9953** |
| LightGBM | 0.01269 | 0.00507 | 0.9949 |
| XGBoost | 0.01283 | 0.00525 | 0.9947 |
| HistGradientBoosting | 0.01310 | 0.00530 | 0.9945 |

> **RandomForest** achieved the highest R² and lowest error, making it the selected model for live congestion inference in the pipeline.

---

## 4. Self-Improving Feedback Loop Results

The Monitoring & Learning Agent was evaluated across **6 daily episodes** (July 13–18, 2022). It autonomously reduced the surge trigger threshold from **0.80 → 0.70**, enabling the network to intervene earlier, shift more demand off-peak, and grow revenue. Results from [`results/monitoring_metrics.csv`](results/monitoring_metrics.csv):

| Episode | Date | Surge Threshold | PES ↑ | Revenue (₹) ↑ | Off-Peak Uplift | Energy (kWh) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 1 | 2022-07-13 | 0.80 | 12.136 | 35,879,225 | 8.990% | 2,956,495 |
| 2 | 2022-07-14 | 0.78 | 12.236 | 36,919,580 | 8.998% | 3,017,186 |
| 3 | 2022-07-15 | 0.76 | 12.381 | 37,836,042 | 8.997% | 3,055,936 |
| 4 | 2022-07-16 | 0.74 | **12.558** | **39,108,511** | 8.996% | 3,114,116 |
| 5 | 2022-07-17 | 0.72 | 12.499 | 37,426,386 | 8.997% | 2,994,282 |
| 6 | 2022-07-18 | **0.70** | **12.655** | 38,683,712 | 8.996% | 3,056,684 |

**Key Observations:**
- **Episode 4** achieved peak single-day revenue of ₹39.1M, triggered by the agent lowering the threshold to 0.74.
- **Episode 5** showed a brief PES dip (12.499) — the agent correctly identified this regression and tightened the threshold further.
- **Episode 6** recovered to the highest recorded PES of **12.655**, validating the adaptive policy loop.
- The off-peak uplift stabilized at ~**9.00%** across all episodes, demonstrating the pricing signal's consistent effectiveness in shifting load.

---

## 5. Folder Structure

```
Analytics/
├── requirements.txt
├── 23411030_EV_Tariff_Optimization_OP26.pdf   # Project Report & Presentation
│
├── notebooks/
│   ├── 01_data_preprocessing.ipynb            # Data cleaning, spatial join, feature engineering
│   ├── 02_eda.ipynb                           # Exploratory Data Analysis & visualizations
│   ├── 03_demand_prediction_agent.ipynb       # Model training, benchmarking, selection
│   ├── 04_tariff_pricing_agent.ipynb          # Surge/discount logic & elasticity modelling
│   └── 05_monitoring_learning_agent.ipynb     # Episode simulation, PES, adaptive policy loop
│
└── results/
    ├── model_metrics.csv                      # RMSE / MAE / R² for all trained models
    ├── demand_predictions.csv                 # Per-grid, per-timestep utilization predictions
    ├── dynamic_tariffs.csv                    # Per-session pricing decisions (76 MB)
    └── monitoring_metrics.csv                 # Episode-level PES, CRR, revenue, uplift metrics
```

---

## 6. Preprocessing & Feature Engineering Assumptions

- **ACN Data Cleaning:** Sessions exceeding 48 hours are dropped as outliers (abandoned EV links). Negative durations are capped at 0.
- **Spatial Join:** Station GPS coordinates (`stations.csv`) are mapped to the nearest Shenzhen grid centroid in `information.csv` using the **Haversine formula**, enabling aggregation of charger capacities per grid cell.
- **Occupancy Density:** Modeled as `occupancy / area` for each Shenzhen grid zone.
- **Energy Cost Baseline:** Time-of-Use (ToU) pricing — ₹12/kWh during peak hours, ₹8/kWh during off-peak hours.
- **Queue Length Proxy:** Estimated as `max(0, occupancy − pile_count)` to simulate wait time without real queue data.

---

## 7. Setup and Execution Instructions

### Prerequisites
- macOS / Linux / Windows
- Python 3.10+
- `uv` (recommended) or `pip`

### Step 1: Install Dependencies
```bash
# Using uv (fast)
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt

# Or using standard pip
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Step 2: Run the Pipeline
Execute all notebooks sequentially:
```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/*.ipynb
```
Or open them one-by-one in Jupyter Lab / VS Code and run cell-by-cell.
