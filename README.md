# Demand Planning + Demand Sensing on Grocery POS Data (Favorita)

An end-to-end supply chain AI project: an **ML demand plan** (weekly) with a **demand sensing layer** (daily) on top, evaluated by **Forecast Value Add (FVA)** by product category, and finally an **AI planner agent** that explains forecast changes.

This repository currently contains **Phase 2 – data preparation**: a cleaned, documented, model-ready dataset built in SQL (DuckDB) from 125.5M rows of grocery sales.

> Companion repo (SQL foundations on Walmart M5): [demand-planning-sql-m5](https://github.com/bhupendrachintale/demand-planning-sql-m5)

---

## Business problem

Favorita, a grocery chain in Ecuador, replenishes each store based on an item × store × day forecast. A forecast built only on sales history reacts slowly to short-term change — a promotion starting, a local holiday, a hot week, a drop in store traffic. For groceries, especially perishables, that means waste when over-forecast and empty shelves when under-forecast.

**Question:** Can short-term signals (promotions, holidays, store traffic, weather, oil price) improve the 1–14 day forecast compared with a weekly ML demand plan — by how much, and for which categories?

| | Demand plan | Demand sensing |
|---|---|---|
| Grain | item × store × week | item × store × day |
| Horizon | 1–12 weeks | 1–14 days |
| Signals | Known in advance: history, seasonality, calendar, holidays, planned promotions | Plan signals + observed: recent sales, store traffic, weather, oil (lagged) |

**FVA chain (planned):** naive baseline → ML demand plan → demand sensing, measured with WMAPE and bias, by perishable vs non-perishable and promo vs non-promo.

---

## Project status

| Phase | Status |
|---|---|
| 1. SQL foundations (M5) | ✅ Done — separate repo |
| **2. Data prep (this repo)** | ✅ **Done** |
| 3. Python analysis | ⏳ Next |
| 4. ML demand plan (LightGBM vs naive) | ⬜ |
| 5. Demand sensing + FVA | ⬜ |
| 6. AI planner agent (Claude API) | ⬜ |

---

## Data

| Source | Content |
|---|---|
| [Corporación Favorita Grocery Sales Forecasting](https://www.kaggle.com/competitions/favorita-grocery-sales-forecasting) (Kaggle) | Daily item × store sales 2013–2017 (125.5M rows), promotion flag, items, stores, holidays/events, store transactions, oil price |
| [Open-Meteo Historical Weather API](https://open-meteo.com/) | Daily max/min/mean temperature and precipitation for each store city |

Raw data is **not** included (size and Kaggle terms). Download it from the links above into `data/raw/`.

---

## Scope

| Dimension | Scope | Why |
|---|---|---|
| Dates | Weekly plan 2014-04-01 → 2017-08-15; daily sensing 2016-01-01 → 2017-08-15 | Promo flag recorded from Apr 2014; ~3.4 years for seasonality |
| Families | Beverages, Dairy, Meats | Perishable vs non-perishable; low vs high promo intensity; weather-sensitive beverages |
| Stores | 13 stores in 9 cities (highlands and coast), all 5 store types | Climate contrast for weather; full history; no unusual gaps |
| Items | 813 active items; 8,133 item × store pairs | Enough history and still selling at the end of the data |

---

## Notebooks

| Notebook | What it does |
|---|---|
| `00_data_profiling` | Profiles every raw file (grain, keys, types — proven with counts) and loads 7 tables with deliberate types and naming |
| `01_data_quality` | 8 data-quality checks with a scorecard and a decision for every issue |
| `02_scope_selection` | Data-driven selection of families, stores and items |
| `03_weather` | Downloads daily weather for 9 cities from the Open-Meteo API (Python) |
| `04_build_combined` | Builds the calendar spine and the final daily and weekly tables |

---

## Data-quality scorecard

| # | Check | Result | Decision |
|---|---|---|---|
| 1 | Duplicate keys (date, store, item) | 0 | Safe to join on business key |
| 2 | Unknown promotion flag | 17.3% of rows, one clean block before Apr 2014 | Out of scope (scope starts Apr 2014) |
| 3 | Negative sales (net returns) | 0.006% of rows | Keep raw; clean column clips to 0 |
| 4 | Orphan items / stores | 0 / 0 | All rows get master data |
| 5 | Oil gaps | 31% of calendar days | Forward-fill last known price (no leakage) |
| 6 | Holiday table key | Date not unique (31 dates repeat) | Collapse to flags per date + location before joining |
| 7 | Missing store-days in transactions | ~10 per store (closures); 6 outlier stores | Store-specific calendars; missing ≠ 0 |
| 8 | Date coverage across tables | All signals cover the sales period | Holidays usable ahead; oil/transactions lagged only |

---

## Output tables

| Table | Grain | Rows | Contents |
|---|---|---|---|
| `demand_daily` | item × store × day | 7,897,788 | Sales (raw + clean), promotion, holiday/event flags, oil (forward-filled), store transactions, weather, item and store attributes |
| `demand_weekly` | item × store × week | 1,136,349 | Weekly sales, promo days, holiday/event days, `days_in_week` flag for partial weeks |

Row counts and unit totals were reconciled at every build step (no fan-out, no lost units).

---

## Key design decisions

- **Calendar spine:** every item × store pair gets a row for every day from its own first sale — 24% of days are zero-sales days that do not exist in the raw data. Without them, models would overestimate demand.
- **No zeros before an item or store existed:** calendars start at each pair's first sale.
- **Target leakage avoided:** the promo flag is set to FALSE on filled zero-sales days; leaving it NULL would let the model infer "no sale" from the missing flag.
- **Point-in-time thinking:** signals are split into *known in advance* (holidays, promotions, calendar) and *observed only* (oil, transactions, weather). The base tables store as-of-day facts; horizon-specific lags are built in the modelling phase.
- **Missing ≠ zero:** missing store transactions stay NULL (closure vs data gap tested against sales).
- **Raw data never overwritten:** cleaned values live in separate columns.

---

## Limitations

- Sales are **net** units (sales − returns); gross sales and returns are not available separately.
- **No inventory or stockout data** — a zero can mean no demand or an empty shelf (censored demand).
- **No price data** — only a yes/no promotion flag; price elasticity is not modelled.
- Weather and oil use **actuals**; production would use weather forecasts and oil futures, trained on point-in-time history.
- Weather is reanalysis at **city-grid level**, not exact store location.
- Data ends August 2017.

## Next steps (parked)

- Intermittent-demand segmentation (ADI / CV²) in FVA results
- New product forecasting (items with little history)
- Extend scope to Grocery I
- Production signal upgrades: weather forecasts, oil futures

---

## How to reproduce

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m jupyterlab
```

1. Download the Kaggle Favorita files into `data/raw/`.
2. Run notebooks `00` → `04` in order (one at a time — DuckDB allows one connection per database file).
3. `03_weather` downloads weather from Open-Meteo (internet required, no API key).

**Tech:** Python 3.13, DuckDB, JupyterLab, pandas, Open-Meteo API.
