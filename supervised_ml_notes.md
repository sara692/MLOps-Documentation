# Supervised Machine Learning — Pipeline Notes

End-to-end reference: loading data → preprocessing → training → evaluation → saving. Regression and Classification are separated in the modeling sections since they use different algorithms and metrics.

---

## 1. Load the Data

```python
import pandas as pd
df = pd.read_csv("data.csv")
df.head()
df.info()
df.describe()
```
- `read_csv` — loads the dataset into a DataFrame.
- `head()` — quick peek at the first 5 rows.
- `info()` — column types + non-null counts (spot missing values fast).
- `describe()` — summary stats (mean, std, min/max) for numeric columns.

---

## 2. Preprocessing

```python
df.isnull().sum()
```
- Count missing values per column.

```python
df[col].fillna(df[col].median(), inplace=True)   # numerical
df[col].fillna(df[col].mode()[0], inplace=True)   # categorical
```
- Fill missing values — median for numeric (robust to outliers), mode for categorical.

```python
from sklearn.preprocessing import LabelEncoder, OneHotEncoder
le = LabelEncoder()
df[col] = le.fit_transform(df[col])
```
- Converts categorical text labels into numbers. `LabelEncoder` for ordinal/binary categories; use `OneHotEncoder` / `pd.get_dummies()` instead when categories have no inherent order.

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)
```
- Scales features to mean 0, std 1. Important for distance-based or gradient-based models (Logistic/Linear Regression, SVM, KNN); not required for tree-based models (Decision Tree, Random Forest).

```python
df[col] = df[col].clip(*df[col].quantile([0.05, 0.95]))
```
- Caps extreme outliers to the 5th/95th percentile range.

---

## 3. Train-Test Split

```python
from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)
```
- Splits data into training and test sets. `test_size=0.2` = 20% held out for testing. `random_state` fixes the split so it's reproducible.

---

## 4. Model Training — Regression

*(Target variable is continuous, e.g. price, income.)*

```python
from sklearn.linear_model import LinearRegression
model = LinearRegression()
model.fit(X_train, y_train)
```
- **Linear Regression** — fits a straight-line relationship between features and target by minimizing squared error. Simple and interpretable, but assumes linear relationships.

```python
from sklearn.linear_model import Ridge
model = Ridge(alpha=1.0)
model.fit(X_train, y_train)
```
- **Ridge Regression** — linear regression with an L2 penalty that shrinks coefficients to reduce overfitting. Good when features are correlated.

```python
from sklearn.linear_model import Lasso
model = Lasso(alpha=0.1)
model.fit(X_train, y_train)
```
- **Lasso Regression** — like Ridge, but uses an L1 penalty that can shrink some coefficients to exactly zero. Useful for automatic feature selection.

```python
from sklearn.tree import DecisionTreeRegressor
model = DecisionTreeRegressor(max_depth=5)
model.fit(X_train, y_train)
```
- **Decision Tree Regressor** — splits data into regions based on feature thresholds. Captures non-linear patterns but can overfit if too deep.

```python
from sklearn.ensemble import RandomForestRegressor
model = RandomForestRegressor(n_estimators=200)
model.fit(X_train, y_train)
```
- **Random Forest Regressor** — trains many decision trees on random subsets and averages their predictions. More robust and accurate than a single tree.

```python
from sklearn.svm import SVR
model = SVR(kernel='rbf')
model.fit(X_train, y_train)
```
- **Support Vector Regressor** — fits a function within a margin of tolerance around the data. Works well on smaller, complex datasets but scales poorly to large ones.

---

## 5. Model Training — Classification

*(Target variable is categorical, e.g. yes/no, class label.)*

```python
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(X_train, y_train)
```
- **Logistic Regression** — despite the name, a classification model; predicts class probabilities using a sigmoid function on a linear combination of features. Fast, interpretable baseline.

```python
from sklearn.tree import DecisionTreeClassifier
model = DecisionTreeClassifier(max_depth=5)
model.fit(X_train, y_train)
```
- **Decision Tree Classifier** — splits data by feature thresholds to separate classes. Easy to interpret and visualize, but prone to overfitting.

```python
from sklearn.ensemble import RandomForestClassifier
model = RandomForestClassifier(n_estimators=200)
model.fit(X_train, y_train)
```
- **Random Forest Classifier** — an ensemble of decision trees voting on the class. Reduces overfitting compared to a single tree and generally performs well out of the box.

```python
from sklearn.svm import SVC
model = SVC(kernel='rbf', probability=True)
model.fit(X_train, y_train)
```
- **Support Vector Classifier** — finds the boundary (hyperplane) that best separates classes with maximum margin. Effective in high-dimensional spaces.

```python
from sklearn.neighbors import KNeighborsClassifier
model = KNeighborsClassifier(n_neighbors=5)
model.fit(X_train, y_train)
```
- **K-Nearest Neighbors** — classifies a point based on the majority class among its `k` nearest neighbors. Simple, no real "training" step, but slow to predict on large datasets.

```python
from sklearn.naive_bayes import GaussianNB
model = GaussianNB()
model.fit(X_train, y_train)
```
- **Naive Bayes** — applies Bayes' theorem assuming features are independent. Fast, works well for text/high-dimensional data despite the "naive" independence assumption.

---

## 6. Predictions

```python
y_pred = model.predict(X_test)
```
- Generates predictions on the held-out test set — used for both regression and classification.

```python
y_proba = model.predict_proba(X_test)
```
- Classification only: returns predicted probabilities per class instead of hard labels (needed for ROC/AUC).

---

## 7. Evaluation — Regression

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

mae = mean_absolute_error(y_test, y_pred)
mse = mean_squared_error(y_test, y_pred)
rmse = mean_squared_error(y_test, y_pred, squared=False)
r2 = r2_score(y_test, y_pred)
```
- `MAE` — average absolute difference between predicted and actual values (same units as target, easy to interpret).
- `MSE` — average squared difference; penalizes larger errors more heavily than MAE.
- `RMSE` — square root of MSE; back in original units, still penalizes large errors.
- `R² score` — proportion of variance in the target explained by the model (1.0 = perfect, 0 = no better than predicting the mean).

---

## 8. Evaluation — Classification

```python
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score, roc_auc_score, confusion_matrix

accuracy = accuracy_score(y_test, y_pred)
precision = precision_score(y_test, y_pred)
recall = recall_score(y_test, y_pred)
f1 = f1_score(y_test, y_pred)
auc = roc_auc_score(y_test, y_proba[:, 1])
cm = confusion_matrix(y_test, y_pred)
```
- `Accuracy` — % of correct predictions overall. Misleading on imbalanced classes.
- `Precision` — of predicted positives, how many were actually positive (matters when false positives are costly).
- `Recall` — of actual positives, how many were correctly caught (matters when false negatives are costly).
- `F1-score` — harmonic mean of precision and recall; balances both.
- `ROC-AUC` — measures ability to distinguish classes across all thresholds (1.0 = perfect, 0.5 = random guessing).
- `Confusion matrix` — table of true/false positives/negatives — the raw breakdown behind the metrics above.

---

## 9. Hyperparameter Tuning (Generic Pattern)

```python
from sklearn.model_selection import GridSearchCV

grid = GridSearchCV(estimator=model, param_grid=param_grid, cv=5, scoring='accuracy')
grid.fit(X_train, y_train)

best_model = grid.best_estimator_
best_params = grid.best_params_
```
- `GridSearchCV` — exhaustively tries every combination in `param_grid` using cross-validation (`cv=5` = 5-fold), keeping the best-scoring combo.
- `RandomizedSearchCV` (alternative) — samples a fixed number of random combinations instead of all of them; faster for large search spaces.

---

## 10. Save the Model

```python
import joblib
joblib.dump(model, "model.pkl")
loaded_model = joblib.load("model.pkl")
```
- `joblib.dump` — serializes the trained model to disk (preferred over `pickle` for sklearn models with large numpy arrays).
- `joblib.load` — reloads a saved model for later use without retraining.

```python
import mlflow.sklearn
mlflow.sklearn.log_model(model, name="model_name")
```
- Alternative: log the model via MLflow instead (see the MLflow notes file) — adds versioning, tracking, and easy serving on top of plain saving.

---

## 11. Summary — Hyperparameters, Loss Functions & Gradient Descent

### Key hyperparameters per model

**Linear / Ridge / Lasso Regression**
- `alpha` — controls how strongly the model penalizes large coefficients; higher values mean simpler models, less overfitting.

**Logistic Regression**
- `C` — inverse of regularization strength; lower values apply stronger regularization (simpler model).
- `penalty` — which regularization type to apply (`l1` shrinks some coefficients to zero, `l2` shrinks all smoothly).
- `solver` — the optimization algorithm used to find the best coefficients (different solvers support different penalties).

**Decision Tree (Regressor/Classifier)**
- `max_depth` — maximum number of splits from root to leaf; limits how complex/deep the tree can grow.
- `min_samples_split` — minimum number of samples a node must have before it's allowed to split further.
- `min_samples_leaf` — minimum number of samples required to keep a leaf node (prevents overly specific splits).
- `criterion` — the metric used to decide the best split at each node (`gini`/`entropy` for classification, `squared_error` for regression).

**Random Forest (Regressor/Classifier)**
- `n_estimators` — number of individual trees built in the forest; more trees generally means more stable predictions.
- `max_depth` — maximum depth allowed for each individual tree in the forest.
- `max_leaf_nodes` — caps the total number of leaf nodes per tree, limiting tree complexity.
- `criterion` — the split-quality metric each tree uses (same idea as in a single Decision Tree).
- `max_features` — number of features considered when looking for the best split at each node.

**SVM (SVR/SVC)**
- `C` — controls the trade-off between a wider margin and correctly classifying every training point; lower values allow more margin violations.
- `kernel` — the function used to transform data into a higher-dimensional space to find a separating boundary (`linear`, `rbf`, `poly`).
- `gamma` — controls how far the influence of a single training point reaches; higher values make the boundary fit training points more tightly.

**KNN**
- `n_neighbors` — number of nearest data points considered when voting on a prediction.
- `weights` — whether all neighbors count equally (`uniform`) or closer neighbors count more (`distance`).
- `metric` — the distance formula used to measure "closeness" between points (e.g. Euclidean).

**Naive Bayes**
- `var_smoothing` — a small value added to variance estimates to avoid numerical instability; rarely needs tuning since the model has few parameters.

### Loss functions (what each model minimizes during training)

- **Linear/Ridge/Lasso Regression** — Mean Squared Error (Ridge/Lasso add an L2/L1 penalty term on top).
- **Logistic Regression** — Log Loss (a.k.a. Binary Cross-Entropy) — penalizes confident wrong predictions heavily.
- **Decision Trees** — not a smooth loss function; splits chosen by impurity measures like Gini impurity or Entropy (classification) or variance reduction (regression).
- **Random Forest** — same per-tree impurity criteria, aggregated across trees (no single global loss being minimized).
- **SVM** — Hinge Loss (classification) — penalizes points inside or on the wrong side of the margin.
- **KNN / Naive Bayes** — no loss function / gradient optimization involved; KNN is instance-based (no real training step), Naive Bayes fits distributions directly from data via probability estimation.

### Gradient Descent (in brief)

- An iterative optimization algorithm used to minimize a loss function — it repeatedly adjusts model parameters in the direction that reduces the loss the fastest (the negative gradient), scaled by a **learning rate**.
- Used directly by: Linear/Ridge/Lasso Regression (when solved iteratively rather than via closed-form), Logistic Regression, and neural networks.
- **Not** used by: Decision Trees, Random Forest, KNN, Naive Bayes — these are optimized (or not optimized at all) by other means (impurity splits, distance lookup, or direct probability calculation).
- Common variants: **Batch GD** (uses the whole dataset per step — stable but slow), **Stochastic GD** (one sample per step — fast but noisy), **Mini-batch GD** (small batches — the common middle ground, used in most deep learning training).
