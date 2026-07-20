# Endocrine–metabolic network architecture in polycystic ovary syndrome

Reproducible Python/Google Colab pipeline accompanying the manuscript:

> **Endocrine–metabolic network architecture reveals key bridge biomarkers in polycystic ovary syndrome**  
> Natalia Piórkowska, Alan Ostromęcki, Grzegorz Franik, Anna Bizoń

This repository reconstructs a biomarker network in women with polycystic ovary syndrome (PCOS), identifies central and inter-domain bridge biomarkers, and evaluates the stability of the inferred network through bootstrap resampling and predefined sensitivity analyses.

## Study overview

The analysis uses routinely measured laboratory biomarkers representing four prespecified physiological domains:

- **Endocrine**
- **Metabolic**
- **Hematological / inflammatory**
- **Thyroid**

The primary analysis is based on 29 directly measured biomarkers from 1,286 women with PCOS. Conditional dependencies are estimated with a sparse Gaussian graphical model using Graphical LASSO. The regularization parameter is selected by minimizing the Extended Bayesian Information Criterion, EBIC(γ = 0.5), subject to a prespecified network-density constraint of ≤ 0.20.

The primary reconstructed network contains 29 nodes and 73 edges, with a density of 0.18 and selected α = 0.125. The principal bridge biomarkers reported in the manuscript are SHBG, fasting insulin, triglycerides, and HDL cholesterol.

## Repository structure

```text
.
├── data_raw/
│   └── pcos_data_base_ml.xlsx        # not publicly distributed
├── scripts/
│   ├── 00_setup.ipynb
│   ├── 01_ingest_harmonize.ipynb
│   ├── 02_primary_network_dataset.ipynb
│   ├── 03_correlation_network.ipynb
│   ├── 04_primary_graphical_lasso.ipynb
│   ├── 04b_alpha_gamma_sensitivity.ipynb
│   ├── 04c_complete_case_network.ipynb
│   ├── 05_centrality_bridge.ipynb
│   ├── 06_bootstrap_stability.ipynb
│   ├── 07_secondary_full_clinical_network.ipynb
│   ├── 08_final_qc_manifest.ipynb
│   ├── 09_publication_readiness.ipynb
│   └── _pipeline_common.py
├── outputs/                         # generated automatically
└── README.md
```

The raw clinical dataset is not included because it contains sensitive patient information and is subject to institutional and ethical restrictions.

## Analytical workflow

Run the notebooks in the following order:

| Stage | Notebook | Purpose |
|---:|---|---|
| 00 | `00_setup.ipynb` | Freezes the analysis plan and random seed. |
| 01 | `01_ingest_harmonize.ipynb` | Loads the source dataset, filters the PCOS cohort, harmonizes raw laboratory names, and creates derived indices used only in sensitivity analysis. |
| 02 | `02_primary_network_dataset.ipynb` | Builds the 29-biomarker primary dataset; applies winsorization, conditional `log1p` transformation, median imputation, and z-score standardization. |
| 03 | `03_correlation_network.ipynb` | Creates a descriptive Spearman correlation network with Benjamini–Hochberg FDR correction. This network is not used for primary model selection. |
| 04 | `04_primary_graphical_lasso.ipynb` | Estimates the primary sparse partial-correlation network using Graphical LASSO and EBIC-based model selection with the density constraint. |
| 04b | `04b_alpha_gamma_sensitivity.ipynb` | Evaluates sensitivity to α, EBIC γ, and the density-selection rule; assesses bridge-rank stability around the selected model. |
| 04c | `04c_complete_case_network.ipynb` | Reconstructs the primary network in complete cases and compares topology and rankings with the median-imputed analysis. |
| 05 | `05_centrality_bridge.ipynb` | Computes node strength, degree, betweenness, bridge strength, and bridge degree. |
| 06 | `06_bootstrap_stability.ipynb` | Performs 500 nonparametric bootstrap resamples to evaluate edge and biomarker-ranking stability. |
| 07 | `07_secondary_full_clinical_network.ipynb` | Builds a secondary sensitivity network that includes deterministic derived indices. It is not used for primary biological inference. |
| 08 | `08_final_qc_manifest.ipynb` | Generates the publication summary and a repository-wide file manifest with file size and MD5 hashes. |
| 09 | `09_publication_readiness.ipynb` | Produces the final publication-readiness report and consolidates key cautions and limitations. |

Run the notebooks strictly in sequence during the first complete execution. Stages `04b` and `04c` branch from the primary network workflow, while all final reports assume that the required upstream outputs already exist.

## Installation

### Google Colab

The notebooks are designed to run directly in Google Colab. Clone the repository and change to the project directory:

```python
!git clone https://github.com/npiorkowska-science/pcos-network-architecture.git
%cd pcos-network-architecture
```

Place the source dataset at:

```text
data_raw/pcos_data_base_ml.xlsx
```

Then open and run the notebooks from `scripts/` in the order listed above. In Colab, the environment cells install the packages that may not already be available.

### Local Jupyter environment

Python 3.10 or newer is recommended.

```bash
git clone https://github.com/npiorkowska-science/pcos-network-architecture.git
cd pcos-network-architecture
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install numpy pandas scipy scikit-learn networkx statsmodels openpyxl matplotlib jupyter
jupyter lab
```

On Windows PowerShell, activate the environment with:

```powershell
.venv\Scripts\Activate.ps1
```

The notebooks locate the project root automatically by searching for directories named `scripts/` and `outputs/`.

## Input data requirements

The pipeline expects an Excel workbook named:

```text
data_raw/pcos_data_base_ml.xlsx
```

When a `group` column is present, rows are restricted to records where `group` equals `PCOS` after case normalization.

Raw laboratory variables may be stored under several alternative names. The predefined aliases are centralized in `scripts/_pipeline_common.py`. During harmonization, candidate columns are coalesced into standardized feature identifiers. Numeric parsing supports decimal commas, threshold expressions such as `<x` and `>x`, and simple numeric ranges.

No network-based or outcome-based feature selection is applied. The primary variables and their physiological domains are defined before network estimation.

## Preprocessing

The primary preprocessing sequence is:

1. Convert raw values to numeric format and replace invalid values with missing values.
2. Winsorize each variable at the 1st and 99th percentiles when at least 20 observations are available.
3. Apply `log1p` to non-negative variables with right skewness greater than 1.
4. Median-impute missing values for the primary analysis.
5. Standardize all variables using z-scores.

The complete-case analysis in stage `04c` is used as a prespecified sensitivity analysis and does not use imputation.

## Network estimation

For standardized data matrix **Z**, Graphical LASSO estimates a sparse precision matrix. Non-zero off-diagonal precision-matrix elements are converted to partial correlations and represented as weighted, undirected network edges.

Candidate α values are evaluated using:

```text
EBIC = -2n × log-likelihood + E × log(n) + 4γE × log(p)
```

where:

- `n` is the number of participants,
- `p` is the number of biomarkers,
- `E` is the number of non-zero undirected edges,
- `γ = 0.5` in the primary analysis.

The primary model is the minimum-EBIC solution among networks satisfying density ≤ 0.20. The secondary derived-index network is intentionally evaluated without this density constraint.

## Centrality and bridge metrics

The pipeline computes:

- **Degree:** number of edges connected to a biomarker.
- **Node strength:** sum of the absolute weights of all edges connected to a biomarker.
- **Betweenness centrality:** frequency with which a biomarker lies on shortest paths through the network.
- **Bridge degree:** number of edges connecting a biomarker to biomarkers from other physiological domains.
- **Bridge strength:** sum of the absolute weights of all inter-domain edges connected to a biomarker.

Bridge strength is the primary measure used to identify biomarkers integrating distinct physiological domains.

## Main generated outputs

Key files include:

```text
outputs/01_ingest_harmonize/harmonized_features.csv
outputs/01_ingest_harmonize/feature_mapping_report.csv
outputs/02_primary_network_dataset/primary_direct_features_z.csv
outputs/03_correlation_network/spearman_edges_fdr_q05_abs015_primary.csv
outputs/04_primary_graphical_lasso/primary_graphical_lasso_partial_edges.csv
outputs/04_primary_graphical_lasso/primary_partial_correlation_matrix.csv
outputs/04_primary_graphical_lasso/primary_graphical_lasso_model_summary.json
outputs/04_primary_graphical_lasso/Figure_primary_partial_network.png
outputs/04b_alpha_gamma_sensitivity/alpha_gamma_density_selection_grid.csv
outputs/04b_alpha_gamma_sensitivity/bridge_rank_sensitivity_by_alpha.csv
outputs/04c_complete_case_network/complete_case_vs_primary_comparison.json
outputs/05_centrality_bridge/Table_primary_centrality_bridge_metrics.csv
outputs/05_centrality_bridge/top_bridge_biomarkers_primary.csv
outputs/06_bootstrap_stability/edge_bootstrap_frequency_primary.csv
outputs/06_bootstrap_stability/centrality_bootstrap_summary_primary.csv
outputs/07_secondary_full_clinical_network/secondary_model_summary.json
outputs/08_final_qc_manifest/publication_results_summary_v3.json
outputs/08_final_qc_manifest/file_manifest_v3.csv
outputs/09_publication_readiness/publication_readiness_report.json
```

Each stage writes its outputs to a dedicated directory. Downstream notebooks read only previously generated files, supporting traceability from the raw dataset to the final manuscript-facing results.

## Reproducibility

All randomized procedures use:

```python
SEED = 42
```

The seed is defined in `_pipeline_common.py`. The primary Graphical LASSO solver is deterministic for a fixed dataset and software environment. Selected α, edge count, network density, and bridge rankings should reproduce exactly, although negligible floating-point differences may occur across platforms or package versions.

Stage `08_final_qc_manifest.ipynb` generates `file_manifest_v3.csv`, containing the relative path, file size, and MD5 checksum for every file present in the repository at execution time.

## Data availability

The clinical dataset is not publicly available because it contains sensitive patient information and unrestricted release is prohibited by institutional and ethical requirements. A de-identified dataset may be made available by the corresponding author upon reasonable request, subject to institutional and ethics approval.

## Ethics

The study was conducted in accordance with the Declaration of Helsinki and approved by the Bioethics Committee of Wroclaw Medical University, approval no. **254/2021**.

## Citation

A formal citation will be added after publication. Until then, cite the manuscript as:

```text
Piórkowska N, Ostromęcki A, Franik G, Bizoń A.
Endocrine–metabolic network architecture reveals key bridge biomarkers in polycystic ovary syndrome.
Manuscript under review.
```

Repository:

```text
https://github.com/npiorkowska-science/pcos-network-architecture
```

## License

No license has yet been specified. Until a license file is added, copyright remains with the authors and reuse beyond viewing or reproducing the published analyses requires permission.

## Contact

**Natalia Piórkowska, PhD**  
Faculty of Information and Communication Technology  
Wroclaw University of Science and Technology  
Email: `natalia.piorkowska@pwr.edu.pl`
