# FCPI: Fraud Complaint Priority Index

**Supplementary Code for Anonymous Submission**

> This repository contains the full implementation of the **Fraud Complaint Priority Index (FCPI)**, an intervention-aware, unsupervised prioritization framework for cyber-enabled investment fraud complaints under partial observability. The code reproduces all experiments, figures, and tables reported in the paper.

---

## Repository Contents

```
├── FCPI_Final.ipynb            # Main notebook — all core experiments
└── README.md                        # This file
```

---

## Environment

**Python version:** 3.9 or higher

### Install dependencies

```bash
pip install pandas numpy scipy scikit-learn matplotlib seaborn lightgbm shap openpyxl
```

All other imports (`re`, `warnings`, `collections`, `subprocess`) are part of the Python standard library.

| Package | Purpose |
|---------|---------|
| `pandas`, `numpy` | Data loading and manipulation |
| `scipy` | Spearman correlation, Mann-Whitney U, statistical tests |
| `scikit-learn` | Gradient Boosting, Isolation Forest, ROC-AUC, cross-validation |
| `matplotlib`, `seaborn` | All figures and visualizations |
| `lightgbm` | Surrogate model for SHAP explainability |
| `shap` | Feature importance and case-level explanations |
| `openpyxl` | Reading `.xlsx` data files |

---

## Data

The experiments are conducted on a real-world anonymized dataset of **19,480 investment fraud complaints** provided by a state-level law enforcement cybercrime reporting system. Due to privacy and data-sharing restrictions, the dataset **cannot be released** as part of this submission.

### Expected input format

The notebook expects a single tabular file (`.xlsx` or `.csv`) with the following columns. Column names are configurable at the top of the preprocessing cell.

| Column | Type | Description |
|--------|------|-------------|
| `SI_No` | int | Unique complaint identifier |
| `Acknowledgement_No` | str | Secondary identifier |
| `District`, `Police_Station` | str | Jurisdictional fields |
| `Name_of_Victim` | str | Victim name (not used in scoring) |
| `Gender` | str | Victim gender |
| `Age` | numeric | Victim age |
| `Age_Group` | str | Derived age bracket (e.g., `18-30`) |
| `Profession` | str | Victim profession |
| `Final_Amount_Loss` | numeric | Reported financial loss (INR) |
| `POH` | numeric | Payment-on-Hold amount (intervention feasibility proxy) |
| `Suspect_Platform` | str | Raw platform label (may be `UNKNOWN`) |
| `Suspect_URL_Website` | str | Associated URL (used for platform extraction) |
| `Minor_Head` | str | Fraud subcategory |
| `English_Translation` | str | LLM-normalized complaint narrative |

To use the notebook with your own data, update the `DATA_PATH` variable in **Cell 4** and verify that column names match (or adjust the preprocessing cell accordingly).

---

## Notebook Walkthrough

### `FCPI_v3_Updated.ipynb` — Main Experiments

Run cells sequentially from top to bottom. Each section is self-contained with a markdown header.

| Cell(s) | Section | Description |
|---------|---------|-------------|
| 2 | Imports | All library imports |
| 4 | Data Loading | Load `.xlsx` or `.csv`; set `DATA_PATH` here |
| 6 | Missing Value Analysis | Coverage report for all columns |
| 8 | Platform Extraction | Regex-based narrative extraction; builds `Platform_Enriched` and `Platform_Tier` |
| 10 | Multi-Platform Analysis | Counts co-occurring platforms; Mann-Whitney test |
| 12 | Preprocessing | Builds the main `data` DataFrame with all cleaned fields |
| 14 | EDA | Six-panel exploratory figure (loss distribution, platform breakdown, demographics) |
| 16 | Component Functions | **All FCPI scoring functions defined here** — `compute_financial_severity`, `compute_platform_risk_v2`, `compute_demographic_v2`, `compute_poh_severity_v2`, `entropy_weights`, `coverage_rankic_weights`, `compute_fcpi` |
| 18 | Compute Components | Produces `S_financial`, `S_platform`, `S_demographic`, `S_poh` columns |
| 20 | Component Diagnostics | Table 3 in the paper: mean, std, IQR, % zero, Spearman ρ per component |
| 22 | Weighting | Entropy (v1) vs Rank-IC (v2) weight comparison; Table 2 in the paper |
| 24 | FCPI Scores | Computes `FCPI`, `FCPI_entropy`, `FCPI_equal` columns |
| 26 | Baselines | Gradient Boosting, Isolation Forest, Rule-Based scores |
| 28 | Evaluation Table | Table 5 in the paper: Spearman ρ, AUC, Precision@10%, Top-10% Mean Loss |
| 30 | Top-K Performance | Table 4 and Figure 5 in the paper |
| 32 | Monotonicity Check | FCPI score vs loss quartile (sanity check) |
| 34 | Outlier Stability | P99 trim robustness test |
| 36 | Ablation Study | Table 6: component-exclusion ablation |
| 38 | Monte Carlo Sensitivity | Weight perturbation and feature sensitivity analysis |
| 40 | Behavioral Features | Keyword extraction for `S_coercion`, `S_persistence`, `S_vuln` |
| 42 | FCPI⁺ Computation | Extended scoring with behavioral signals and financial floor constraint |
| 44 | FCPI⁺ Validation | Table 7: FCPI vs FCPI⁺ performance |
| 46–49 | SHAP Explainability | LightGBM surrogate + global SHAP importance (Figure 8, Table 8) |
| 51 | SHAP Waterfall | Case-level explanation for top-ranked complaint (Figure 9) |
| 53 | Top-5 Case Explanations | Table 9: case-level SHAP drivers |
| 55 | Comprehensive Report | Console summary of all key results |
| 57 | Circularity Analysis | Figure 6: FCPI performance without S_financial |
| 59 | Temporal Generalization | Figure 7: out-of-time validation |
| 61 | Scatter / Calibration | Figure 4: FCPI vs loss scatter and rank-concordance heatmap |
| 63 | Export | Saves prioritized complaint list to CSV |


## Key Design Decisions

**Rank-IC Weighting.**  
Component weights are estimated via Spearman rank correlation with financial loss, adjusted for sparsity (`weight_i = |ρ_i| × (1 − pct_zero_i)`). This is implemented in the `coverage_rankic_weights` function (Cell 16) and produces the final weights: Financial (0.531), POH (0.264), Platform (0.109), Demographic (0.096).

**Bayesian Smoothing.**  
Both platform risk and demographic susceptibility use Bayesian-smoothed conditional probability estimates to prevent overconfident estimates from small groups. The smoothing strength (`alpha`) is set in the component functions and can be adjusted.

**Financial Floor Constraint (FCPI⁺).**  
Behavioral signals are capped at 30% of total weight and cannot elevate cases below the 25th percentile loss threshold (`floor_pct=25`, `floor_penalty=0.15`). These are configurable parameters in `compute_fcpi_plus_v2`.

**Platform Tier Priority.**  
When multiple platforms are detected in a narrative, the highest-risk tier is assigned. Tier ordering from highest to lowest: `HIGH > SOCIAL > MESSAGING > APP > PAYMENT > UNKNOWN`.

---

## Reproducing Paper Results

All reported numbers are produced by running the main notebook end-to-end. Key output locations:

| Paper Result | Notebook Location |
|-------------|-------------------|
| Table 2 (Weighting Schemes) | Cell 22 |
| Table 3 (Component Diagnostics) | Cell 20 |
| Table 4 (Top-K Mean Loss) | Cell 28–30 |
| Table 5 (Method Comparison) | Cell 28 |
| Table 6 (Ablation) | Cell 36 |
| Table 7 (FCPI vs FCPI⁺) | Cell 44 |
| Table 8 (SHAP Importance) | Cell 47 |
| Table 9 (Case Explanations) | Cell 53 |
| Figure 2 (Loss Distribution) | Cell 14 |
| Figure 3 (Platform Enrichment) | Cell 8–10 |
| Figure 4 (Scatter / Calibration) | Cell 61 |
| Figure 5 (Top-K Performance) | Cell 30 |
| Figure 6 (Circularity Analysis) | Cell 57 |
| Figure 7 (Temporal Generalization) | Cell 59 |
| Figure 8 (SHAP Global) | Cell 49 |
| Figure 9 (SHAP Waterfall) | Cell 51 |
| Spearman ρ = 0.9211 | Cell 22 or 28 |
| AUC = 0.9558 | Cell 28 |
| P@10% = 0.9959 | Cell 28 |

---

## Configuration Reference

The following variables can be adjusted without modifying core logic:

```python
# Cell 4 — Data path
DATA_PATH = 'fraud_data_FINAL_FIXED (1).xlsx'

# Cell 16 — Financial severity
lambda_ = 0.6          # blend: 0=pure ECDF, 1=pure log-norm
clip_pct = (1, 99.5)   # percentile clip for log-norm normalization

# Cell 16 — Platform risk
high_loss_pct = 75     # threshold for "high loss" label
min_support   = 20     # minimum cases for platform-specific estimate
alpha         = 2.0    # Bayesian smoothing strength

# Cell 16 — Demographic susceptibility
min_sup = 30           # minimum support for hierarchical level
alpha   = 5.0          # Bayesian smoothing strength

# Cell 26 — Evaluation
TOP_K = 0.10           # default top-K fraction for metrics

# Cell 42 — FCPI+
behav_cap     = 0.30   # max weight fraction for behavioral components
floor_pct     = 25     # percentile below which financial floor applies
floor_penalty = 0.15   # score penalty for below-floor cases
```

---

## Output Files

Running the full notebook produces the following files in the working directory:

| File | Contents |
|------|---------|
| `final_results_fcpi.csv` | Evaluation table (Spearman ρ, AUC, P@K, Top-K loss) for all methods |
| `fcpi_prioritized_output.csv` | Full dataset with all component scores and final FCPI/FCPI⁺ ranks |
| `*.png` | All figures (named to match paper figure numbers) |

---

## Notes on Anonymization

- All complaint identifiers, victim names, and location fields are included in the dataset schema but are **not used** in any scoring computation.
- The dataset itself is not included in this submission. The code operates on the schema described above and can be adapted to similarly structured complaint datasets.
- No external API calls, web requests, or proprietary services are used. The notebook runs entirely offline.

---

## License

Code released under the MIT License for research reproducibility purposes.
