# Data Preprocessing & EDA — Notes

Reference for exploring a dataset, understanding it visually and statistically, and cleaning it up (missing values, outliers) before modeling.

---

## 1. Exploratory Data Analysis (EDA) — What & Why

EDA is the step where you **look at the data before touching it** — understanding its shape, spotting problems (missing values, outliers, skew), and getting a feel for relationships between variables. The goal isn't to fix anything yet, just to understand what you're working with.

```python
df.shape
```
- Returns (rows, columns) — quick sense of dataset size.

```python
df.head()
df.sample(5)
```
- `head()` — first 5 rows. `sample(5)` — 5 random rows (better for spotting patterns than always seeing the top).

```python
df.info()
```
- Column names, data types, and non-null counts in one view — the fastest way to spot missing values and wrong types (e.g. a number stored as text).

```python
df.dtypes.value_counts()
```
- Quick count of how many columns are numeric vs. categorical.

```python
df.duplicated().sum()
```
- Counts fully duplicated rows — worth checking and dropping (`df.drop_duplicates()`) before analysis.

---

## 2. Statistics of the Data

```python
df.describe()
```
- Summary stats for **numeric** columns: count, mean, std, min, 25%/50%/75% quartiles, max. This alone tells you a lot — e.g. a huge gap between the 75th percentile and max often signals outliers.

```python
df.describe(include='object')
```
- Summary stats for **categorical** columns: count, unique values, most frequent value (top), and its frequency.

```python
df[col].value_counts()
```
- Frequency count of each category in a column — good for spotting class imbalance (e.g. 90% "No" vs 10% "Yes").

```python
df.corr(numeric_only=True)
```
- Correlation matrix between numeric columns (values from -1 to 1). Helps spot which features move together, and which are strongly related to the target.

```python
df.skew(numeric_only=True)
```
- Measures how lopsided a numeric column's distribution is. Values far from 0 (roughly beyond ±1) suggest skew that may benefit from a transformation (e.g. log transform).

---

## 3. Visualizing the Data

Different chart types answer different questions — here's what to reach for and when.

### Distribution of a single numeric column
```python
import seaborn as sns
sns.histplot(df[col], kde=True)
```
- **Histogram** — shows the shape of a numeric column's distribution (normal, skewed, bimodal, etc.). `kde=True` overlays a smoothed curve.

### Spotting outliers in a numeric column
```python
sns.boxplot(x=df[col])
```
- **Boxplot** — shows the median, quartiles (the "box"), and any points beyond the whiskers are flagged as potential outliers. The single fastest way to visually check for outliers.

### Relationship between two numeric columns
```python
sns.scatterplot(x=df[col1], y=df[col2])
```
- **Scatter plot** — shows whether two numeric variables move together (linear, non-linear, or no relationship).

### Relationships between all numeric columns at once
```python
sns.heatmap(df.corr(numeric_only=True), annot=True, cmap='coolwarm')
```
- **Correlation heatmap** — color-coded version of the correlation matrix from §2; `annot=True` prints the actual numbers on each cell. Fast way to scan for strongly related (or redundant) features.

### Frequency of a categorical column
```python
sns.countplot(x=df[col])
```
- **Count plot (bar chart)** — how many rows fall into each category. Good for spotting class imbalance visually.

### Numeric column across categories
```python
sns.boxplot(x=df[cat_col], y=df[num_col])
```
- **Grouped boxplot** — compares the distribution of a numeric column (e.g. income) across categories (e.g. education level). Useful for spotting whether a feature actually differs meaningfully by group.

### Pairwise overview of multiple columns
```python
sns.pairplot(df[[col1, col2, col3, 'target']], hue='target')
```
- **Pairplot** — grid of scatter plots (and histograms on the diagonal) for every pair of chosen columns at once. Good for a quick first look at a handful of important features together, but gets cluttered with too many columns — pick a small subset.

---

## 4. Outlier Detection & Handling

Outliers are extreme values that can distort statistics and skew model training (especially for linear/distance-based models).

### Detecting outliers

**Boxplot (visual)** — already shown above; fastest first check.

**IQR method (statistical)**
```python
Q1 = df[col].quantile(0.25)
Q3 = df[col].quantile(0.75)
IQR = Q3 - Q1
lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR
outliers = df[(df[col] < lower_bound) | (df[col] > upper_bound)]
```
- Anything outside `1.5 × IQR` beyond the quartiles is flagged as an outlier — the same rule a boxplot's whiskers use under the hood.

**Z-score method (statistical)**
```python
from scipy import stats
z_scores = stats.zscore(df[col])
outliers = df[abs(z_scores) > 3]
```
- Flags values more than 3 standard deviations from the mean. Works best when the column is roughly normally distributed — less reliable on skewed data.

### Handling outliers

**Clip/cap to a percentile range** (keeps all rows, just limits extreme values)
```python
df[col] = df[col].clip(*df[col].quantile([0.05, 0.95]))
```

**Remove outlier rows entirely**
```python
df = df[(df[col] >= lower_bound) & (df[col] <= upper_bound)]
```

**Transform to reduce the impact of extreme values** (common for skewed, strictly positive data like income or price)
```python
import numpy as np
df[col] = np.log1p(df[col])   # log(1 + x), handles zeros safely
```

> **Which to pick?** Clipping is the safest default — it keeps your dataset size intact while reducing distortion. Removing rows is fine if outliers are clearly data-entry errors and few in number. Log transforms are best when the whole column is naturally skewed (not just a few stray points).

---

## 5. Handling Missing Values (Imputation)

### Detecting missing values
```python
df.isnull().sum()
df.isnull().mean() * 100   # percentage missing per column
```

### Visualizing missing values
```python
import missingno as msno
msno.matrix(df)
```
- Shows a visual map of where values are missing across the dataset — useful for spotting patterns (e.g. missing together in related columns).

### Imputation techniques

**Mean/Median (numeric columns)**
```python
df[col].fillna(df[col].mean(), inplace=True)    # if roughly symmetric
df[col].fillna(df[col].median(), inplace=True)  # if skewed / has outliers — usually the safer default
```

**Mode (categorical columns)**
```python
df[col].fillna(df[col].mode()[0], inplace=True)
```

**Forward-fill / backward-fill** (useful for time-ordered data)
```python
df[col].fillna(method='ffill', inplace=True)  # carry last valid value forward
df[col].fillna(method='bfill', inplace=True)  # carry next valid value backward
```

**Constant/placeholder value** (when "missing" itself is meaningful)
```python
df[col].fillna("Unknown", inplace=True)
```

**KNN Imputation** (estimates missing values based on similar rows)
```python
from sklearn.impute import KNNImputer
imputer = KNNImputer(n_neighbors=5)
df_imputed = imputer.fit_transform(df[numeric_cols])
```
- More accurate than simple mean/median fill since it uses patterns from similar rows, but more computationally expensive — best for smaller/medium datasets.

**Drop rows/columns instead of imputing**
```python
df.dropna(subset=[col], inplace=True)          # drop rows missing this column
df.drop(columns=[col], inplace=True)            # drop the whole column
```
- Reasonable when a column is missing too much data to reliably impute (commonly >40-50% missing), or when only a handful of rows are affected.

> **Which to pick?** Median/mode fill is the standard beginner-safe default. Use KNN imputation when you want more accuracy and can afford the extra computation. Drop a column only when too much of it is missing to trust any imputation.

---

## Quick Reference — Cheat Sheet

| Task | Command |
|---|---|
| Dataset shape | `df.shape` |
| Column types + nulls | `df.info()` |
| Numeric stats | `df.describe()` |
| Categorical stats | `df.describe(include='object')` |
| Category frequencies | `df[col].value_counts()` |
| Correlation matrix | `df.corr(numeric_only=True)` |
| Distribution chart | `sns.histplot(df[col])` |
| Outlier check (visual) | `sns.boxplot(x=df[col])` |
| Relationship (2 numeric) | `sns.scatterplot(x=..., y=...)` |
| Correlation chart | `sns.heatmap(df.corr(), annot=True)` |
| Category counts chart | `sns.countplot(x=df[col])` |
| Outlier detection (stat) | IQR rule or Z-score |
| Outlier handling | `.clip()`, row removal, or log transform |
| Missing value check | `df.isnull().sum()` |
| Simple imputation | `.fillna(mean/median/mode)` |
| Smarter imputation | `KNNImputer` |
