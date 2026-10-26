# Khetha Mutual Data Pipeline

Milestone 1 establishes the data engineering layer for Khetha Mutual's claims
analytics system. The pipeline converts raw claims data into validated,
model-ready feature and target sets while preventing identifiers, future
information, and post-outcome proxy variables from entering the predictors.

## What This Pipeline Does

`KhethaDataPipeline` performs the following workflow:

1. Loads `claims_data.csv` and reads optional settings from `config.json`.
2. Detects whether the target is a classification or regression problem using
	target cardinality and descriptive statistics.
3. Separates predictors (`X`) from the target (`y`).
4. Removes configured and automatically detected leakage columns.
5. Validates target values, row counts, index alignment, and feature dtypes.
6. Splits the data into training and test sets with a fixed random seed.
7. Uses `stratify=y` for classification targets to preserve class balance.
8. Compares numeric means/standard deviations and categorical proportions
	between train and test for the requested key features.
9. Calculates class counts, the majority/minority ratio, and whether the
	minority class represents less than 5% of observations.

## Requirements

- Python 3.10 or newer
- pandas
- NumPy
- scikit-learn

Install the dependencies with:

```bash
python -m pip install pandas numpy scikit-learn
```

## Run the Pipeline

Run from the repository root:

```bash
python khetha_data_pipeline.py
```

The script locates data relative to its own file, so it can also be started
from another working directory:

```bash
python /path/to/khetha_data_pipeline.py
```

The executable example uses `claims_data.csv`, automatically selects
`fraud_label` when no target is configured, and uses `province` as the default
representativeness feature. A successful run prints the task type, split sizes,
number of model features, removed leakage columns, and diagnostic summaries.

Typical output includes:

```text
Pipeline Complete | Task: classification | Train: 8000, Test: 2000
```

The supplied data has a minority fraud class below 5%. That warning is
intentional: it identifies a modeling risk and does not stop a valid pipeline
run.

## Repository Data

| File | Purpose |
| --- | --- |
| `claims_data.csv` | Primary claims table and default modeling input |
| `fraud_labels.csv` | Investigation and fraud-label reference data |
| `vehicles.csv` | Vehicle attributes |
| `policyholders.csv` | Policyholder attributes |
| `geographic_risks.csv` | Geographic risk reference data |
| `geographic_risk.csv` | Alternate geographic-risk filename supported by the runner |
| `config.json` | Optional target, key-feature, and leakage configuration |

The command-line runner validates that companion datasets are present, but it
does not blindly merge them. A merge is only appropriate when an unambiguous
foreign key is available. The supplied claims table does not contain all of
the keys required to safely join every auxiliary table, so the primary claims
table remains the modeling input for this milestone.

## Leakage Prevention

The pipeline removes any columns supplied through `leakage_cols` and also
detects common leakage patterns. Removed fields include:

- Record identifiers such as claim, policy, vehicle, UUID, VIN, and reference IDs
- Dates and timestamps, including settlement, payment, decision, and closure dates
- Investigation and outcome fields
- Fraud labels, confirmation fields, results, recovered amounts, payouts, and
  other target or post-processing proxies

Datetime-typed columns are removed even when their names do not match a known
pattern. The removed columns are available in
`pipeline.leakage_columns_removed` for auditability.

## Python API

```python
import pandas as pd
from khetha_data_pipeline import KhethaDataPipeline

claims = pd.read_csv("claims_data.csv")
pipeline = KhethaDataPipeline(
	 claims,
	 leakage_cols=["claim_settlement_date"],
)

X_train, X_test, y_train, y_test = pipeline.run(
	 target_col="fraud_label",
	 key_features=["province"],
)
```

Important public attributes after `run()`:

- `task_type`: either `classification` or `regression`
- `X`, `y`: validated predictors and target before splitting
- `X_train`, `X_test`, `y_train`, `y_test`: model-ready split data
- `leakage_columns_removed`: columns excluded from `X`
- `representativeness_report`: per-feature distribution comparisons
- `class_imbalance_report`: class counts, ratio, minority percentage, and flag

The lower-level methods can also be called independently:

```python
pipeline.task_type = pipeline.detect_task_type("fraud_label")
pipeline.isolate_features("fraud_label")
pipeline.split_with_stratification(test_size=0.2, random_state=42)
pipeline.check_representativeness(["province"])
pipeline.check_class_imbalance()
```

## Configuration

`config.json` is optional for execution. The runner searches nested JSON
objects for these settings:

- `target_column`, `target_col`, `target_variable`, or related target names
- `key_features`, `representativeness_features`, or provincial feature names
- `leakage_cols`, `leakage_columns`, or `columns_to_drop`

Column matching is case-insensitive and ignores spaces, underscores, and other
punctuation. If configuration does not identify a target, the runner tries
`fraud_label`, `fraud_confirmed`, and then `claim_amount`.

## Validation and Diagnostics

The implementation raises an error when:

- The input is not a pandas DataFrame, is empty, or has duplicate columns
- The target is missing, null, or has fewer than two distinct values
- Feature and target indexes or row counts do not align
- A classification split cannot place every class in both partitions
- Train/test feature dtypes differ from the original feature dtypes

Representativeness checks use a 5% threshold. Numeric features compare train
and test means and standard deviations. Categorical features compare the
largest difference in category proportions. Deviations above the threshold
are recorded and surfaced as runtime warnings.

For classification, the imbalance diagnostic reports:

- Class counts
- Majority/minority ratio
- Minority proportion and percentage
- Whether the minority class is below 5%

## Verification

The minimum smoke test is:

```bash
python -m py_compile khetha_data_pipeline.py
python khetha_data_pipeline.py
```

The second command is the same root-level execution pattern used by the
milestone autograder.