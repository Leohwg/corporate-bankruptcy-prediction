[README.md](https://github.com/user-attachments/files/33168997/README.md)
# Corporate Bankruptcy Prediction

Predicting whether a firm goes bankrupt within 12 months from three years of financial statements, using an ensemble of an Altman-inspired risk score, a logistic regression and a random forest.

Developed for the **International AI & Finance Project** prediction challenge, a joint initiative of the **University of Padova** and **Vilnius University**.

## Contents

- [The challenge](#the-challenge)
- [Data](#data)
- [Approach](#approach)
- [Results](#results)
- [Known limitations](#known-limitations)
- [How to run](#how-to-run)
- [Repository structure](#repository-structure)
- [Team](#team)
- [References](#references)

## The challenge

Teams from the two universities received a labelled training set and an unlabelled test set of European firms and had to predict a binary target:

| Target | Meaning |
|--------|---------|
| `1` | Bankruptcy within 12 months |
| `0` | No bankruptcy within 12 months |

Deliverables were a 12-slide presentation, a video of at most 5 minutes and an Excel file with the predicted labels. The score was weighted as follows:

| Component | Weight |
|-----------|--------|
| Model performance (accuracy 30%, precision 10%, recall 10%) | 50% |
| Slide presentation | 20% |
| Video and oral presentation (method, features, model choice, metrics) | 30% |

## Data

**The dataset is not included in this repository.** It was provided by the course organisers and is based on commercial financial data (Moody's Global Standard Format), so it cannot be redistributed.

What the data looks like:

- Firms from **Italy and Lithuania** with financial data for **2016-2018**, 2016 revenue between EUR 4 and 5 million, unconsolidated accounts and no consolidated companion.
- 30 income statement and balance sheet variables per year (90 in total), plus identifiers. Column names follow the pattern `<item> th EUR <year>`, for example `Total assets th EUR 2018`.
- **Training set:** 1,780 firms with the target. **Test set:** 446 firms without the target.
- Defaults are rare: about 8% of the training firms are labelled `1`.

To reproduce the results you need the three files supplied by the organisers: the training file, the test file and the submission template.

## Approach

The pipeline has four steps.

### 1. Feature engineering

From the raw statements the code builds 14 families of financial ratios for each year, covering:

- **Liquidity:** working capital to assets, cash to assets, current ratio.
- **Profitability:** EBIT to assets, net income to assets, retained funds to assets, gross margin.
- **Leverage:** total liabilities to assets, equity to assets, financial debt to assets.
- **Financial burden:** interest coverage, implied interest rate.
- **Efficiency:** asset turnover, payables to sales.

For every ratio it also computes trend features (change 2016-17, change 2017-18, two-year change, acceleration, standard deviation across the three years). On top of that come growth rates (revenue, assets) and red-flag indicators: negative equity, years of net loss, interest coverage below 1. In total this gives **123 engineered features**.

### 2. Feature selection

1. **Correlation filter:** drop one of every pair of features with absolute correlation above 0.85. This removes 59 features and leaves 64.
2. **Random forest importance:** fit a 200-tree forest on the survivors and keep the top 10.

The 10 selected features:

| # | Feature |
|---|---------|
| 1 | `op_profit_to_assets_2018` |
| 2 | `rev_growth_2yr` |
| 3 | `interest_coverage_mult_2018` |
| 4 | `total_loss_years` |
| 5 | `equity_deficit_18` |
| 6 | `rev_growth_1yr` |
| 7 | `current_solvency_2018` |
| 8 | `retained_funds_to_assets_2017` |
| 9 | `asset_growth_1yr` |
| 10 | `work_cap_to_assets_2018` |

### 3. Models

| Model | Description |
|-------|-------------|
| **Altman-inspired score** | Not the original 1968 Z-score with its fixed coefficients. Ratios are grouped into 9 economic blocks (liquidity, profitability, leverage, financial burden, efficiency and four trend blocks), winsorised at the 1st and 99th percentile, standardised, averaged within each block, signed by risk direction and summed. The riskiest 4% of firms are flagged. It is built from the full set of engineered ratios, not from the 10 selected features. |
| **Logistic regression** | Median imputation, standardisation, L2 penalty with `C=0.1`, balanced class weights. Decision threshold 0.8077. |
| **Random forest** | 300 trees, `max_depth=7`, `min_samples_leaf=10`, balanced class weights, median imputation. Decision threshold 0.5089. |

A probit model was also tested during exploration but it is not part of the final ensemble.

### 4. Ensemble

Each model casts a vote and a firm is predicted to default when **at least 2 of the 3 models** flag it. The out-of-sample probabilities for the logit and the random forest come from stratified 5-fold cross-validation, so no firm is scored by a model that saw it during training. The decision thresholds were chosen from the precision-recall curves of those out-of-sample predictions.

## Results

Performance on the training set (out-of-fold predictions for the logit and the random forest):

| Model | Recall | Precision | F1 | Accuracy |
|-------|--------|-----------|----|----------|
| Altman-inspired score | 0.314 | 0.611 | 0.415 | 0.930 |
| Logistic regression (5-fold CV) | 0.471 | 0.617 | 0.534 | 0.935 |
| Random forest (5-fold CV) | 0.671 | 0.443 | 0.534 | 0.908 |
| **Ensemble (majority vote)** | **0.500** | **0.654** | **0.567** | **0.940** |

The ensemble raises precision over each individual model while keeping F1 close to the best single model. It catches half of the defaults with about one false alarm for every two correct flags.

On the 446 test firms the ensemble flags **29 firms as likely defaults (6.5%)** and classifies the other 417 as non-defaults.

## Known limitations

These numbers are estimates on the training data, not a score on held-out labels, because the test labels are not public. They are probably optimistic for three reasons:

- Feature selection (correlation filter and importance ranking) was done on the full training set before cross-validation, so the folds are not fully independent of the selection.
- The decision thresholds were tuned on the same out-of-fold predictions that are used to report the metrics.
- The Altman-inspired score is evaluated in-sample, with no cross-validation.
- Median imputation for the random forest is fitted on the full training set before cross-validation, so the validation folds are not completely independent of the imputation step.

A cleaner evaluation would put feature selection and threshold choice inside the cross-validation loop, or reserve a separate validation split.

## How to run

The project was developed in Google Colab with Python 3.

1. Install the dependencies: `pip install pandas numpy scikit-learn statsmodels matplotlib openpyxl`
2. Place the three Excel files supplied by the organisers in the `data/` folder. They are not tracked by git.
3. Open the notebook and run all cells from top to bottom. The feature selection step overwrites the feature tables, so always run in order from a fresh kernel.
4. The last cell writes the predicted labels to `challenge_submission_final.xlsx`.

## Repository structure

```
.
├── README.md
├── notebooks/        # analysis notebook
├── data/             # not tracked: put the challenge files here
└── requirements.txt
```

## Team

Behlaryan Dzhuliieta, Beltrame Michele, Brasolin Alessandro, Giura Alessandro, Rozenberg Modestas, Sciupakov Artiom, Zamprotta Leonardo

University of Padova and Vilnius University.

## References

- Altman, E. I. (1968). *Financial Ratios, Discriminant Analysis and the Prediction of Corporate Bankruptcy.*
- Breiman, L. (2001). *Random Forests.*
- Hastie, T., Tibshirani, R., & Friedman, J. (2009). *The Elements of Statistical Learning* (2nd ed.). Springer.
- Lundholm, R., & Sloan, R. (2023). *Equity Valuation and Analysis* (6th ed.).
- McLaney, E., & Atrill, P. (2018). *Accounting and Finance: An Introduction* (10th ed.). Pearson Education Limited.
