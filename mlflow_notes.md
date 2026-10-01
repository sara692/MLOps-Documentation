# MLflow Notes — MLOps Bootcamp

Personal reference for tracking experiments, saving models, and serving them with MLflow.

---

## 1. Setup: Tell MLflow Where to Track Runs

```python
import mlflow
mlflow.set_tracking_uri("http://127.0.0.1:5000")
```

- **What it does:** Points the MLflow client at a tracking server (must already be running — see §7).
- **Optional?** No, if you skip it, MLflow defaults to logging locally into a `mlruns/` folder in your current directory (`file:./mlruns`). Setting it explicitly is only needed when you're using a server (local or remote) instead of local file storage.

```python
mlflow.get_tracking_uri()
```

- **What it does:** Just returns the URI currently in use, as a string. Useful for debugging ("where is this actually logging to?"). Doesn't set anything.

---

## 2. Experiments — Group Your Runs

An **experiment** is a folder-like container that groups related runs together (e.g. all your "Loan_prediction" attempts).

```python
mlflow.create_experiment("Loan_prediction")
```

- **What it does:** Creates a brand-new experiment. **Requires the name argument** — `mlflow.create_experiment()` with nothing inside the parentheses will error out.
- **Optional?** Only needed **once** per experiment name. If it already exists, calling this again will throw an error (`Experiment already exists`). You don't need this if the experiment is already created — skip straight to `set_experiment`.

```python
mlflow.set_experiment("Loan_prediction")
```

- **What it does:** Tells MLflow "log all following runs under this experiment." If the experiment doesn't exist yet, `set_experiment` will **create it automatically** — so in practice, you rarely need `create_experiment` at all.
- **Optional?** No — without this, runs go into the "Default" experiment (id 0), which gets messy fast.

> **Beginner tip:** Just call `mlflow.set_experiment("Loan_prediction")` and skip `create_experiment` entirely. One line does both jobs.

---

## 3. Starting a Run and Logging Data

```python
with mlflow.start_run() as run:
    run_id = run.info.run_id
    ...
```

- **What it does:** Opens a new "run" (one training attempt) inside the current experiment. Everything logged inside the `with` block gets attached to this run. Using `with` auto-closes the run when the block ends — so you generally **don't need `mlflow.end_run()`** at the bottom (see note below).

```python
mlflow.set_tag("run_id", run_id)
```

- **What it does:** Attaches a custom label to the run for your own reference/filtering later (tags are free-form key-value pairs, separate from params/metrics).
- **Optional?** Yes — purely for your own organization. Not required for the model to work.

```python
mlflow.log_params(model.best_params_)
```

- **What it does:** Logs a **dictionary** of hyperparameters at once (e.g. from `GridSearchCV.best_params_`). Use `log_param()` (singular) if logging just one value.
- **Optional?** No — this is how you record what settings produced this run's results.

```python
mlflow.log_metric("Accuracy", accuracy)
```

- **What it does:** Logs a single numeric metric (score, loss, etc.) tied to this run. Call it once per metric name, or repeatedly with a `step` argument to track a metric changing over time (e.g. loss per epoch).

```python
mlflow.log_artifact("plots/ROC_curve.png")
```

- **What it does:** Uploads **one file** (a plot, a CSV, anything) into the run's artifact storage.

```python
mlflow.log_artifacts("plots/")
```

- **What it does:** Uploads an **entire folder** of files at once, preserving the folder structure. **Requires a path argument** — `mlflow.log_artifacts()` with nothing inside will error.
- **When to use which:** `log_artifact` (singular) for one file, `log_artifacts` (plural) for a whole directory. You typically use one or the other, not both, unless you have a single file _and_ a separate folder to log.

```python
mlflow.end_run()
```

- **What it does:** Manually closes the current run.
- **Optional?** Yes, **if** you used the `with mlflow.start_run() as run:` block — the `with` statement closes it automatically when the block exits, even if an error occurs inside. Calling `end_run()` again after that is harmless but redundant. Only needed if you started a run _without_ `with` (e.g. `run = mlflow.start_run()`).

---

## 4. Saving the Model Itself

This is the part that's separate from logging metrics/params — it saves the actual trained model object so it can be reloaded or served later.

```python
mlflow.sklearn.log_model(model, name=name)
```

- **What it does:** Serializes the model and stores it as an artifact of the run, under a subfolder called `name` (e.g. `"RandomForestClassifier"`). This is what creates the `MLmodel`, `model.pkl`, `conda.yaml`, etc. files.

- **Naming note:** the `name` parameter is the modern replacement for the older `artifact_path` parameter (which is now deprecated but still works with a warning).

**What gets saved on disk**, once logged:

```
RandomForestClassifier/
    - conda.yaml        # conda environment spec for reproducing the model's env
    - MLmodel            # metadata file describing the model flavor/format
    - model.pkl           # the actual pickled sklearn model
    - requirements.txt    # pip package requirements
```

---

## 5. Full Corrected Example

Putting it together (assuming `model`, `accuracy`, `f1`, `auc`, and `name` are already defined earlier in your script):

```python
import mlflow

mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("Loan_prediction")   # creates it if it doesn't exist yet

with mlflow.start_run() as run:
    run_id = run.info.run_id
    mlflow.set_tag("run_id", run_id)

    mlflow.log_params(model.best_params_)
    mlflow.log_metric("Mean CV score", model.best_score_)
    mlflow.log_metric("Accuracy", accuracy)
    mlflow.log_metric("f1-score", f1)
    mlflow.log_metric("AUC", auc)

    mlflow.log_artifact("plots/ROC_curve.png")
    mlflow.sklearn.log_model(model.best_estimator_, name=name)
    # no need for mlflow.end_run() — the 'with' block handles it
```

---

## 6. MLflow Projects — Packaging & Reproducible Runs

```bash
mlflow run . --experiment-name Loan_prediction
```

- **What it does:** Runs an MLflow Project (a folder with an `MLproject` file defining entry points and dependencies) from your **current directory** (`.`). Logs the run under the given experiment name.

```bash
#MLproject
name: Loan_prediction

conda_env: conda.yaml

entry_points:
  main:
    command: "python loan_prediction.py"
```

```bash
mlflow run https://github.com/sara/ml-flow-project --experiment-name Loan_prediction
```

- **What it does:** Same idea, but pulls the project directly from a **Git repository** instead of a local folder — useful for reproducing someone else's exact setup.

---

## 7. Serving a Saved Model (Local REST API)

```bash
mlflow models serve -m runs:/<RUN_ID>/model --port 9000
```

- **What it does:** Spins up a local REST server that loads a saved model and exposes it for predictions. `-m` points to the model location — `runs:/<RUN_ID>/model` means "the model artifact from this specific run ID."
- `--port 9000` — optional; if omitted, defaults to port 5000 (which would conflict with your tracking server if that's also running on 5000 — that's why 9000 is used here).

```bash
mlflow models serve -m /path/to/mlruns/<experiment_id>/<run_id>/artifacts/RandomForestClassifier --port 9000
```

- **What it does:** Same as above, but pointing directly at the artifact folder path on disk instead of using the `runs:/` shorthand. Useful when you're not connected to the tracking server, or want the model from a specific artifact folder.

### Sending predictions to the served model

Once the server above is running, send a POST request to:

```
http://127.0.0.1:9000/invocations
```

**Request body format** (`dataframe_split` — column names + row values):

```json
{
  "dataframe_split": {
    "columns": [
      "Gender",
      "Married",
      "Dependents",
      "Education",
      "Self_Employed",
      "LoanAmount",
      "Loan_Amount_Term",
      "Credit_History",
      "Property_Area",
      "TotalIncome"
    ],
    "data": [[1.0, 0.0, 0.0, 0.0, 0.0, 4.98745, 360.0, 1.0, 2.0, 8.698]]
  }
}
```

- **Column order matters** — it must match the order the model was trained on.

**Same thing via curl:**

```bash
curl --location 'http://127.0.0.1:9000/invocations' \
--header 'Content-Type: application/json' \
--data '{
    "dataframe_split": {
        "columns": [
            "Gender", "Married", "Dependents", "Education", "Self_Employed",
            "LoanAmount", "Loan_Amount_Term", "Credit_History",
            "Property_Area", "TotalIncome"
        ],
        "data": [
            [1.0, 0.0, 0.0, 0.0, 0.0, 4.98745, 360.0, 1.0, 2.0, 8.698]
        ]
    }
}'
```

---

## 8. Running the Tracking Server with a Database Backend (MySQL)

```bash
mlflow server \
  --host 0.0.0.0 \
  --port 5000 \
  --backend-store-uri mysql://root:admin123@localhost/db_mlflow \
  --default-artifact-root $PWD/mlruns
```

| Flag                      | What it does                                                                                                                                                                                                    | Optional?                                                                                |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `--host 0.0.0.0`          | Makes the server reachable from **any network interface**, not just your own machine (useful for a shared/team server). Default is `127.0.0.1` (local-only access).                                             | Optional — use `127.0.0.1` if you only need local access.                                |
| `--port 5000`             | Port the server listens on.                                                                                                                                                                                     | Optional — defaults to `5000` anyway.                                                    |
| `--backend-store-uri`     | Where **metadata** (params, metrics, tags, run info) is stored. Here it's a MySQL database (`db_mlflow`) instead of the default local SQLite/file store. Format: `mysql://<user>:<password>@<host>/<database>`. | Optional — defaults to local `./mlruns` files if omitted. Needed for shared/team setups. |
| `--default-artifact-root` | Where actual **artifact files** (models, plots, etc.) get stored — here, a local folder (`$PWD/mlruns`, i.e. current directory). Could also be S3, Azure Blob, etc.                                             | Optional — defaults to local `./mlruns` too.                                             |

> **Key distinction:** `--backend-store-uri` = metadata (numbers/text, goes in a database), `--default-artifact-root` = actual files (models/plots, goes on disk or cloud storage). They're independent — you can mix a database backend with local file artifacts, as shown above.

⚠️ Security note: storing your DB password in plain text in a command (`admin123` here) is fine for local learning, but avoid this in any real/shared environment — use environment variables or a secrets manager instead once you're past the bootcamp basics.

---

## Quick Reference — Command Cheat Sheet

| Command                                                             | Purpose                                                 |
| ------------------------------------------------------------------- | ------------------------------------------------------- |
| `mlflow.set_tracking_uri(uri)`                                      | Point client at tracking server or local folder         |
| `mlflow.set_experiment(name)`                                       | Select (or create) the experiment for runs              |
| `mlflow.start_run()`                                                | Begin a new run (use with `with`)                       |
| `mlflow.log_param(s)`                                               | Log hyperparameter(s)                                   |
| `mlflow.log_metric`                                                 | Log a numeric result                                    |
| `mlflow.log_artifact(path)`                                         | Upload one file                                         |
| `mlflow.log_artifacts(folder)`                                      | Upload a whole folder                                   |
| `mlflow.sklearn.log_model(model, name=)`                            | Save the trained model itself                           |
| `mlflow run .`                                                      | Run an MLflow Project locally                           |
| `mlflow models serve -m ... --port`                                 | Serve a saved model as a REST API                       |
| `mlflow server --backend-store-uri ... --default-artifact-root ...` | Start a full tracking server with DB + artifact storage |
