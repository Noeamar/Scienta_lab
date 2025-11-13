### RA–SLE–Healthy Classification from RNA-seq Data for Scienta Lab

This repository contains my solution to the task of predicting patient diagnosis (Healthy vs RA vs SLE) from RNA-seq data and identifying the most relevant genes driving each condition.

⸻

1. Data Preprocessing

The three cohorts did not measure exactly the same genes, so I restricted the analysis to the intersection of shared genes to keep a consistent feature space.
I also checked for missing values (none were found, but I kept an imputation fallback), removed zero-variance genes, and computed a cached correlation matrix to filter out highly correlated genes (threshold = 0.7).
This reduced the feature space from ~23,000 genes to ~8,000, which made the models faster to train.

I extracted patient/group identifiers and used StratifiedGroupKFold to avoid data leakage between samples coming from the same individual or cohort.

⸻

2. Baseline Models

Before applying feature selection, I trained three baseline models:
	•	Logistic Regression
	•	Random Forest
	•	XGBoost

Since I hadn’t worked much with multiclass medical prediction before, I looked into which metrics to use. I chose:
	•	Accuracy
	•	Macro-F1
	•	Macro ROC AUC
	•	Macro PR AUC

because they treat all classes equally and avoid biases toward the most represented one.

These baselines serve as reference scores.

⸻

3. Stable Gene Selection (STABL)

For feature selection, I used STABL, the method I contributed to during my internship at Stanford.
Although STABL supports several base models, I only ran STABL-RF here because Random Forests parallelize well and were the only variant I could run fully within the time constraints.

I applied STABL in a One-vs-Rest setup (Healthy vs Rest, RA vs Rest, SLE vs Rest) to extract class-specific stable genes.

⸻

4. Final Multiclass Classifier

After merging the genes selected by STABL across the three OvR tasks, I trained a final multiclass XGBoost classifier on the reduced feature set.
The test set (Healthy + RA + SLE) was kept fully held out.

The final evaluation reports:
	•	Accuracy
	•	Macro-F1
	•	Macro ROC AUC
	•	Macro PR AUC
	•	Confusion matrix

along with gene importances for interpretation.

⸻

5. Results (to complete)
	•	Baseline performance
	•	STABL gene lists
	•	Final XGBoost performance
	•	Gene interpretation

⸻

6. Running the Code

pip install -r requirements.txt
jupyter notebook homework.ipynb

7. Notes

This project focuses mainly on preprocessing, stability-based gene selection, and building a clean ML pipeline. If I had more time, I would test union-based gene sets with imputation, other multiclass feature-selection strategies, and alternative evaluation metrics.