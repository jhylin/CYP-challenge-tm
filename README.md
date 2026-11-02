# Tsetlin Machine feature net regressors for OpenADMET CYP Blind Challenge

**Work-in-progress**

This repository provides predicted pIC50 values for four cytochrome P450 (CYP) isoforms using a **Tsetlin machine (TM) feature net** approach by training auxiliary TMs on three related open datasets - two provided by OpenADMET (auxiliary dataset A & B) and one provided by Peteani G. et al. from this [paper](https://doi.org/10.1038/s41467-024-49979-3) by using its supplementary data (auxiliary dataset C). The auxiliary TMs turn the predictions into extra Boolean inputs for a main TM to use to predict on a test set.

---

## Contents

- [Tsetlin Machine feature net regressors for OpenADMET CYP Blind Challenge](#tsetlin-machine-feature-net-regressors-for-openadmet-cyp-blind-challenge)
  - [Contents](#contents)
  - [Overview](#overview)
  - [Repository structure](#repository-structure)
  - [Installation](#installation)
  - [Data](#data)
  - [Method](#method)
  - [Configuration](#configuration)
  - [Running the notebook](#running-the-notebook)

---

## Overview

- **Task:** regression of pIC50 for 4 CYP isoforms: CYP3A4, CYP2D6, CYP2C9 and CYP1A2
- **Inputs:** ECFP fingerprints (radius = 3, size = 2048 bits) from canonical SMILES.
- **Model:** one `TMRegressor` (tmu) per isoform, trained on ECFP bits plus features produced by auxiliary TMs.
- **Auxiliary data:** three open datasets (A, B, C) of related measurements; see [Data](#data).
- **Baseline:** random forest on the same fingerprints and splits ([**TODO: notebook name**]).
- **Why a TM:** predictions come from human-readable AND rules over fingerprint bits and auxiliary features.

## Repository structure

```
.
├── notebooks/
│   ├── 10_from_scratch_tm_reg_featurenet.ipynb   # main notebook (feature net, aux A, B and C)
│   └── [**TODO: RF baseline notebook, if included**]
├── data/                                         # input CSVs (DATA_DIR)
│   ├── main_train.csv
│   ├── aux_A.csv
│   ├── aux_B.csv
│   ├── aux_C.csv
│   └── test.csv
├── outputs/                                      # created on first run, not tracked (OUTPUT_DIR)
├── pixi.toml / pixi.lock                         # pinned environment
├── LICENSE
└── README.md
```

## Installation

Requires [pixi](https://pixi.sh) and a C compiler (tmu builds a C extension).

```bash
git clone https://github.com/[TODO: owner]/[TODO: repo].git
cd [TODO: repo]
pixi install
```

Key dependencies are pinned in `pixi.lock` and `pixi.toml` files.

Quick check:

```bash
pixi run python -c "from tmu.models.regression.vanilla_regressor import TMRegressor; from tm_qsar_benchmark.descriptors import gen_ecfp_arr; print('OK')"
```

## Data

All datasets are open. **[TODO: confirm each licence allows redistribution; otherwise replace files with download instructions.]**

| File | Content | Molecules | Source | Licence |
| --- | --- | --- | --- | --- |
| `main_train.csv` | SMILES + pIC50 for 4 isoforms (wide; sparse) | [TODO] | [TODO] | [TODO] |
| `aux_A.csv` | SMILES + isoform + 2 lab values (long) | 4,376 | [TODO] | [TODO] |
| `aux_B.csv` | SMILES + 4 related measurements (wide) | 6,145 | [TODO] | [TODO] |
| `aux_C.csv` | SMILES + pIC50 for 3 of the 4 isoforms (wide) | 273,706 | [TODO] | [TODO] |
| `test.csv` | Name + SMILES (unlabelled) | [TODO] | [TODO] | [TODO] |

## Method

1. **Molecules**: SMILES canonicalised with RDKit; standardised via the ChEMBL structure pipeline (through `tm_qsar_benchmark`).
2. **Auxiliary TMs** (trained once, cached, all details in notebook provided):
   - **Aux A**: Single-concentration dataset from OpenADMET and Octant
   - **Aux B**: Emax dataset from OpenADMET and Octant
   - **Aux C**: Supplementary data from paper mentioned above
3. **Aux features**: details in notebook provided
4. **Leakage control**: molecules shared between auxiliary data and main/test receive out-of-fold predictions (k = 5).
5. **Main TMs**: one `TMRegressor` per isoform on ECFP bits + aux features (scope: all isoforms).
6. **Final model**: retrained on all labelled main data (`VAL_FRACTION = 0`) for test predictions.

## Configuration

Final settings used for the reported results (set in featurenet_tm_regressor notebook's configuration cell).

## Running the notebook

Paths are relative (`DATA_DIR = "../data"`, `OUTPUT_DIR = "../outputs"`), so the repository runs from any location.

**Interactively**: open `notebooks/featurenet_tm_regressor.ipynb` in Jupyter and run all cells.

**Headless**: (recommended for long runs; use `tmux` so a dropped connection doesn't stop it):

```bash
pixi run papermill notebooks/featurenet_tm_regressor.ipynb outputs/run.ipynb \
  --cwd notebooks --log-output 2>&1 | tee outputs/run.log
```

**Outputs** (in `outputs/`):

- `featurenet_3aux_aux_cache_epo50.pkl`: trained aux TMs and features. Reused automatically when aux settings and data are unchanged. This is retrained otherwise.
- `test_predictions_final_1_full_set.csv`: `pred_pIC50_<isoform>` and `pred_active_<isoform>` per test molecule. Note: `pred_active_<isoform>` column is for classifying for actives or inactives, for regression outputs, use the other column instead.
- `featurenet_3aux_pipeline.pkl`: full pipeline to be saved if needed (`SAVE_PIPELINE = True`).

**Quick smoke test**: set all epochs to 1 and `AUX_CACHE = None`, then run.

**Acknowledgement**: