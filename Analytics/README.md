# Agentic AI-Based Dynamic Tariff Optimization for EV Charging Networks

This repository contains the end-to-end agentic AI pipeline for EV dynamic tariff optimization, developed for the **Society of Business Open Project 2026**.

The system is designed to ingest real-world charging session data (ACN-Data commuter logs and ST-EVCDP public grid logs), predict localized demand and congestion, dynamically optimize tariffs (surge vs off-peak discounts), and simulate a self-improving feedback loop where the pricing agent adjusts its policy parameters over time.

---

## 1. Project Overview & Multi-Agent Architecture

The pipeline consists of three specialized agents interacting in a closed loop:

1.  **Demand Prediction Agent:** Forecasts grid-level charger utilization rates and binary congestion probabilities (utilization > 80%) using machine learning.
2.  **Tariff Pricing Agent:** Optimizes per-kWh pricing, recommending surge rates (up to 1.5x) during high congestion and off-peak discounts (30% off) during low utilization, using a causal demand elasticity model.
3.  **Monitoring & Learning Agent:** Evaluates outcomes day-by-day (episodes), computing a Pricing Efficiency Score (PES), Customer Response Rate (CRR), and Waiting Time Reduction. If metrics decline, the agent automatically adapts policy parameters (e.g. lowering the surge threshold) for the next episode.

---

## 2. Key Success Metrics & Results

*   **Demand Agent Accuracy:** XGBoost was selected as the best regressor, achieving $R^2 = 0.9148$ (RMSE = 0.0516, MAE = 0.0309) on out-of-sample temporal test data.
*   **Off-Peak Energy Uplift:** Dynamic discounts successfully shifted demand to off-peak periods, yielding a **+8.88% increase** in off-peak energy volume.
*   **Learning Efficiency:** The Pricing Efficiency Score (PES) increased from **12.00 to 12.38** over 6 episodes, demonstrating that the self-improving feedback loop works.

---

## 3. Folder Structure

```
.
├── data/
│   ├── raw/
│   │   ├── acndata_sessions_json.xlsx        # Raw ACN session data
│   │   └── UrbanEV_ SZ_districts/            # Raw ST-EVCDP CSV files
│   └── processed/
│       ├── acn_clean.csv                     # Cleaned ACN sessions (anomaly-free)
│       ├── stevcdp_panel.csv                 # Melted and spatially joined grid panel
│       └── unified_features.csv              # Unified hourly site/grid metrics
│
├── notebooks/                                # Executed Jupyter Notebooks
│   ├── 01_data_preprocessing.ipynb
│   ├── 02_eda.ipynb
│   ├── 03_demand_prediction_agent.ipynb
│   ├── 04_tariff_pricing_agent.ipynb
│   └── 05_monitoring_learning_agent.ipynb
│
├── outputs/
│   ├── figures/                              # Generated high-res charts (1-13)
│   ├── results/                              # Output CSV metrics and predictions
│   │   ├── demand_predictions.csv
│   │   ├── dynamic_tariffs.csv
│   │   ├── model_metrics.csv
│   │   └── monitoring_metrics.csv
│   └── presentation_outline.md               # Structured 6-slide deck outline
│
├── requirements.txt
└── README.md
```

---

## 4. Setup and Execution Instructions

### Prerequisites
- macOS/Linux/Windows
- Python 3.10+
- `uv` (recommended) or `pip`

### Step 1: Install Dependencies
Create a virtual environment and install the requirements:
```bash
# Using uv (very fast)
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt

# Or using standard pip
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Step 2: Run the Pipeline Notebooks
To run the notebooks sequentially from the command line:
```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/*.ipynb
```
Or open the notebooks inside Jupyter Lab / VS Code and execute them cell by cell.

---

## 5. Preprocessing & Feature Engineering Assumptions

*   **ACN Data Cleaning:** Sessions exceeding 48 hours are dropped as outliers (abandoned links). Negative durations are capped at 0.
*   **Spatial Join:** Station coordinates in `stations.csv` are mapped to the nearest grid center in `information.csv` using the Haversine formula to compute aggregated charger capacities.
*   **Occupancy Density:** Modeled as `occupancy / area` for Shenzhen grids.
*   **Energy Cost:** Modeled as a Time-of-Use (ToU) baseline cost (₹12/kWh peak, ₹8/kWh off-peak).
*   **Queue Length Proxy:** Modeled as `max(0, occupancy - pile_count)`.
