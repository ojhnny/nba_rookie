# 🏀 NBA Draft Value Model

Predicting NBA career impact from **pre-draft information only** — draft position, college performance, and NBA Combine measurements — to find the players who most exceeded (and fell short of) what their pre-draft profile suggested.

The guiding question isn't *"who is good?"* It's *"who beat expectations?"* — the difference between a scout and an analyst. The biggest overperformer in the dataset is **Nikola Jokić**, a 41st pick with modest college numbers and no combine data: a profile that gave any model almost nothing to work with.

## 📊 Live Dashboard

[Explore the interactive Tableau dashboard →](https://public.tableau.com/app/profile/johnny.lin2176/viz/nba_dashboard_rookie/ModelInsights)

---

## 🎯 Project Overview

An end-to-end analytics project:

1. **Data engineering** — scrape and merge draft, combine, and college data from three sources into one clean modeling table
2. **Modeling** — train a Random Forest to predict a runway-neutral measure of NBA career impact, validated honestly
3. **Visualization** — surface over/underperformers and draft-class trends in an interactive Tableau dashboard

The project is deliberately built to avoid the two methodological traps that quietly invalidate most "predict the draft" projects (see [Methodology](#methodology-what-makes-this-rigorous)).

---

## 📚 Data Sources

| Source | What it provides | Coverage |
|---|---|---|
| [Basketball Reference](https://www.basketball-reference.com/draft/) | Draft picks + NBA career stats (BPM, VORP, WS, per-game) | 2010–2025 |
| [Kaggle - NBA Draft Combine & Player Info + "Drafted?" Data](https://www.kaggle.com/datasets/natelapin/nba-draft-combine-and-player-info-drafted-data) | Athletic testing — vertical, sprint, lane agility, bench, measurements | 2000–2024 |
| [toRvik](https://github.com/andreweatherman/toRvik-data) | College advanced, box-score, and shooting stats | 2014–2023 |

Records were matched across sources by **normalized player name** (accent-stripped, suffix-removed) and draft year. Combine data matched ~69% of draft picks; college data ~48% (the gap is mostly international players and pre-2014 picks). Unmatched players are retained with missing-data flags rather than dropped.

---

## 🔬 Methodology (What Makes This Rigorous)

Three deliberate decisions separate this from a typical draft model:

**1. No data leakage.** NBA career statistics are *never* used as model inputs — only information available before a player is drafted. Using career stats to "predict" career outcomes is the most common silent bug in projects like this.

**2. Runway-neutral target.** A naive impact score built from career totals (Win Shares, VORP) is badly confounded by *how long a player has been in the league*. Among players with ≥50 career games:

| Target quantity | Correlation with career length (`Yrs`) |
|---|---|
| Career-total impact (WS + VORP z-avg) | **0.69** |
| Win Shares (total) | **0.77** |
| VORP (total) | 0.58 |
| BPM (rate) | 0.51 |
| WS/48 (rate) | 0.43 |

A totals-based model partly measures *time*, not *talent*. The fix: predict a **rate-based** score — the z-scored average of **BPM** and **WS/48** (`rate_impact`) — apply a **50-game minimum** so rates are stable, and **exclude `DraftYear`** as a feature so the model can't relearn draft age as a proxy.

**3. Honest evaluation.** Over/underperformer labels come from **out-of-fold predictions** (`cross_val_predict`) — every player is scored by a model that never trained on them. Performance is reported with 5-fold cross-validation *and* a **time-based split** (train ≤ 2018, test ≥ 2019) to measure how the model generalizes to future draft classes, which is how it would actually be used.

---

## 💡 Key Findings

**Model performance (691 qualified players, ≥50 career games):**

| Check | R² | MAE |
|---|---|---|
| Ridge (baseline), 5-fold CV | 0.129 | 0.691 |
| Random Forest, 5-fold CV | 0.115 | 0.705 |
| Random Forest, time-based split (train ≤2018 / test ≥2019) | 0.107 | 0.715 |

- **Pre-draft data explains only a modest share of career impact** — and that's the honest result. Work ethic, injuries, coaching, and team fit are invisible before the draft.
- **Random Forest does not beat a Ridge baseline** on this data. The relationships look close to linear; RF is kept mainly for feature-importance interpretability, not predictive lift.
- **Per-fold RF R² is unstable** (`[0.18, 0.053, 0.035, 0.172, 0.133]`), so treat the headline score as a rough average, not a precise repeatable number.
- **Top feature-importance signals** (Random Forest): `combine_weight_lbs` (~18%) > `Pk` (~15%) > `college_dbpm` (~13%). No single feature dominates — the model spreads weight across many weak signals.
- **The residuals are the payoff.** Players like Jokić, Giannis, Kawhi, and SGA emerge as the largest overperformers — exactly the late-round and overlooked picks that good analytics is meant to flag.

**Over/underperformer labels** use OOF residuals with a ±1 SD threshold (~±0.90 on `impact_vs_prediction`): 97 overperformers, 103 underperformers, 491 near prediction among qualified players.

---

## 📁 Repository Structure

```
nba_rookie-main/
├── data_clean.ipynb              # Scrape + merge draft, combine, and college data
├── predictive_model.ipynb        # Target design, modeling, validation, Tableau export
├── nba_dashboard_model_filter_tabs_fixed.twbx   # Tableau dashboard
├── README.md
├── tableau_nba_draft_master.csv              # Master table (also in data/)
├── tableau_nba_draft_model_predictions.csv   # Model output (also in data/)
└── data/
    ├── NBA_Draft_Combine_2000_2024_nl.csv    # Raw Kaggle combine file (input)
    ├── nba_draft_data.csv                    # Scraped draft history
    ├── combine_data.csv                      # Cleaned combine data
    ├── college_advanced_2014_2023.csv        # toRvik advanced stats
    ├── college_box_2014_2023.csv             # toRvik box-score stats
    ├── college_shooting_2014_2023.csv        # toRvik shooting stats
    ├── college_stats_2014_2023.csv            # Merged college table
    ├── tableau_nba_draft_master.csv          # Final merged modeling table
    └── tableau_nba_draft_model_predictions.csv  # Predictions + labels for Tableau
```

Some intermediate CSVs are written to the **project root** when you run `data_clean.ipynb`; copies also live under `data/`. The repo ships with both so you can explore without re-scraping.

---

## 📄 File Guide — What Each File Is For

| File | Created by | Purpose |
|---|---|---|
| `data/NBA_Draft_Combine_2000_2024_nl.csv` | Manual download (Kaggle) | Raw combine input — required before running `data_clean.ipynb` |
| `nba_draft_data.csv` | `data_clean.ipynb` | Draft picks + NBA career stats from Basketball Reference (2010–2025) |
| `combine_data.csv` | `data_clean.ipynb` | Cleaned combine measurements with normalized player names |
| `college_advanced_2014_2023.csv` | `data_clean.ipynb` | College BPM, porpag, efficiency metrics from toRvik |
| `college_box_2014_2023.csv` | `data_clean.ipynb` | College counting stats (ppg, rpg, apg, etc.) |
| `college_shooting_2014_2023.csv` | `data_clean.ipynb` | College usage, eFG%, three-point rate, etc. |
| `college_stats_2014_2023.csv` | `data_clean.ipynb` | Single merged college table used for player matching |
| `tableau_nba_draft_master.csv` | `data_clean.ipynb` | **Main dataset** — draft + combine + college + NBA outcomes + `has_*` flags |
| `tableau_nba_draft_model_predictions.csv` | `predictive_model.ipynb` | Master table plus `rate_impact`, OOF predictions, residuals, and `model_result` labels |
| `nba_dashboard_model_filter_tabs_fixed.twbx` | Tableau | Interactive dashboard — connect to the predictions CSV |

**Key columns in the master table:**

- `Player_clean` / `uniqueID` — normalized name and Basketball Reference player ID for matching
- `has_combine_data` / `has_college_stats` — boolean flags for which pre-draft sources matched
- `college_year_gap` — seasons between final college year and NBA draft (useful age/experience signal)
- NBA career columns (`BPM`, `WS/48`, `G`, etc.) — used only as **targets** and EDA, never as model inputs

**Key columns added by the model notebook:**

- `rate_impact` — z-scored average of BPM and WS/48 (the target)
- `predicted_rate_impact` — out-of-fold model prediction
- `impact_vs_prediction` — residual (actual minus predicted)
- `model_result` — `Overperformed Prediction`, `Underperformed Prediction`, `Near Prediction`, or `Insufficient NBA sample`

---

## 🚀 How to Run

**Requirements:** Python 3.10+ with:

```bash
pip install pandas numpy scikit-learn matplotlib lxml pyarrow
```

**Important:** Launch Jupyter from the **project root** (`nba_rookie-main/`), not from `data/`. The notebooks use a mix of root-level and `data/`-prefixed paths.

### 🔄 Option A — Full rebuild (needs internet)

1. Place the Kaggle combine file at `data/NBA_Draft_Combine_2000_2024_nl.csv` (already included in this repo).
2. Start Jupyter from the project root:
   ```bash
   cd nba_rookie-main
   jupyter lab
   ```
3. Run **`data_clean.ipynb`** top to bottom. This scrapes Basketball Reference, pulls toRvik parquet files from GitHub, merges everything, and writes `tableau_nba_draft_master.csv` (project root). Copy or sync it to `data/` if you want both locations in sync:
   ```bash
   copy tableau_nba_draft_master.csv data\
   ```
4. Run **`predictive_model.ipynb`** top to bottom. The main modeling cells read `tableau_nba_draft_master.csv` from the project root and export `tableau_nba_draft_model_predictions.csv` there. Copy to `data/` for the dashboard if needed:
   ```bash
   copy tableau_nba_draft_model_predictions.csv data\
   ```
5. Open **`nba_dashboard_model_filter_tabs_fixed.twbx`** in Tableau Desktop or Tableau Public. If the data source path is broken, point it at `tableau_nba_draft_model_predictions.csv` in this folder.

### ⚡ Option B — Explore without re-scraping

If you only want to review results or tweak the model:

1. Use the pre-built `data/tableau_nba_draft_master.csv` (or the copy in the project root).
2. Run **`predictive_model.ipynb`** from the project root. Ensure `tableau_nba_draft_master.csv` exists in the root, or update the `read_csv` path in the notebook to `data/tableau_nba_draft_master.csv`.
3. Open the Tableau workbook.

### 📓 Notebook walkthrough

| Notebook | What it does | Main output |
|---|---|---|
| `data_clean.ipynb` | Scrape drafts → clean combine → pull college stats → match players → export master table | `tableau_nba_draft_master.csv` |
| `predictive_model.ipynb` | Define `rate_impact` → select pre-draft features → Ridge + RF CV → feature importance → OOF labels → export | `tableau_nba_draft_model_predictions.csv` |

**`data_clean.ipynb` pipeline (in order):**

1. Scrape 2010–2025 draft tables from Basketball Reference; drop blank/forfeited pick rows
2. Load and clean Kaggle combine data; filter to draft years 2010–2025
3. Download toRvik college parquet files (advanced, box, shooting) for 2014–2023
4. Match college seasons to draft picks by normalized name, draft year, and pick disambiguation
5. Left-join combine and college onto draft data; add `has_combine_data` / `has_college_stats` flags
6. Export `tableau_nba_draft_master.csv`

**`predictive_model.ipynb` pipeline (in order):**

1. EDA — show why rate stats beat totals for the target
2. Build `rate_impact` on players with ≥50 career games (~691 players)
3. Train Random Forest (with `SimpleImputer`) on pre-draft features only — no `DraftYear`, no NBA outcome columns
4. Compare against Ridge baseline; run 5-fold CV and a time-based holdout
5. Plot feature importance and OOF actual vs predicted scatter
6. Label over/underperformers from OOF residuals (±1 SD); export for Tableau
7. Optional: draft-class summaries and single-player lookup cells at the end

---

## 📈 Using the Tableau Dashboard

1. Open `nba_dashboard_model_filter_tabs_fixed.twbx`.
2. Confirm the data source points to `tableau_nba_draft_model_predictions.csv` (update the path if you moved the file).
3. Use the filter tabs to slice by draft year, combine availability, college coverage, or model result.
4. **`model_result`** is the primary storytelling field — it flags who beat or missed their pre-draft profile, not simply who had the best career.
5. Compare `rate_impact` (actual) vs `predicted_rate_impact` (expectation) to see how conservative the model is — predictions cluster near zero while stars and busts spread wider.

---

## 🛠️ Troubleshooting

| Issue | Fix |
|---|---|
| `FileNotFoundError` for `tableau_nba_draft_master.csv` | Run `data_clean.ipynb` first, or copy the file from `data/` to the project root |
| `ArrowKeyError: pandas.period already defined` when reading toRvik parquet | Restart the Jupyter kernel; upgrade `pandas` and `pyarrow`; or skip the scrape and use existing college CSVs |
| `KeyError: 'has_combine_data'` during EDA | That column exists on the final `tableau_data` / exported CSV, not on mid-pipeline `master` objects — load `tableau_nba_draft_master.csv` instead |
| `TypeError` comparing `G` (games) | Coerce first: `df["G"] = pd.to_numeric(df["G"], errors="coerce")` |
| Combine `Position` is null | Expected for picks with no combine match (left join) — use `has_combine_data` or non-null `uniqueID` as the match indicator |
| Very recent draft picks lack labels | Players under 50 career games get `Insufficient NBA sample`, not a fabricated residual |

---

## 🧰 Tech Stack

**Python** (pandas, scikit-learn, matplotlib) · **Jupyter** · **Tableau** · web scraping via `pandas.read_html` · Parquet via `pyarrow`

---

## 🔮 Limitations & Next Steps

- **Fixed-window target (gold standard).** The rate-based target de-biases career length but doesn't fully eliminate survivorship. The cleanest fix is a true **first-3-year** or **first-5-year** impact window, which requires scraping season-by-season NBA logs from Basketball Reference — the highest-value next extension.
- **College coverage starts in 2014**, so pre-2014 picks have combine + draft data but no college features.
- **Composite weighting.** BPM and WS/48 are equally weighted after standardizing; a defensible alternative is to weight by minutes played or validate the weighting against an external value metric.
- **Modest R² is a feature, not a bug.** The model is best used as an exploratory over/underperformer flagger, not a precise draft-value predictor.
