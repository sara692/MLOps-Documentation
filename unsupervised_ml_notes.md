# Unsupervised Machine Learning — Pipeline Notes

End-to-end reference: loading data → preprocessing → training → evaluation → saving. There's no target/label here — the model finds structure in the data on its own. Split into **Clustering** (grouping similar points) and **Dimensionality Reduction** (compressing features) since they solve different problems and use different evaluation approaches.

---

## 1. Load the Data

```python
import pandas as pd
df = pd.read_csv("data.csv")
df.head()
df.info()
```
- Same as supervised learning — load and inspect the dataset first.
- **Key difference:** there's no target column to separate out — all (or most) columns become input features.

---

## 2. Preprocessing

```python
df.isnull().sum()
df[col].fillna(df[col].median(), inplace=True)
```
- Handle missing values same as usual (see the Preprocessing & EDA notes file for full detail).

```python
from sklearn.preprocessing import LabelEncoder
le = LabelEncoder()
df[col] = le.fit_transform(df[col])
```
- Encode any categorical columns into numbers — unsupervised algorithms need purely numeric input.

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_scaled = scaler.fit_transform(df)
```
- **Scaling is far more critical here than in supervised learning.** Almost all clustering and dimensionality-reduction algorithms rely on distance calculations — an unscaled column with large numbers (e.g. income in the thousands) will silently dominate a column like age (0–100), regardless of which one actually matters more.

---

## 3. Model Training — Clustering

*(Groups similar data points together without predefined labels.)*

```python
from sklearn.cluster import KMeans
model = KMeans(n_clusters=3, random_state=42)
model.fit(X_scaled)
labels = model.labels_
```
- **K-Means** — partitions data into `k` clusters by repeatedly assigning points to the nearest cluster center and recalculating centers. Fast and widely used, but you must choose `k` in advance and it assumes roughly round, similarly-sized clusters.

```python
from sklearn.cluster import AgglomerativeClustering
model = AgglomerativeClustering(n_clusters=3)
labels = model.fit_predict(X_scaled)
```
- **Hierarchical (Agglomerative) Clustering** — starts with every point as its own cluster and repeatedly merges the closest pairs until reaching `k` clusters. Produces a dendrogram (tree) you can inspect, and doesn't assume round clusters, but is slower on large datasets.

```python
from sklearn.cluster import DBSCAN
model = DBSCAN(eps=0.5, min_samples=5)
labels = model.fit_predict(X_scaled)
```
- **DBSCAN** — groups points that are closely packed together and marks isolated points as noise/outliers (`label = -1`). Doesn't require specifying the number of clusters upfront, and naturally handles irregular cluster shapes and outlier detection together.

```python
from sklearn.mixture import GaussianMixture
model = GaussianMixture(n_components=3, random_state=42)
labels = model.fit_predict(X_scaled)
```
- **Gaussian Mixture Model (GMM)** — assumes data comes from a mix of overlapping bell-curve (Gaussian) distributions and assigns each point a probability of belonging to each cluster (soft clustering), rather than a hard single assignment like K-Means.

---

## 4. Model Training — Dimensionality Reduction

*(Compresses many features into fewer while preserving as much information/structure as possible — useful for visualization or speeding up downstream models.)*

```python
from sklearn.decomposition import PCA
pca = PCA(n_components=2)
X_reduced = pca.fit_transform(X_scaled)
```
- **PCA (Principal Component Analysis)** — finds new axes (principal components) that capture the maximum variance in the data, letting you keep just the top few and discard the rest. Fast, linear, and the standard first choice for reducing dimensions before modeling or plotting.

```python
from sklearn.manifold import TSNE
tsne = TSNE(n_components=2, perplexity=30, random_state=42)
X_embedded = tsne.fit_transform(X_scaled)
```
- **t-SNE** — a non-linear technique that's especially good at preserving *local* structure (nearby points stay nearby), making it popular purely for 2D/3D visualization of clusters. Not meant to be used as input for further modeling — it's for looking at data, not feeding it forward.

```python
import umap
reducer = umap.UMAP(n_components=2, random_state=42)
X_embedded = reducer.fit_transform(X_scaled)
```
- **UMAP** — similar goal to t-SNE (great for visualization) but generally faster and better at preserving some global structure alongside local structure. Increasingly used as a t-SNE alternative. (Requires separately installing the `umap-learn` package.)

---

## 5. Choosing the Number of Clusters

Unlike supervised learning, you often don't know `k` in advance — these techniques help estimate it.

```python
inertia = []
for k in range(1, 11):
    km = KMeans(n_clusters=k, random_state=42).fit(X_scaled)
    inertia.append(km.inertia_)
# then plot k vs inertia
```
- **Elbow Method** — plot inertia (see §6) against different values of `k`; look for the "elbow" point where adding more clusters stops giving a meaningful improvement.

```python
from scipy.cluster.hierarchy import dendrogram, linkage
linked = linkage(X_scaled, method='ward')
dendrogram(linked)
```
- **Dendrogram** — a tree diagram from hierarchical clustering; the number of clusters can be chosen by visually picking where to "cut" the tree (longest vertical line you can cross without crossing a horizontal merge line).

---

## 6. Evaluation

Since there's no ground-truth label to compare against, evaluation relies on measuring how well-separated and internally consistent the clusters are.

```python
model.inertia_
```
- **Inertia (K-Means only)** — sum of squared distances from each point to its assigned cluster center. Lower is generally better (tighter clusters), but always decreases as `k` increases — that's why it's used with the elbow method rather than compared directly across different `k`.

```python
from sklearn.metrics import silhouette_score
score = silhouette_score(X_scaled, labels)
```
- **Silhouette Score** — measures how similar a point is to its own cluster vs. other clusters, averaged across all points. Ranges from -1 to 1; higher is better (well-separated clusters), near 0 means overlapping clusters, negative means points may be in the wrong cluster.

```python
from sklearn.metrics import davies_bouldin_score
score = davies_bouldin_score(X_scaled, labels)
```
- **Davies-Bouldin Index** — measures average similarity between each cluster and its most similar other cluster. Lower is better (0 = best possible, clusters are maximally distinct).

```python
pca.explained_variance_ratio_
```
- **Explained Variance Ratio (PCA only)** — how much of the original data's variance each principal component captures. Summing the first few components tells you how much information you kept after reducing dimensions (e.g. `0.95` = you retained 95% of the original information).

---

## 7. Save the Model

```python
import joblib
joblib.dump(model, "cluster_model.pkl")
loaded_model = joblib.load("cluster_model.pkl")
```
- Same pattern as supervised learning — serialize the fitted model (and the scaler too, if used, since you'll need it to transform new data the same way later).

```python
joblib.dump(scaler, "scaler.pkl")
```
- Important: save the **scaler** alongside the model — new/incoming data must be scaled using the exact same fitted scaler before being passed to the model.

---

## 8. Summary — Hyperparameters, Loss/Objective & Optimization

*(Default and Range values below are for scikit-learn's implementations.)*

### Key hyperparameters per model

**K-Means**
- `n_clusters` — the number `k` of clusters to partition the data into.
  **Default:** `8` · **Range:** positive integer, typically chosen via the elbow method or domain knowledge
- `init` — how initial cluster centers are chosen (`'k-means++'` spreads them out smartly; `'random'` picks randomly).
  **Default:** `'k-means++'` · **Range:** `'k-means++'`, `'random'`, or a custom array of starting points
- `n_init` — number of times the algorithm runs with different centroid seeds, keeping the best result (reduces the risk of a bad random start).
  **Default:** `'auto'` (currently resolves to `10`) · **Range:** positive integer, or `'auto'`
- `max_iter` — maximum number of iterations for a single run before stopping even if it hasn't fully converged.
  **Default:** `300` · **Range:** positive integer

**Agglomerative (Hierarchical) Clustering**
- `n_clusters` — the number of clusters to form (used to decide where to stop merging).
  **Default:** `2` · **Range:** positive integer, or `None` if using `distance_threshold` instead
- `linkage` — the rule used to decide which clusters are "closest" when deciding what to merge next.
  **Default:** `'ward'` · **Range:** `'ward'`, `'complete'`, `'average'`, `'single'`
- `metric` (formerly `affinity`) — the distance measure used between points.
  **Default:** `'euclidean'` · **Range:** `'euclidean'`, `'manhattan'`, `'cosine'`, and others (note: `'ward'` linkage only supports `'euclidean'`)

**DBSCAN**
- `eps` — the maximum distance between two points for them to be considered neighbors; the core radius the algorithm searches within.
  **Default:** `0.5` · **Range:** positive float — highly dependent on your data's scale, so tuning this is essential
- `min_samples` — the minimum number of neighboring points required for a point to be considered a "core point" (dense region).
  **Default:** `5` · **Range:** positive integer (commonly set to at least the number of dimensions + 1)

**Gaussian Mixture Model**
- `n_components` — the number of Gaussian distributions (clusters) to fit.
  **Default:** `1` · **Range:** positive integer
- `covariance_type` — controls the shape/flexibility each Gaussian cluster is allowed to take.
  **Default:** `'full'` · **Range:** `'full'`, `'tied'`, `'diag'`, `'spherical'`
- `max_iter` — maximum number of iterations of the fitting algorithm (EM algorithm) before stopping.
  **Default:** `100` · **Range:** positive integer

**PCA**
- `n_components` — how many principal components (compressed dimensions) to keep.
  **Default:** `None` (keeps all components) · **Range:** positive integer (a count of dimensions), or a float `(0.0, 1.0)` meaning "keep enough components to explain this much variance"
- `svd_solver` — the underlying algorithm used to compute the decomposition.
  **Default:** `'auto'` · **Range:** `'auto'`, `'full'`, `'arpack'`, `'randomized'`

**t-SNE**
- `n_components` — the number of output dimensions (almost always 2 or 3, for visualization).
  **Default:** `2` · **Range:** positive integer (rarely more than 3, since t-SNE is built for visualization)
- `perplexity` — roughly reflects the number of nearby neighbors considered around each point; balances local vs. global structure.
  **Default:** `30.0` · **Range:** positive float, commonly tuned between `5`–`50` (should be smaller than the number of samples)
- `learning_rate` — step size used during optimization.
  **Default:** `'auto'` · **Range:** `'auto'`, or a positive float (commonly `10`–`1000`)

---

### "Loss"/objective function per model

Unsupervised models don't have a "correct answer" to minimize error against — instead they optimize an internal objective describing what a "good" grouping or compression looks like:

- **K-Means** — minimizes **inertia** (within-cluster sum of squared distances to each centroid) — same value shown in §6.
- **Agglomerative Clustering** — no single global objective being minimized; it greedily merges the closest clusters at each step according to the chosen `linkage` rule.
- **DBSCAN** — no explicit objective function; it's a density-based rule-following algorithm (core points, reachability), not an optimization problem in the traditional sense.
- **Gaussian Mixture Model** — maximizes the **likelihood** that the data was generated by the mixture of Gaussians (via the EM algorithm — see below).
- **PCA** — maximizes the **variance explained** by each successive component (equivalently, minimizes reconstruction error when projecting back to the original space).
- **t-SNE** — minimizes the difference (KL divergence) between how similar points are in the original high-dimensional space vs. the compressed low-dimensional space.

### Gradient Descent — does it apply here?

- **K-Means** — not gradient descent; uses **Lloyd's algorithm**, an iterative "assign points → recompute centers → repeat" loop until centers stop moving.
- **Agglomerative Clustering / DBSCAN** — no optimization loop at all; both are rule-based/greedy procedures, not iterative error-minimization.
- **Gaussian Mixture Model** — uses the **EM (Expectation-Maximization) algorithm**, a different iterative optimization approach (alternating between estimating cluster assignments and updating distribution parameters) — related in spirit to gradient descent but not the same mechanism.
- **PCA** — solved directly via linear algebra (eigendecomposition / SVD), no iterative optimization needed at all.
- **t-SNE** — **does** use actual gradient descent to iteratively minimize the KL divergence mentioned above, making it the one algorithm in this list that behaves most like supervised learning's optimization process.
