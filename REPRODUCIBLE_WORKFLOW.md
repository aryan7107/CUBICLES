# DATA VORTEX — Reproducible Workflow

## Purpose

This document describes how to reproduce the Phase 1 data-restoration and EDA workflow from the raw recovered CSV files.

The workflow intentionally keeps raw data unchanged and performs transformations on copies.

---

## 1. Project Structure

Recommended structure:

```text
DATA_VORTEX/
│
├── Social_Engine_Posts_Corrupted.csv
├── Social_Engine_Users.csv
├── Social_Engine_Posts_Cleaned.csv
│
├── 01_Data_Vortex_Phase1.ipynb
├── DOCUMENTATION.md
├── REPRODUCIBLE_WORKFLOW.md
└── requirements.txt
```

---

## 2. Environment

Recommended Python environment:

- Python 3.12
- pandas
- numpy
- scipy
- matplotlib
- pathlib

Example installation:

```bash
pip install pandas numpy scipy matplotlib
```

---

## 3. Load the Raw Files

Use paths relative to the project directory so the notebook remains portable.

```python
from pathlib import Path
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

from scipy.stats import (
    chi2_contingency,
    chisquare,
    f_oneway,
    pearsonr,
    spearmanr
)

pd.set_option("display.max_columns", None)
pd.set_option("display.width", 140)

BASE_DIR = Path.cwd()

POSTS_PATH = BASE_DIR / "Social_Engine_Posts_Corrupted.csv"
USERS_PATH = BASE_DIR / "Social_Engine_Users.csv"

posts_raw = pd.read_csv(POSTS_PATH)
users_raw = pd.read_csv(USERS_PATH)
```

---

## 4. Preserve a Literal-Token Audit Copy

This prevents pandas from automatically converting tokens such as `NULL` into NaN during the audit.

```python
posts_audit = pd.read_csv(
    POSTS_PATH,
    keep_default_na=False,
    na_filter=False
)

users_audit = pd.read_csv(
    USERS_PATH,
    keep_default_na=False,
    na_filter=False
)
```

This copy is used only for forensic inspection.

---

## 5. Baseline Audit

Record:

```python
print(posts_raw.shape)
print(users_raw.shape)

print(posts_raw.columns.tolist())
print(users_raw.columns.tolist())

print(posts_raw.dtypes)
print(users_raw.dtypes)

posts_raw.info()
users_raw.info()
```

Also record:

```python
posts_raw.isna().sum()
users_raw.isna().sum()

posts_raw.duplicated().sum()
users_raw.duplicated().sum()

posts_raw["post_id"].nunique()
users_raw["user_id"].nunique()
```

The baseline should show:

- Posts: 12,360 × 8
- Users: 1,500 × 5
- 360 exact duplicate Posts rows
- 1,500 unique Users IDs

---

## 6. Forensic Missing-Value Audit

Inspect literal tokens in `posts_audit`:

```python
for col in ["platform", "text_content", "likes"]:
    counts = posts_audit[col].value_counts(dropna=False)
    print(col)
    print(counts.head(20))
```

Important findings:

- empty platform: 1,219
- `NULL` platform: 627
- empty text: 1,196
- `NULL` text: 550
- `NULL\n\n` text: 24
- empty likes: 1,229
- `NULL` likes: 629

---

## 7. Missingness Relationship Analysis

Create missingness flags:

```python
cols = ["platform", "text_content", "likes"]
missing_flags = posts_raw[cols].isna()
```

Evaluate combinations:

```python
missing_flags.value_counts()
```

Then test pairwise independence with chi-square contingency tables.

No data is modified during this stage.

---

## 8. Timestamp Forensics

Identify formats using regular expressions:

```python
timestamp_raw = posts_raw["timestamp"].astype(str).str.strip()

is_ddmmyyyy = timestamp_raw.str.fullmatch(
    r"\d{2}-\d{2}-\d{4}"
)

is_iso = timestamp_raw.str.fullmatch(
    r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}"
)

is_epoch = timestamp_raw.str.fullmatch(
    r"\d{10}"
)
```

Validate each representation separately:

```python
pd.to_datetime(
    timestamp_raw.loc[is_ddmmyyyy],
    format="%d-%m-%Y",
    errors="coerce"
)

pd.to_datetime(
    timestamp_raw.loc[is_iso],
    format="%Y-%m-%dT%H:%M:%S",
    errors="coerce"
)

pd.to_datetime(
    timestamp_raw.loc[is_epoch].astype("int64"),
    unit="s",
    errors="coerce"
)
```

All three formats should validate with zero failures.

---

## 9. Create Clean Copies

```python
posts_clean = posts_raw.copy()
users_clean = users_raw.copy()
```

Never modify `posts_raw` or `users_raw`.

---

## 10. Add Traceability Fields

```python
posts_clean.insert(
    0,
    "record_id",
    [f"REC_{i:05d}" for i in range(1, len(posts_clean) + 1)]
)
```

Add the original timestamp before standardization:

```python
posts_clean["timestamp_original"] = posts_clean["timestamp"]
```

---

## 11. Clean Platform

```python
posts_clean["platform"] = posts_clean["platform"].fillna("Unknown")
```

---

## 12. Clean Text

Treat the literal missing-like text values as missing:

```python
text_sentinel_mask = (
    posts_clean["text_content"]
    .astype("string")
    .str.strip()
    .isin(["NULL", ""])
)

posts_clean.loc[text_sentinel_mask, "text_content"] = np.nan

posts_clean["text_content"] = (
    posts_clean["text_content"]
    .fillna("No text content")
)
```

---

## 13. Repair Negative Likes

Use the supported sign-restoration rule:

```python
negative_mask = posts_clean["likes"] < 0

posts_clean.loc[negative_mask, "likes"] = (
    posts_clean.loc[negative_mask, "likes"].abs()
)
```

---

## 14. Impute Missing Likes

Calculate the median from originally valid observed likes:

```python
valid_like_median = posts_clean.loc[
    posts_clean["likes"].notna(),
    "likes"
].median()
```

Expected value:

`2500`

Apply:

```python
posts_clean["likes"] = (
    posts_clean["likes"]
    .fillna(valid_like_median)
)
```

---

## 15. Standardize Timestamp

```python
timestamp_raw_clean = (
    posts_clean["timestamp"]
    .astype(str)
    .str.strip()
)

is_ddmmyyyy_clean = timestamp_raw_clean.str.fullmatch(
    r"\d{2}-\d{2}-\d{4}"
)

is_iso_clean = timestamp_raw_clean.str.fullmatch(
    r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}"
)

is_epoch_clean = timestamp_raw_clean.str.fullmatch(
    r"\d{10}"
)

standardized_timestamp = pd.Series(
    pd.NaT,
    index=posts_clean.index,
    dtype="datetime64[ns]"
)

standardized_timestamp.loc[is_ddmmyyyy_clean] = pd.to_datetime(
    timestamp_raw_clean.loc[is_ddmmyyyy_clean],
    format="%d-%m-%Y"
)

standardized_timestamp.loc[is_iso_clean] = pd.to_datetime(
    timestamp_raw_clean.loc[is_iso_clean],
    format="%Y-%m-%dT%H:%M:%S"
)

standardized_timestamp.loc[is_epoch_clean] = pd.to_datetime(
    timestamp_raw_clean.loc[is_epoch_clean].astype("int64"),
    unit="s"
)

posts_clean["timestamp"] = standardized_timestamp
```

---

## 16. Preserve Duplicate Rows

Do not use `drop_duplicates()`.

Instead:

```python
duplicate_columns = [
    "post_id",
    "user_id",
    "platform",
    "text_content",
    "timestamp",
    "likes",
    "shares",
    "comments"
]

posts_clean["is_duplicate_record"] = posts_clean.duplicated(
    subset=duplicate_columns,
    keep=False
)
```

This preserves every source row.

---

## 17. Final Analytical Dataset

Keep:

```python
final_columns = [
    "record_id",
    "post_id",
    "user_id",
    "platform",
    "text_content",
    "timestamp",
    "likes",
    "shares",
    "comments",
    "is_duplicate_record"
]

posts_final = posts_clean[final_columns].copy()
```

---

## 18. Validation

Run:

```python
assert len(posts_final) == len(posts_raw)
assert posts_final.isna().sum().sum() == 0
assert (posts_final["likes"] < 0).sum() == 0
assert posts_final["record_id"].nunique() == len(posts_final)
assert posts_final["user_id"].isin(users_raw["user_id"]).all()
assert posts_final["timestamp"].notna().all()
```

Also verify:

```python
assert posts_final["likes"].between(0, 5000).all()
assert posts_final["shares"].between(0, 2000).all()
assert posts_final["comments"].between(0, 1000).all()
```

---

## 19. Export

```python
OUTPUT_PATH = BASE_DIR / "Social_Engine_Posts_Cleaned.csv"

posts_final.to_csv(
    OUTPUT_PATH,
    index=False
)
```

Reload and validate the exported file:

```python
posts_final_check = pd.read_csv(OUTPUT_PATH)

assert len(posts_final_check) == 12360
assert posts_final_check.isna().sum().sum() == 0
assert (posts_final_check["likes"] < 0).sum() == 0
assert posts_final_check["record_id"].nunique() == 12360
```

For EDA, parse the exported timestamp:

```python
posts_eda = posts_final_check.copy()

posts_eda["timestamp"] = pd.to_datetime(
    posts_eda["timestamp"],
    errors="raise"
)
```

---

## 20. EDA Reproduction

The notebook then reproduces the Phase 1 analyses:

1. Numerical engagement summary
2. Distribution shape and boundary analysis
3. Platform-level engagement
4. Platform contribution
5. Monthly activity and engagement
6. Day-of-week posting activity
7. Day-of-week engagement
8. Text length
9. Hashtag/mention combinations
10. Engagement by content group
11. User-level engagement
12. Follower count vs engagement
13. Location distribution and engagement
14. Language distribution and engagement
15. Engagement anomaly analysis
16. Engagement composition and robustness check

All derived metrics are calculated from the cleaned dataset and documented as analytical measures rather than original source fields.

---

## 21. Reproducibility Principles

- Raw CSVs are never overwritten.
- Cleaning is performed on copies.
- Every transformation is explicit.
- Every major transformation is followed by validation.
- No rows are removed.
- Duplicate source records are retained and flagged.
- Missing values are resolved using documented rules.
- Timestamp formats are validated before conversion.
- Statistical tests use a predefined α = 0.05 threshold.
- Exported files are reloaded and checked.
- EDA is performed only after the final dataset passes structural validation.
