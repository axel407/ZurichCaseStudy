# Case Study Methodology Proposal

Here is my proposal on how to approach this case study. It covers everything from EDA to deployment, with my own observations of the dataset provided in the case study and a critique of the provided solution in the notebook, where I caught multiple critical errors that would affect the performance and usability of the final model in production.

---

## 1. Exploratory Data Analysis

### 1.1 Approach

1. Read the background and the data dictionary. The first thing that caught my eye in the background is the divide between Auto Physical Damage (APD) and Auto Liability (AL).
2. Separate the dataset based on coverage_type. The background says those coverages have different drivers for the target, so the relevant features are going to be different.
3. Look into numeric distributions and outliers (especially loss amounts, which are heavy tailed).
4. Look at claim rate and loss by business_type and state.

### 1.2 Findings that influence modeling choices

- **Leaky features:** first_claim_reported_date, claim_paid_amount_current_period, claim_status_current_period and days_to_first_claim_report are available only after claim resolution, not at policy inception. Two of them were used as features in the notebook (see 2.2).
- **Inconsistency between train and score:** those columns are also in score.csv, even though there should be no outcomes for 2023-2024. 300 rows have claim_paid_amount_current_period filled and claim_status_current_period is "INVALID" for them, while in train.csv the same column has proper values (No Claim, Open, Closed, Under Review). So those columns are not only leaky, but also not consistent between train and score (see 3.1).
- **Target:** right now the target is the binary flag claim_count > 0, but it contradicts our goal. We are trying to predict risk, not just answer the question "Was there a claim?". Risk is calculated as Frequency × Severity, so we would need both claim_count and total_loss_amount, and I would look into both of them separately (see 2.3).

---

## 2. Modeling Methodology Improvements (top 3)

An accuracy of one, which is practically impossible, indicates that a lot of things have gone wrong. I would change these three things first:

### 2.1 Time-based split

Change the random split to a time-based one: 2020-2021 for train and 2022 for test. It would better represent what we are trying to do with our model.

### 2.2 Remove leaky columns, fix preprocessing and feature selection

- **Leakage:** numeric missing values are just substituted with zero, which can lead to data leakage, and that is what happened. Two of the leaky columns (claim_paid_amount_current_period and days_to_first_claim_report) were directly in feat_cols of the original notebook. Those columns are clear indicators that a claim had been made, which makes the model "look" for those flags and just remember the correlation for prediction. I would remove them from the train and test datasets.
- **Missing data and duplicates:** I would drop duplicates, if any. If EDA shows a significant amount of missing data, I would consider substituting it with the mean or median; if there are too many missing entries, I would drop columns and rows. For features where a missing value can mean something (like prior_loss_amount) I would also add a flag column "is_missing", so the model can learn from it.
- **Encoding:** categorical columns (coverage_type, business_type, state) are currently encoded with .cat.codes, which gives them a fake order (for example Trucking < Retail), and the codes are created separately for train and score, so the same number can mean a different category. I would use one-hot encoding fitted only on train data and applied the same way to score data.
- **Feature selection:** I would run a proper feature selection method, rather than blindly picking every column from the dataframe, which is currently done in the notebook. Without actually digging into the data I cannot say what method I would choose.

### 2.3 Redefine the target

Right now it's the binary question "Was there a claim?" and it doesn't reflect the goal of finding a risk score. I would model frequency (claim_count, Poisson model) and severity (total_loss_amount of policies with a claim, Gamma model) separately and multiply them to get expected loss, or use one Tweedie model on total_loss_amount directly.

---

## 3. Production Readiness Improvements (top 3)

With the issues from section 2 fixed, I would add:

### 3.1 Input data validation

Catch, for example, "leaky" columns in score.csv, missing columns (has_safety_program), unknown categories or values like "INVALID" before scoring runs.

### 3.2 Monitoring

Track inputs and outputs for data drift or concept drift.

### 3.3 Reproducibility

Fixed random_state, serialized model artifact + version, logged hyperparameters/training data snapshot (currently the model is fit fresh with no seed, and nothing is saved beyond predictions.csv).

---

## 4. Deployment Roadmap

Before this model could go to production I would do these steps:

1. Retrain with the changes from section 2 and validate on the time-based split. The current notebook model stays as a baseline to compare with.
2. Check results with underwriters / actuaries to see if risk scores make sense by business_type, state and coverage_type. For example, trucking with a lot of heavy vehicles should come out as riskier than a small service company.
3. Explainability: insurance pricing is regulated, so for every score we should be able to say why it is high or low (GLM coefficients or SHAP values).
4. Shadow deployment: run the model next to the current underwriting process for some time without using its output for decisions, and compare its scores with real outcomes when they come.
5. Set up validation, monitoring and alerts from section 3 before go-live.
6. Go-live with a rollback plan (keep the previous version ready) and a defined retraining schedule, for example once a year when new claim outcomes are available.

---

## 5. AI Tools Used

I used Claude through Claude Code in VS Code. I used it to discuss the direction of EDA, check my reasoning about the problems in the prototype notebook, and to help with editing parts of this proposal. All findings were checked against the data and notebook.
