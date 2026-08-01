# Price-Bench

<p align="center">
  <img src="https://img.shields.io/badge/status-concept%20%2F%20early%20development-orange" alt="Status: Early Development">
  <img src="https://img.shields.io/badge/Python-3.9%2B-blue" alt="Python 3.9+">
  <img src="https://img.shields.io/badge/License-Apache%202.0-green" alt="License: Apache 2.0">
</p>

<p align="center">
  <b>A cross-modal benchmark for predicting local crop price responses to supply shocks from pre-harvest remote sensing and weather</b>
</p>

---

## Overview

Local agricultural markets are often informationally inefficient. A smallholder market in a landlocked district may not fully price in a regional drought or bumper crop until physical harvest arrives and traders adjust bids. **Pre-harvest supply signals** — derived from satellite vegetation indices, weather anomalies, or crop condition reports — could theoretically improve local price forecasts beyond what lagged price trends or global commodity futures alone provide. Yet no benchmark exists to test this cross-modal hypothesis at scale.

Existing price forecasting benchmarks are pure time-series tasks: they predict future price from past price, macro indicators, and global futures. They do not evaluate whether **agronomic supply-side information** (remote sensing, weather, early yield estimates) improves local price prediction. Conversely, existing crop yield benchmarks predict biophysical production but do not evaluate the **downstream economic consequences** of those predictions.

**Price-Bench** proposes a cross-modal benchmark: given **pre-harvest supply indicators** for a region (satellite-derived yield anomalies, weather shocks, or biophysical production proxies) and **market structure context** (transport access, storage capacity, trade policy), predict the **post-harvest local price response** — defined as the deviation of local price from its expected seasonal trend. The benchmark tests whether models can bridge agronomic supply information and economic price formation.

---

## Motivation

### The Cross-Modal Problem

In efficient global commodity markets, prices rapidly incorporate supply expectations. In fragmented local markets, this incorporation is slower and noisier. A model that knows a region's maize crop is 30% below normal before harvest could anticipate a local price spike that a pure price-history model misses. But this requires fusing:

- **Agronomic signals:** Vegetation health, rainfall anomalies, temperature stress during grain-filling
- **Economic context:** Market isolation, storage availability, trade policy, global price transmission
- **Temporal structure:** Supply signals accumulate during the growing season; price responses manifest after harvest

No existing benchmark provides standardized, paired agronomic-economic data with a causal temporal structure (growing-season inputs → post-harvest outcomes).

### Why Existing Benchmarks Are Insufficient

| Benchmark | Supply-Side Inputs | Local Price Focus | Cross-Modal Evaluation | Causal Temporal Structure |
|-----------|-------------------|-------------------|------------------------|---------------------------|
| **Commodity futures datasets** | Global production reports | No (global/national) | No | Varies |
| **M-competitions (M4/M5)** | None | No | No | Yes |
| **Satellite yield benchmarks** | Full remote sensing | No (predicts yield, not price) | N/A | Yes |
| **Single-country price studies** | Varies | Yes | Ad hoc | Varies |

None provide a **multi-region benchmark** where the core task is to predict **local price deviation from trend using pre-harvest supply proxies** with explicit train-test separation between growing-season (supply) and post-harvest (price) windows.

---

## Proposed Benchmark Design

### Task Definition

**Primary Task (Regression):**  
Given pre-harvest supply indicators (e.g., satellite vegetation anomaly, rainfall deviation, growing degree-day shortfall) for a region-crop combination during a specific growing season, plus market structure covariates, predict the **post-harvest local price response** — defined as the percentage deviation of actual local price from the expected seasonal price trend for that market-crop.

**Secondary Task (Anomaly Detection):**  
Classify whether the region will experience a **supply-driven price anomaly** (e.g., price deviation exceeding ±20% from trend) in the post-harvest window.

**Tertiary Task (Spatial Transfer):**  
Given a model trained on price-supply relationships in one set of regions, predict price responses in **held-out regions** with similar market structures but distinct geographic locations. This tests whether models learn transferable supply-shock dynamics or merely memorize local price histories.

### Dataset Requirements (Proposed)

| Criterion | Specification |
|-----------|---------------|
| **Unit of observation** | Region-market-crop-season tuple |
| **Supply indicators** | Pre-harvest remote sensing anomalies (e.g., NDVI deviation from multi-year mean), weather anomalies (rainfall, temperature), or crop condition indices |
| **Price data** | Post-harvest local wholesale or retail prices (weekly or monthly) |
| **Price baseline** | Expected seasonal trend (e.g., historical mean for that market-crop-week) |
| **Market structure** | Transport cost proxy, storage capacity indicator, distance to major trade hub, trade policy flags |
| **Global context** | Corresponding global commodity futures price at harvest (optional) |
| **Crops** | Multiple staple commodities |
| **Regions** | Multiple markets across distinct agro-ecological and economic zones |
| **Temporal structure** | Strict causal split: supply features from growing season; price targets from post-harvest window |
| **Splits** | Temporal split (unseen seasons); spatial split (unseen regions); combined spatiotemporal split |

### Evaluation Protocol (Proposed)

| Metric | Purpose |
|--------|---------|
| **RMSE** | Accuracy of price deviation prediction |
| **MAE** | Robustness to outliers |
| **MAPE** | Interpretable percentage error |
| **Directional accuracy** | Correct prediction of price increase vs. decrease vs. trend |
| **Anomaly F1** | For spike/crash detection |
| **Skill score** | Improvement over naive seasonal-trend baseline |

**Cross-validation:** Strict causal split (growing-season features only; no post-harvest information leakage). Spatial split (hold out entire regions). Random temporal splits provided for diagnostic comparison only.

### Planned Baselines

1. **Seasonal trend baseline** — Predict zero deviation from historical mean price for that market-crop-week
2. **Global futures baseline** — Predict local price deviation from global commodity futures movement alone
3. **Supply-shock regression** — Linear regression on supply anomaly indices
4. **Classical machine learning** — Random Forest, XGBoost fusing supply anomalies and market structure
5. **Cross-modal deep learning** — Architecture with separate encoders for temporal supply series and static market context, fused for price response prediction

---

## Current Status

This repository is in **early development**. No stable dataset or working API is available yet.

| Component | Status |
|-----------|--------|
| Literature survey for paired supply-price datasets | In progress |
| Data schema design | Proposed |
| Data loaders | Planned |
| Causal temporal splitting | Planned |
| Seasonal trend baseline | Planned |
| Supply-shock regression baseline | Planned |
| Classical ML baselines | Planned |
| Cross-modal baselines | Planned |
| Leaderboard | Planned |

---

## Proposed Dataset Schema

```python
{
    "region_id": str,                 # Unique region-market identifier
    "region_name": str,               # Human-readable location
    "country": str,                   # Country code
    "crop": str,                      # e.g., "maize", "rice", "wheat"
    "season_year": int,               # Growing season / harvest year
    "planting_date": str,             # ISO date
    "harvest_date": str,              # ISO date (price window begins after this)
    "supply_indicators": {            # Pre-harvest, growing-season features
        "ndvi_anomaly": float,        # Deviation from multi-year mean NDVI
        "rainfall_anomaly_pct": float,# Seasonal rainfall deviation
        "gdd_shortfall": float,       # Growing degree day deficit
        "crop_condition_index": Optional[float],
    },
    "market_structure": {
        "distance_to_hub_km": float,
        "storage_capacity_index": Optional[float],
        "road_quality_index": Optional[float],
        "trade_policy_flag": Optional[int],  # e.g., 1 if export restriction
    },
    "global_context": {
        "harvest_futures_price": Optional[float],  # Global futures at harvest
    },
    "price_response": {               # Post-harvest targets
        "local_price": float,         # Observed local price
        "expected_trend_price": float,# Historical seasonal expectation
        "deviation_pct": float,       # Target: (local - trend) / trend * 100
        "is_anomaly": bool,           # True if |deviation| > threshold
    },
    "source": str,                    # Data provider
    "source_doi": Optional[str],      # Literature provenance
}
```

*The above schema is a design target. The actual schema may evolve during data curation.*

---

## Installation (Proposed)

```bash
git clone https://github.com/your-org/Price-Bench.git
cd Price-Bench

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install -e .
```

### Expected Dependencies

- Python ≥ 3.9
- pandas, numpy
- scikit-learn
- PyTorch (optional, for deep learning baselines)

---

## Usage (Illustrative)

The following examples illustrate the intended API. They are not guaranteed to run until the v0.1 release.

```python
from price_bench.datasets import SupplyPriceDataset
from price_bench.models import TrendBaseline
from price_bench.evaluation import BenchmarkEvaluator

dataset = SupplyPriceDataset(split="spatiotemporal")
model = TrendBaseline()

evaluator = BenchmarkEvaluator(model, dataset=dataset)
results = evaluator.run()
print(results)
```

```python
from price_bench.evaluation import BenchmarkEvaluator

evaluator = BenchmarkEvaluator(your_model, dataset="supply_price_dev")
results = evaluator.run()
```

---

## Repository Structure

```
Price-Bench/
├── price_bench/           # Main Python package (planned)
│   ├── datasets/          # Data loaders, causal splitters
│   ├── models/            # Baseline implementations
│   ├── features/          # Supply anomaly construction, market encoding
│   ├── evaluation/        # Metrics, causal and spatial cross-validation
│   └── utils/             # Helper functions
├── data/
│   ├── raw/               # Raw supply and price data (with attribution)
│   └── processed/         # Cleaned, standardized datasets
├── experiments/           # Training scripts for baselines
├── tests/                 # Unit tests
├── notebooks/             # Exploratory data analysis
├── requirements.txt
├── setup.py
├── LICENSE
└── README.md
```

---

## Roadmap

| Milestone | Target | Deliverable |
|-----------|--------|-------------|
| **v0.1** | TBD | Frozen dataset for 1–2 crops across 3+ regions; seasonal trend + supply-shock regression + RF baselines; causal temporal and spatial splits |
| **v0.2** | TBD | Expanded crop and regional coverage; XGBoost and cross-modal baselines; anomaly detection task |
| **v0.3** | TBD | Spatial transfer protocol; multi-season evaluation; public leaderboard |
| **v1.0** | TBD | Final dataset freeze; benchmark paper submission; community challenge |

---

## Citation

```bibtex
@software{price_bench,
  title = {Price-Bench: A Cross-Modal Benchmark for Predicting Local Crop Price Responses to Supply Shocks},
  author = {TBD},
  year = {2026},
  url = {https://github.com/your-org/Price-Bench},
  note = {Concept and early development release}
}
```

---

## Contributing

We welcome contributions in:

- **Data curation:** Paired pre-harvest supply indicators and post-harvest local price series, with clear provenance and attribution
- **Supply proxies:** Standardized remote sensing anomaly pipelines or weather shock indicators
- **Baselines:** Reference implementations of cross-modal or econometric models
- **Evaluation:** Protocols for causal temporal splitting and spatial generalization across market types

Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

---

## Contact

- **Issues:** [GitHub Issues](https://github.com/your-org/Price-Bench/issues)
- **Discussions:** [GitHub Discussions](https://github.com/your-org/Price-Bench/discussions)

---

<p align="center">
  <i>This is a living research document. The benchmark design, dataset, and protocols are open to community feedback prior to the v0.1 freeze.</i>
</p>
```
