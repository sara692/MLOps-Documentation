# Prometheus + Docker Notes: House Price Prediction API

## 1. Big picture

```
Browser / client ──► API container (FastAPI, port 8000)
                        │  exposes /metrics
                        ▼
Prometheus container (port 9090)  ── scrapes api:8000/metrics every 5s
                        │
                        ▼
              stores time series in volume prometheus_data
```

- **Your API** exposes a text page at `/metrics`.
- **Prometheus** *pulls* (scrapes) that page on a schedule and stores each value with a timestamp.
- **You** query the stored data with **PromQL** in the Prometheus UI (`http://localhost:9090`).
- Prometheus is a **pull-based** system. The app never sends data to it.

---

## 2. Project changes

### 2.1 Install packages

```powershell
uv add prometheus-fastapi-instrumentator prometheus-client
```

### 2.2 `pyproject.toml` fix (needed for `uv add` to work)

`uv` tried to build the project as a package named `house_price_prediction`, but the real module is `house_price`.

```toml
[tool.uv.build-backend]
module-name = "house_price"
```

(This applies when `build-backend = "uv_build"`. For hatchling use `[tool.hatch.build.targets.wheel] packages = ["src/house_price"]`.)

### 2.3 `src/house_price/main.py`

```python
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator
from house_price.config import settings
from house_price.views import api

app = FastAPI(title=settings.app_name, version=settings.app_version)
app.include_router(api.base_router)
app.include_router(api.data_router)

# Automatic HTTP metrics + the /metrics endpoint.
# excluded_handlers hides Prometheus' own scrapes from the request stats.
Instrumentator(excluded_handlers=["/metrics"]).instrument(app).expose(app)
```

### 2.4 `src/house_price/metrics.py` (custom ML metrics)

```python
from prometheus_client import Counter, Histogram, Gauge

PREDICTIONS_TOTAL = Counter(
    "house_price_predictions_total", "Total prediction requests", ["status"]
)
PREDICTED_PRICE = Histogram(
    "house_price_predicted_lacs", "Predicted price in lacs",
    buckets=(10, 25, 50, 75, 100, 150, 250, 500, 1000),
)
PREDICTION_SECONDS = Histogram(
    "house_price_prediction_seconds", "Model inference time in seconds"
)
MODEL_LOADED = Gauge("house_price_model_loaded", "1 if model artifacts are loaded")
```

### 2.5 Using them in `views/api.py`

```python
from house_price.metrics import PREDICTIONS_TOTAL, PREDICTED_PRICE, PREDICTION_SECONDS

@data_router.post("/predict", response_model=PredictionResult)
def predict_price(input_data: InputDataSchema):
    try:
        with PREDICTION_SECONDS.time():
            result = get_controller().predict_price(input_data)
        PREDICTED_PRICE.observe(result.predicted_price)
        PREDICTIONS_TOTAL.labels(status="success").inc()
        return result
    except FileNotFoundError as e:
        PREDICTIONS_TOTAL.labels(status="error").inc()
        raise HTTPException(status_code=503, detail=str(e))
    except ValueError as e:
        PREDICTIONS_TOTAL.labels(status="error").inc()
        raise HTTPException(status_code=422, detail=str(e))
```

---

## 3. Docker setup

### 3.1 Layout

```
House_Price_Prediction/
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── prometheus/prometheus.yml
├── Artifacts/            (your .pkl files, mounted, not copied)
├── src/house_price/...
├── pyproject.toml
└── uv.lock               (must be up to date)
```

### 3.2 `Dockerfile`

```dockerfile
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

WORKDIR /app
ENV PYTHONPATH=/app/src \
    PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

COPY src ./src

EXPOSE 8000
CMD ["uvicorn", "house_price.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Why it is written this way:
- Dependencies are installed **before** the code is copied, so Docker caches that layer and rebuilds are fast.
- `--no-install-project` plus `PYTHONPATH=/app/src` keeps `config.py` at `/app/src/house_price/config.py`. Its `parents[2]` then resolves to `/app`, where `Artifacts/` is mounted.
- Use the same Python version as your local one (`.python-version`) so the pickled model loads without version issues.

### 3.3 `.dockerignore`

```
.venv
.git
mlruns
mlflow.db
Data
__pycache__
*.pyc
```

### 3.4 `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: house-price-api
    metrics_path: /metrics
    static_configs:
      - targets: ["api:8000"]
```

The target is `api:8000`, not `localhost:8000`. Inside a container, `localhost` means that container itself. Compose puts all services on one network, and each service name works as a hostname.

### 3.5 `docker-compose.yml`

```yaml
services:
  api:
    build: .
    container_name: house-price-api
    ports:
      - "8000:8000"
    volumes:
      - ./Artifacts:/app/Artifacts:ro
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    depends_on:
      - api
    restart: unless-stopped

volumes:
  prometheus_data:
```

### 3.6 Compose file explained

| Key | Meaning |
|---|---|
| `build: .` | Build the image from the `Dockerfile` in this folder |
| `image:` | Use a ready-made image from Docker Hub |
| `ports: "8000:8000"` | `HOST:CONTAINER`. Your PC's port 8000 goes to the container's port 8000 |
| `restart: unless-stopped` | Restart after a crash or a Docker restart, unless you stopped it manually |
| `depends_on` | Controls start order only. It does not wait for the app to be ready |

### 3.7 Volumes explained

Format: `WHAT_IS_OUTSIDE : WHERE_IT_APPEARS_INSIDE : OPTIONS`

| Type | Example | Left side | Managed by | Purpose |
|---|---|---|---|---|
| Bind mount | `./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro` | A real path on your PC | You | Give the container **your settings/files** |
| Named volume | `prometheus_data:/prometheus` | Just a name | Docker | Keep the container's **data** after it is deleted |

- `:ro` means read-only. The container cannot change the file.
- A named volume must also be **declared** in the top-level `volumes:` block. That block is outside `services:`, so the storage belongs to the project, not to one container.
- Bind mounts need no declaration, because the path already exists on your PC.
- `docker compose down` keeps named volumes. `docker compose down -v` deletes them, and the Prometheus history is lost.
- `./Artifacts` is mounted rather than copied. After retraining, run `docker compose restart api`. No rebuild is needed.

---

## 4. Commands

| Task | Command |
|---|---|
| Build and run (foreground) | `docker compose up --build` |
| Build and run (background) | `docker compose up -d --build` |
| Follow API logs | `docker compose logs -f api` |
| Show containers | `docker compose ps` |
| Stop and remove containers | `docker compose down` |
| Also delete metric history | `docker compose down -v` |
| Restart after retraining | `docker compose restart api` |
| Restart after editing `prometheus.yml` | `docker compose restart prometheus` |
| Rebuild after code change | `docker compose up -d --build` |

- In the foreground, stop with **Ctrl + C**, or run `docker compose down` from a second terminal.
- URLs:
  - API docs: `http://localhost:8000/docs`
  - Raw metrics: `http://localhost:8000/metrics`
  - Targets: `http://localhost:9090/targets` (should be **UP**)
  - Query UI: `http://localhost:9090/graph`

---

## 5. Problems we hit and their fixes

| Symptom | Cause | Fix |
|---|---|---|
| `Could not import module "main"` | `main.py` is in `src/house_price/`, not the project root | `uv run uvicorn house_price.main:app --reload --app-dir src` |
| `predict.py` syntax error | `def predict(self, input_data: ):` | Use `input_data: dict` |
| `/health` crashed | `APP_NAME` and `APP_VERSION` were undefined | Added `app_name` and `app_version` to `Settings` |
| 422 `float_type`, `array([60.31])` | `predict()` returned a numpy array | `return float(self.model.predict(X)[0])` |
| 422 "Expected 2D array, got 1D" | `[0]` applied to the input before `model.predict` | Take `[0]` **after** predicting, and keep the input shape `(1, n)` |
| 422 `json_invalid` | Trailing comma before `}` in the request body | Remove the last comma. JSON does not allow it |
| 422 missing fields | `PHOSPHORUS`, `RAINFALL`, `LOAN` were left out | All schema fields are required |
| Wrong prices | `LATITUDE` and `LONGITUDE` values swapped | Chennai is lat `12.85`, lon `80.21`. KMeans uses them to pick the location tier |
| `uv add` failed: `Expected a Python module at src\house_price_prediction` | Build backend guessed the wrong module name | Set `module-name = "house_price"` (section 2.2) |
| `GET /metrics 404` in Docker | `main.py` was not edited, or the image was not rebuilt | Edit `main.py`, run `uv add ...`, then `docker compose up --build` |
| MLflow UI is empty | It looked at `./mlruns`, not the SQLite DB | `uv run mlflow ui --backend-store-uri sqlite:///mlflow.db` |

**Key rule:** the code is copied into the image at build time. Editing files on your PC does not change a running container until you rebuild. The exceptions are the mounted files (`Artifacts/`, `prometheus.yml`).

---

## 6. Why metrics exist when nobody uses the app

You did not define them, but they still appear:

1. **Prometheus is a client.** It hits `/metrics` every 5 seconds. The Instrumentator counts these as normal requests (`http_requests_total{handler="/metrics"}`). Fix: `excluded_handlers=["/metrics"]`.
2. **Default process collectors** in `prometheus_client` need no traffic:
   - `process_*`: CPU, memory, open files, start time
   - `python_gc_*`, `python_info`
3. **Instrumentator metrics**: `http_requests_total`, `http_request_duration_seconds`, request and response sizes.
4. **Metrics Prometheus creates itself for each target**: `up`, `scrape_duration_seconds`, `scrape_samples_scraped`.
5. **Your own metrics**: histograms and unlabeled metrics appear at 0 immediately. A **labeled** counter (like `house_price_predictions_total{status=...}`) only appears after `.labels(...)` is first used, meaning after the first prediction.

`up{job="house-price-api"}` is the most useful health signal: 1 means the last scrape worked, 0 means it failed.

---

## 7. Metric types

Every metric has a **name**, optional **labels** (`{status="success"}`) and a **numeric value**. There are four types.

### 7.1 Counter

- **What:** a number that only goes **up** (it resets to 0 when the app restarts).
- **Use for:** requests, errors, predictions, items processed.
- **Never query the raw value.** Use `rate()` or `increase()`, which also handle restarts.

```python
from prometheus_client import Counter
PREDICTIONS_TOTAL = Counter("house_price_predictions_total", "Predictions", ["status"])
PREDICTIONS_TOTAL.labels(status="success").inc()      # +1
PREDICTIONS_TOTAL.labels(status="error").inc(2)       # +2
```

```promql
# predictions per second, averaged over the last 5 minutes
rate(house_price_predictions_total[5m])

# number of predictions in the last hour
increase(house_price_predictions_total[1h])

# error percentage
100 * sum(rate(house_price_predictions_total{status="error"}[5m]))
    / sum(rate(house_price_predictions_total[5m]))
```

### 7.2 Gauge

- **What:** a number that can go **up and down**.
- **Use for:** current memory, queue size, temperature, "is the model loaded" (0/1), requests in progress.
- **Query the raw value directly.** Do not use `rate()` on gauges.

```python
from prometheus_client import Gauge
MODEL_LOADED = Gauge("house_price_model_loaded", "1 if model loaded")
MODEL_LOADED.set(1)
# also: .inc(), .dec(), .set_to_current_time()
```

```promql
house_price_model_loaded                       # current value
process_resident_memory_bytes / 1024 / 1024    # memory in MB
avg_over_time(process_resident_memory_bytes[10m])   # average of the last 10 min
max_over_time(process_resident_memory_bytes[1h])    # peak in the last hour
delta(process_resident_memory_bytes[1h])       # change over an hour (gauges only)
```

### 7.3 Histogram

- **What:** counts observations in **buckets** (for example "how many predictions were ≤ 50 lacs"). It is stored as three kinds of series:
  - `<name>_bucket{le="X"}`: cumulative count of values ≤ X (including `le="+Inf"`)
  - `<name>_sum`: sum of all observed values
  - `<name>_count`: number of observations
- **Use for:** latency, response size, predicted price. It answers "what is the 95th percentile?"
- **Buckets are chosen by you.** Pick them around the range you expect. Percentiles are estimated, so they are only as accurate as the buckets.
- **Can be aggregated** across several app instances.

```python
from prometheus_client import Histogram
PREDICTED_PRICE = Histogram("house_price_predicted_lacs", "Price", buckets=(10, 25, 50, 100, 250, 500))
PREDICTED_PRICE.observe(60.3)

PREDICTION_SECONDS = Histogram("house_price_prediction_seconds", "Inference time")
with PREDICTION_SECONDS.time():
    ...   # code to time
```

```promql
# 95th percentile inference time
histogram_quantile(0.95, rate(house_price_prediction_seconds_bucket[5m]))

# median predicted price
histogram_quantile(0.5, rate(house_price_predicted_lacs_bucket[5m]))

# average predicted price
rate(house_price_predicted_lacs_sum[5m]) / rate(house_price_predicted_lacs_count[5m])

# how many predictions were at most 50 lacs
house_price_predicted_lacs_bucket{le="50"}
```

If a query for a multi-series histogram returns odd results, aggregate by `le`:
`histogram_quantile(0.95, sum by (le) (rate(x_bucket[5m])))`.

Note: the Instrumentator's default `http_request_duration_seconds` has only a few buckets (around 0.1, 0.5 and 1 s), so percentiles from it are rough.

### 7.4 Summary

- **What:** like a histogram, but the **app calculates quantiles itself**. Series: `<name>{quantile="0.9"}`, `<name>_sum`, `<name>_count`.
- **In the Python client**, a `Summary` only exposes `_count` and `_sum`. It does **not** produce quantiles. So it is good for averages only.
- **Histogram vs Summary:** prefer a histogram. Summary quantiles **cannot be aggregated** across instances, and you cannot change them later.

```python
from prometheus_client import Summary
REQUEST_TIME = Summary("request_processing_seconds", "Time spent processing")

@REQUEST_TIME.time()
def process(): ...
```

```promql
# average duration
rate(request_processing_seconds_sum[5m]) / rate(request_processing_seconds_count[5m])
```

### 7.5 Comparison

| Type | Goes down? | Query with | Typical use |
|---|---|---|---|
| Counter | No | `rate()`, `increase()` | Requests, errors, predictions |
| Gauge | Yes | Raw value, `avg_over_time`, `delta` | Memory, model loaded, queue size |
| Histogram | Buckets only grow | `histogram_quantile(rate(..._bucket))` | Latency, predicted price, percentiles |
| Summary | Count and sum only grow | `rate(_sum)/rate(_count)` | Averages only (in Python) |

---

## 8. PromQL query notes

### 8.1 Data types of query results

| Type | Meaning | Example |
|---|---|---|
| **Instant vector** | One value per series **now** | `up` |
| **Range vector** | Several values per series over a time window | `up[5m]` |
| **Scalar** | A single number | `3.14` |

Range vectors cannot be graphed directly. Wrap them in a function such as `rate()`, `increase()`, `avg_over_time()`.

### 8.2 Selecting series and label matchers

```promql
up                                    # all series named up
up{job="house-price-api"}             # exact match
http_requests_total{status!="2xx"}    # not equal
http_requests_total{handler=~"/api.*"}    # regex match
http_requests_total{handler!~"/metrics|/docs"}   # regex does not match
```

Operators: `=`  `!=`  `=~`  `!~`

### 8.3 Time windows and offsets

```promql
http_requests_total[5m]               # last 5 minutes of samples (range vector)
rate(http_requests_total[5m])         # per-second rate over 5 minutes
http_requests_total offset 1h         # value 1 hour ago
rate(http_requests_total[5m]) / rate(http_requests_total[5m] offset 1d)   # vs same time yesterday
```

Units: `s` `m` `h` `d` `w`. The rate window should be at least about 4× the scrape interval (with a 5s scrape, `[30s]` is a sensible minimum).

### 8.4 Counter functions

| Function | Result |
|---|---|
| `rate(x[5m])` | Average per-second increase over the window (smooth, best for graphs and alerts) |
| `irate(x[5m])` | Per-second rate between the **last two** samples (spiky, good for zoomed views) |
| `increase(x[1h])` | Total increase over the window |

All three handle counter resets caused by restarts.

### 8.5 Aggregation operators

Combine many series into fewer.

```promql
sum(rate(http_requests_total[5m]))                   # total across everything
sum by (handler) (rate(http_requests_total[5m]))     # keep only the handler label
sum without (instance) (rate(http_requests_total[5m]))   # drop instance, keep the rest
avg, min, max, count, topk(3, ...), bottomk(3, ...)
```

- `by (...)` keeps the listed labels.
- `without (...)` drops the listed labels.

### 8.6 Arithmetic and comparison

```promql
process_resident_memory_bytes / 1024 / 1024          # bytes to MB
house_price_model_loaded == 0                        # only series where the model is NOT loaded
rate(house_price_predictions_total{status="error"}[5m]) > 0.1   # filter: errors above 0.1/s
```

Comparisons filter series. Add `bool` to return 0/1 instead: `x > bool 5`.

### 8.7 Useful queries for this project

| Question | Query |
|---|---|
| Is the API up? | `up{job="house-price-api"}` |
| Requests per second by endpoint | `sum by (handler) (rate(http_requests_total[1m]))` |
| Server errors (5xx) per second | `sum(rate(http_requests_total{status="5xx"}[5m]))` |
| Successful predictions per minute | `increase(house_price_predictions_total{status="success"}[1m])` |
| Prediction error rate (%) | `100 * sum(rate(house_price_predictions_total{status="error"}[5m])) / sum(rate(house_price_predictions_total[5m]))` |
| p95 inference time | `histogram_quantile(0.95, rate(house_price_prediction_seconds_bucket[5m]))` |
| Average predicted price | `rate(house_price_predicted_lacs_sum[5m]) / rate(house_price_predicted_lacs_count[5m])` |
| Memory (MB) | `process_resident_memory_bytes / 1024 / 1024` |
| Is the model loaded? | `house_price_model_loaded` |
| Real traffic only (no scrapes) | `http_requests_total{handler!="/metrics"}` |

Note: the Instrumentator groups status codes by default, so labels look like `2xx`, `4xx`, `5xx`.

### 8.8 Common mistakes

- Using a **counter raw value** (`http_requests_total`) instead of `rate()`. The raw number just keeps growing.
- Using `rate()` on a **gauge**. Use the raw value, `delta()` or `avg_over_time()`.
- Forgetting the range window: `rate(x)` is invalid, `rate(x[5m])` is correct.
- A labeled counter with no series yet. Send at least one request first.
- Counters and histograms **reset on restart** (with `--reload`, on every code save). That is normal.
- Multiple uvicorn workers need `prometheus_client` multiprocess mode. The current setup assumes a single process.

### 8.9 Example alert rules (for later, with Alertmanager)

```yaml
groups:
  - name: house-price
    rules:
      - alert: ApiDown
        expr: up{job="house-price-api"} == 0
        for: 1m
      - alert: HighPredictionErrorRate
        expr: |
          sum(rate(house_price_predictions_total{status="error"}[5m]))
            / sum(rate(house_price_predictions_total[5m])) > 0.1
        for: 5m
      - alert: SlowPredictions
        expr: histogram_quantile(0.95, rate(house_price_prediction_seconds_bucket[5m])) > 1
        for: 5m
```

---

## 9. Next steps

- Set `MODEL_LOADED.set(1)` when artifacts load successfully.
- Add **Grafana** to the same compose file for dashboards (data source URL: `http://prometheus:9090`).
- Add **Alertmanager** and the alert rules above.
- Pin image versions (for example `prom/prometheus:v2.55.0`) instead of `latest` for stable deployments.
