# RNA-seq Multiclass Disease Classification (Healthy vs RA vs SLE)

This project explores how to predict patient diagnosis (Healthy / Rheumatoid Arthritis / Systemic Lupus Erythematosus) from RNA-seq data, and how to identify the most informative genes driving each condition.  
The pipeline covers data cleaning, feature filtering, baseline multiclass models, sparse feature selection (STABL), and biological interpretation.

---

## 1. Data Loading & Preprocessing

I used the 6 datasets provided (train/test for Healthy, RA, SLE).  
Each table contains transcriptomic measurements for thousands of genes.

### Preprocessing steps
- **Kept only common genes** across the three diseases to avoid dataset-specific artifacts.
- **Checked for missing values** — there were none, but I left the imputation logic in case it’s needed.
- **Removed zero-variance features**, which are uninformative.
- **Computed and cached a correlation matrix** to avoid recomputing it each time (it was large and slow).
- **Removed correlated genes above 0.7**, which reduced dimensionality from ~23,000 to ~8,500 genes.
  - This makes the models faster and helps avoid redundant signals.
- Extracted **GroupID** from sample names to avoid *data leakage* (same patient across train/val splits).

These steps gave a cleaner transcriptomic matrix suitable for feature selection and modeling.

---

## 2. Baseline Multiclass Models (Before Feature Selection)

I first trained simple multiclass models on the full filtered dataset:

- **Multinomial Logistic Regression**
- **Random Forest**
- **XGBoost**

Using **StratifiedGroupKFold (10 folds)** to avoid patient overlap.

**Metrics used:**  
- Accuracy  
- Macro-F1  
- Macro ROC-AUC  
- Macro PR-AUC  

I hadn’t worked much on multiclass medical prediction before, so I looked into which metrics matter.  
Macro averaging makes sense here because we want to evaluate all three diseases equally, not just the most represented ones.

**Baseline results (averaged over folds):**

| Model | Accuracy | F1-macro | ROC-AUC-macro | PR-AUC-macro |
|------|----------|----------|----------------|----------------|
| Logistic Regression | 0.592 | 0.685 | 0.804 | 0.768 |
| Random Forest       | 0.542 | 0.646 | 0.754 | 0.713 |
| XGBoost             | 0.510 | 0.624 | 0.778 | 0.735 |

These serve as a reference to compare feature-selection approaches.

---

## 3. Baseline Multiclass Feature Selection (Top-K curves)

To build a simple reference for feature selection, I trained:

- multinomial Logistic Regression  
- Random Forest  
- XGBoost  

on the full dataset, ranked genes by:
- absolute coefficient magnitude (LogReg)
- feature importance (RF / XGB)

Then I evaluated a **“top-K features” curve** for K in {10, 20, 50, 100, 150, 200}.

This gives a straightforward multiclass baseline to compare against STABL.

Observed pattern:
- Logistic Regression performs surprisingly well with **50–100 genes**.  
- Random Forest and XGBoost benefit from more features (≈150–200).  
- Performance drops sharply when K is too small (10–20).

---

## 4. STABL Feature Selection (One-vs-Rest)

I used **STABL**, the stability-based method I contributed to during my internship at Stanford.  
Originally STABL was for linear models, but part of my work involved extending it to **tree-based methods**, which is exactly what I used here (STABL-RF).

Why STABL?
- It selects **stable and reproducible** genes across many bootstraps.
- It avoids chasing noisy or dataset-specific markers.
- Works particularly well in **high-dimensional omics**.

Limitations:
- The current STABL pipeline is designed for **binary** classification.
- I didn’t have time to fully adapt it for three classes.
- So I used a **one-vs-rest** setup (Healthy vs Rest, RA vs Rest, SLE vs Rest).
- I could only run **STABL-RF**, because STABL-Lasso / EN / XGB take significantly longer.

STABL-RF produced ≈85 stable genes across the three comparisons.

---

## 5. Final Multiclass Model (Using STABL Genes Only)

Using the union of 85 STABL-selected genes, I trained a final XGBoost model:

**Results on the independent test set:**

| Metric | Score |
|--------|--------|
| Accuracy | **0.514** |
| Macro-F1 | **0.575** |
| ROC-AUC-macro | **0.791** |
| PR-AUC-macro | **0.729** |

### Per-Class Behavior
- **Healthy (0):** perfectly predicted (100/100).
- **RA (1):** strong recall (1.00) but moderate precision.
- **SLE (2):** most errors → many SLE predicted as RA.

Confusion matrix: