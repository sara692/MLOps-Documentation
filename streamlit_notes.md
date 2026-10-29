# Streamlit — Summary Notes

Reference for building interactive web apps/dashboards in pure Python — commonly used to wrap ML models, data explorations, and demos with a UI, without writing HTML/CSS/JS.

---

## 1. Setup & Running an App

```bash
pip install streamlit
```

```bash
streamlit run app.py
```
- **What it does:** Starts a local web server (default `http://localhost:8501`) and runs `app.py` top to bottom, rendering whatever Streamlit commands it contains. The app **re-runs the entire script from top to bottom** on every interaction (a button click, a slider move) — this is the core mental model to understand before anything else.

```python
import streamlit as st
```
- Standard import — `st` is used for virtually every Streamlit command.

---

## 2. Displaying Text & Data

```python
st.title("My App")
st.header("Section Header")
st.subheader("Subsection")
st.write("Any text, dataframe, chart, or object — Streamlit infers how to display it.")
```
- `title`/`header`/`subheader` — sized headings, largest to smallest.
- `st.write()` — the "magic" catch-all function; auto-detects the type passed in (string, DataFrame, chart, dict) and renders it appropriately. Good default when you're not sure which specific command to use.

```python
st.markdown("**Bold text**, *italics*, and [links](https://example.com)")
```
- Renders Markdown-formatted text directly.

```python
st.dataframe(df)
```
- Displays a Pandas DataFrame as an **interactive** table (sortable, scrollable, resizable columns).

```python
st.table(df)
```
- Displays a DataFrame as a **static** table (no interactivity) — better for small, fixed reference tables.

```python
st.json(my_dict)
```
- Pretty-prints a dictionary/JSON object with collapsible nesting.

---

## 3. Input Widgets

Every widget below returns the current value directly — assign it to a variable and use it immediately in the same script run.

```python
name = st.text_input("Enter your name", value="")
```
- Single-line text input. `value` sets the default/starting text — optional.

```python
age = st.number_input("Age", min_value=0, max_value=120, value=25)
```
- Numeric input with optional bounds. `min_value`/`max_value` are optional but recommended to prevent invalid entries.

```python
score = st.slider("Score", min_value=0, max_value=100, value=50, step=1)
```
- Drag-to-select numeric input. `step` controls the increment. Can also take a `(min, max)` tuple as `value` for a **range** slider.

```python
option = st.selectbox("Choose one", ["Option A", "Option B", "Option C"])
```
- Dropdown — single selection from a list.

```python
options = st.multiselect("Choose several", ["A", "B", "C"])
```
- Same as `selectbox`, but allows multiple selections (returns a list).

```python
agree = st.checkbox("I agree")
```
- Boolean toggle. Returns `True`/`False`.

```python
choice = st.radio("Pick one", ["Yes", "No", "Maybe"])
```
- Radio buttons — single selection, all options visible at once (unlike a dropdown).

```python
clicked = st.button("Run")
if clicked:
    st.write("Button was clicked!")
```
- Returns `True` only on the exact script run where it was clicked (resets to `False` on the next re-run) — this is why button logic is usually wrapped in an `if`.

```python
uploaded_file = st.file_uploader("Upload a CSV", type=["csv"])
if uploaded_file is not None:
    df = pd.read_csv(uploaded_file)
```
- File upload widget. `type` restricts accepted file extensions (optional — omit to accept any file).

```python
date = st.date_input("Pick a date")
```
- Calendar date picker.

---

## 4. Layout & Organization

```python
col1, col2 = st.columns(2)
with col1:
    st.write("Left column")
with col2:
    st.write("Right column")
```
- **Columns** — arranges elements side-by-side instead of stacked vertically. Pass a list of relative widths (e.g. `st.columns([2, 1])`) for uneven columns.

```python
with st.sidebar:
    st.header("Settings")
    option = st.selectbox("Model", ["A", "B"])
```
- **Sidebar** — a collapsible panel on the left, commonly used for filters/settings so they don't clutter the main content area.

```python
tab1, tab2 = st.tabs(["Overview", "Details"])
with tab1:
    st.write("Overview content")
with tab2:
    st.write("Details content")
```
- **Tabs** — splits content into switchable sections within the same page.

```python
with st.expander("Show more"):
    st.write("Hidden content revealed on click")
```
- **Expander** — collapsible section, useful for optional details or long content that shouldn't dominate the page by default.

```python
with st.container():
    st.write("Grouped content")
```
- **Container** — groups elements together logically (useful for controlling render order when combined with placeholders), without any visual styling on its own.

---

## 5. Charts & Visualization

```python
st.line_chart(df)
st.bar_chart(df)
st.area_chart(df)
```
- Quick native charts — take a DataFrame directly, minimal setup, good for fast exploration.

```python
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot(x, y)
st.pyplot(fig)
```
- Renders any Matplotlib figure — use when you need more control than the native chart commands offer.

```python
import plotly.express as px
fig = px.scatter(df, x="col1", y="col2")
st.plotly_chart(fig)
```
- Renders an interactive Plotly chart (zoom, hover tooltips, pan).

```python
st.map(df)  # df needs 'lat' and 'lon' columns
```
- Quick geographic point map.

```python
st.image("photo.png", caption="A photo", width=300)
```
- Displays an image. `width` is optional — omitted defaults to the image's natural size (capped to container width).

---

## 6. Session State (Persisting Values Across Re-Runs)

Since the entire script re-runs on every interaction, regular Python variables reset every time. `st.session_state` is how you persist values across those re-runs.

```python
if "counter" not in st.session_state:
    st.session_state.counter = 0

if st.button("Increment"):
    st.session_state.counter += 1

st.write(f"Count: {st.session_state.counter}")
```
- `st.session_state` behaves like a dictionary that survives across re-runs (but resets if the browser tab is closed/refreshed, or a new session starts). The `if "key" not in st.session_state` pattern initializes a value only once, on the very first run.

```python
st.text_input("Name", key="username")
st.write(st.session_state.username)
```
- Passing a `key` to any widget automatically stores its value in `st.session_state` under that key — an alternative to manually assigning the widget's return value.

---

## 7. Caching (Avoiding Repeated Expensive Work)

Since the whole script re-runs constantly, expensive operations (loading a large model, querying a database, reading a big file) would otherwise repeat on every single interaction. Caching prevents that.

```python
@st.cache_data
def load_data():
    return pd.read_csv("large_file.csv")

df = load_data()
```
- **`@st.cache_data`** — caches the **return value** of a function based on its inputs; re-running with the same arguments returns the cached result instantly instead of re-executing. Use for data loading/processing (DataFrames, lists, dicts — anything serializable).

```python
@st.cache_resource
def load_model():
    return joblib.load("model.pkl")

model = load_model()
```
- **`@st.cache_resource`** — same idea, but for objects that shouldn't be copied/serialized (ML models, DB connections, API clients). Ensures a heavy model is loaded into memory **once**, not on every interaction.

> **Which to use?** `cache_data` for data (safe to copy). `cache_resource` for live objects/connections (must stay as a single shared instance).

---

## 8. Status & Feedback Elements

```python
st.success("Done!")
st.error("Something went wrong")
st.warning("Careful with this")
st.info("Just so you know")
```
- Colored, iconified message boxes for user feedback.

```python
with st.spinner("Running inference..."):
    result = model.predict(X)
st.success("Prediction complete")
```
- Shows a loading spinner for the duration of a block — useful around any slow operation (model inference, API calls) so the user knows something's happening.

```python
progress = st.progress(0)
for i in range(100):
    progress.progress(i + 1)
```
- A progress bar you update manually — useful for long-running loops (batch processing, training).

---

## 9. Multipage Apps

```
my_app/
├── app.py                 # main/home page
└── pages/
    ├── 1_Data_Explorer.py
    └── 2_Model_Predictions.py
```
- Any `.py` file inside a `pages/` folder automatically becomes a separate page, with navigation added to the sidebar automatically — no extra routing code needed. The number prefix (`1_`, `2_`) controls display order; underscores in filenames are shown as spaces in the nav.

```python
st.page_link("pages/1_Data_Explorer.py", label="Go to Data Explorer")
```
- Creates a clickable link to another page from anywhere in the app.

---

## 10. Deploying an ML Model — Typical Pattern

```python
import streamlit as st
import joblib

@st.cache_resource
def load_model():
    return joblib.load("model.pkl")

model = load_model()

st.title("Loan Approval Predictor")

income = st.number_input("Income", min_value=0)
credit_history = st.selectbox("Credit History", [0, 1])

if st.button("Predict"):
    prediction = model.predict([[income, credit_history]])
    st.success(f"Prediction: {'Approved' if prediction[0] == 1 else 'Rejected'}")
```
- The standard shape: load the model once (cached), collect inputs via widgets, run prediction on button click, display the result.

---

## 11. Deployment

```bash
streamlit run app.py --server.port 8080
```
- Runs the app on a specific port (optional — defaults to `8501`).

**Streamlit Community Cloud** — the free, official hosting option: push your app to a public GitHub repo (with a `requirements.txt`), then connect it at share.streamlit.io. No server setup required.

```
requirements.txt
```
- List your dependencies (`streamlit`, `pandas`, `scikit-learn`, etc.) — required for both Community Cloud and Docker deployment so the environment can be rebuilt.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.address=0.0.0.0"]
```
- Standard Dockerfile pattern for containerizing a Streamlit app (see the Docker notes file for a line-by-line breakdown of each instruction). `--server.address=0.0.0.0` is important — without it, the app only listens on localhost inside the container and won't be reachable from outside.

---

## Quick Reference — Cheat Sheet

| Task | Command |
|---|---|
| Run the app | `streamlit run app.py` |
| Display text | `st.write(...)` / `st.markdown(...)` |
| Show a DataFrame | `st.dataframe(df)` |
| Text input | `st.text_input(label)` |
| Number input | `st.number_input(label)` |
| Slider | `st.slider(label, min, max)` |
| Dropdown | `st.selectbox(label, options)` |
| Multi-select | `st.multiselect(label, options)` |
| Checkbox | `st.checkbox(label)` |
| Button | `st.button(label)` |
| File upload | `st.file_uploader(label)` |
| Columns layout | `st.columns(n)` |
| Sidebar | `with st.sidebar:` |
| Tabs | `st.tabs([...])` |
| Expander | `with st.expander(label):` |
| Native line/bar chart | `st.line_chart(df)` / `st.bar_chart(df)` |
| Matplotlib figure | `st.pyplot(fig)` |
| Persist a value | `st.session_state["key"]` |
| Cache data | `@st.cache_data` |
| Cache a model/connection | `@st.cache_resource` |
| Loading spinner | `with st.spinner(text):` |
| Progress bar | `st.progress(value)` |
| Success/error message | `st.success(...)` / `st.error(...)` |
| Multipage app | files inside a `pages/` folder |
