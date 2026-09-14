# DATA VORTEX — Round 1 Phase 1
## Dataset 01: Data Intake Restoration

### 1. Project Objective

The objective of Phase 1 was to restore and analyze Dataset 01 from the Social Engine. The recovered dataset consisted of a Users table and a deliberately corrupted Posts table.

The workflow followed:

**Observe → Audit → Test → Decide → Transform → Validate → Analyze**

The original recovered files were preserved as the raw source. Cleaning was performed on copies so that every transformation remained traceable.

---

## 2. Source Files

### Users
`Social_Engine_Users.csv`

- 1,500 rows
- 5 columns
- 1,500 unique users
- No missing values detected
- No duplicate rows detected

Columns:

- `user_id`
- `location`
- `language`
- `account_created`
- `follower_count`

### Posts
`Social_Engine_Posts_Corrupted.csv`

- 12,360 rows
- 8 original columns
- 12,000 unique `post_id` values
- 360 extra duplicate records
- All post `user_id` values matched the Users table

Columns:

- `post_id`
- `user_id`
- `platform`
- `text_content`
- `timestamp`
- `likes`
- `shares`
- `comments`

---

## 3. Data-Quality Findings

### 3.1 Missing Values

The Posts dataset contained missing values in three fields:

| Field | Missing |
|---|---:|
| `platform` | 1,846 |
| `text_content` | 1,746 native missing + 24 `NULL\n\n` sentinel records |
| `likes` | 1,858 |

The literal CSV audit showed that missing values were represented by both empty fields and the literal token `NULL`.

There were no whitespace-only values in `platform` or `text_content`.

### 3.2 Missingness Investigation

Missingness rates were approximately:

- `platform`: 14.94%
- `text_content`: 14.13%
- `likes`: 15.03%

Pairwise chi-square tests found no statistically significant association between the missingness indicators:

- `platform` + `text_content`: p = 0.141928
- `platform` + `likes`: p = 0.396916
- `text_content` + `likes`: p = 0.513834

The observed number of records with all three fields missing was 40, compared with 39.2 expected under independence.

For missing `likes`, additional tests found no statistically significant association with:

- known platform: p = 0.619161
- month: p = 0.748295
- user activity level: p = 0.919330

These results supported treating missing `likes` as broadly distributed rather than concentrated in a tested subgroup.

---

## 4. Cleaning Decisions

### 4.1 Platform

Missing `platform` values were replaced with:

`Unknown`

Valid platform labels were preserved because no inconsistent capitalization or spelling variants were detected.

### 4.2 Text Content

The following were treated as missing text:

- empty values
- `NULL`
- `NULL\n\n`

They were replaced with the explicit marker:

`No text content`

No original post text was fabricated.

### 4.3 Likes

The raw dataset contained 525 negative `likes` values.

Forensic analysis found:

- negative values ranged from -4,987 to -11
- their absolute values remained within the normal observed likes range
- the distribution of absolute negative values was consistent with the distribution of valid likes
- the sign-flip hypothesis was therefore strongly supported

Negative likes were restored using absolute value:

`likes = abs(likes)`

The remaining 1,858 missing `likes` values were replaced with the median of the valid observed likes:

`2500`

This choice was made after testing platform, time, user activity, and engagement-variable relationships. Shares and comments showed essentially no useful linear relationship with likes.

### 4.4 Timestamp

Three valid timestamp representations were identified:

- `DD-MM-YYYY`: 3,622 records
- ISO datetime: 4,950 records
- Unix epoch seconds: 3,788 records

All 12,360 timestamps were successfully validated and converted to a common datetime representation.

No timestamp rows were removed.

### 4.5 Duplicates

There were:

- 352 duplicated `post_id` values
- 712 physical records belonging to duplicate groups
- 360 extra duplicate copies

No rows were removed.

A unique `record_id` was assigned to every physical record, and `is_duplicate_record` was added to identify records belonging to exact duplicate groups.

This preserves all source observations while maintaining physical-record traceability.

---

## 5. Final Validation

The final analytical dataset contains:

- 12,360 rows
- 10 columns
- 0 missing values
- 0 negative likes
- 0 missing timestamps
- 12,360 unique `record_id` values
- 0 duplicate `record_id` values
- 0 invalid `user_id` references
- `likes` range: 0–5,000
- `shares` range: 0–2,000
- `comments` range: 0–1,000

No source rows were removed.

The exported CSV was reloaded and independently checked after writing to disk.

---

## 6. Exploratory Data Analysis

### Platform

Instagram had the highest descriptive average composite engagement per post:

**4,044.22**

YouTube had the highest total composite engagement:

**8,611,269**

Platform engagement contribution broadly followed post volume.

### Time

Highest monthly post volume:

**May 2024 — 1,074 posts**

Lowest monthly post volume:

**February 2025 — 946 posts**

Highest average monthly composite engagement:

**August 2024 — 4,088.19**

Lowest:

**January 2025 — 3,920.71**

No sustained annual trend was established.

### Weekday

Posting volume was statistically consistent with a uniform seven-day distribution:

- χ² = 4.1487
- p = 0.65656

Mean composite engagement did not differ significantly by weekday:

- F = 0.8016
- p = 0.568456

### Text

Among the 10,590 posts with genuinely observed text:

- mean text length = 117.63 characters
- median = 118 characters

Spearman correlation between text length and composite engagement:

**ρ = -0.0138**

This indicates a negligible relationship.

### Hashtags and Mentions

Among genuine-text posts:

- hashtag present: 10,520
- mention present: 1,415
- neither: 70

All mention-containing posts also contained a hashtag.

The three content groups did not differ significantly in mean composite engagement:

- F = 1.3351
- p = 0.263166

### User Characteristics

Follower count showed essentially no relationship with average engagement:

- Pearson r = -0.0017, p = 0.947942
- Spearman ρ = 0.0047, p = 0.856526

Location differences were not statistically significant:

- F = 0.7559
- p = 0.835694

Language differences were not statistically significant:

- F = 1.1293
- p = 0.338218

### Engagement Anomalies

Using the 1.5×IQR rule:

- total engagement outliers: 0
- likes outliers: 0
- shares outliers: 0
- comments outliers: 0

This means no observations were removed based on engagement outlier detection.

---

## 7. Limitations

1. `likes` contains 1,858 median-imputed observations at 2,500. Therefore, analyses involving likes should be interpreted with awareness of the imputation.
2. The composite engagement measure is a derived metric defined as:
   `likes + shares + comments`
3. Descriptive differences do not imply causation.
4. Statistical non-significance does not prove that two variables are absolutely independent.
5. Duplicate source observations were intentionally retained to satisfy the no-row-removal requirement.

---

## 8. Final Output

Primary cleaned dataset:

`Social_Engine_Posts_Cleaned.csv`

The cleaned dataset is intended for the next competition phase, where it can be converted into SQL tables and analyzed using SQL queries.
