# PyTorch & TensorFlow — Summary Notes

Side-by-side reference for the two major deep learning frameworks — same concepts, different syntax and philosophy. TensorFlow examples use **Keras** (its high-level API, the standard way TensorFlow is used day-to-day).

---

## 1. Quick Framing — Core Difference

- **TensorFlow/Keras** — higher-level, more "batteries included." `.fit()` handles the entire training loop for you. Historically preferred in production/deployment pipelines.
- **PyTorch** — lower-level, more explicit/"Pythonic." You write the training loop yourself, giving more control and easier debugging. Historically preferred in research, and now dominant there — most new papers/models release PyTorch code first.

Both can do the same things; the difference is mainly how much is automated vs. written by hand.

---

## 2. Installation

```bash
pip install tensorflow
```
```bash
pip install torch torchvision torchaudio
```
- Both offer GPU-accelerated builds — TensorFlow auto-detects CUDA if installed correctly; PyTorch has separate install commands per CUDA version (check pytorch.org for the exact command matching your system).

---

## 3. Tensors — The Core Data Structure

Both frameworks operate on **tensors** (multi-dimensional arrays, like NumPy arrays but capable of running on GPU and tracking gradients).

**TensorFlow**
```python
import tensorflow as tf
x = tf.constant([[1, 2], [3, 4]])
x = tf.Variable([[1, 2], [3, 4]])   # trainable tensor
```
- `tf.constant` — fixed, non-trainable tensor. `tf.Variable` — a tensor whose values can be updated during training (used for weights).

**PyTorch**
```python
import torch
x = torch.tensor([[1, 2], [3, 4]])
x = torch.tensor([[1., 2.], [3., 4.]], requires_grad=True)
```
- `requires_grad=True` — tells PyTorch to track operations on this tensor so gradients can be computed automatically during backpropagation.

---

## 4. Building a Model

### TensorFlow (Keras)
```python
from tensorflow import keras

model = keras.Sequential([
    keras.layers.Dense(64, activation='relu', input_shape=(n_features,)),
    keras.layers.Dense(32, activation='relu'),
    keras.layers.Dense(1, activation='sigmoid')
])
```
- `Sequential` — stacks layers in order, one after another. Simplest way to define a model when it's a straight pipeline (no branches).

```python
inputs = keras.Input(shape=(n_features,))
x = keras.layers.Dense(64, activation='relu')(inputs)
outputs = keras.layers.Dense(1, activation='sigmoid')(x)
model = keras.Model(inputs, outputs)
```
- **Functional API** — same idea, but allows more complex architectures (multiple inputs/outputs, branches, skip connections) that `Sequential` can't express.

### PyTorch
```python
import torch.nn as nn

class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(n_features, 64)
        self.fc2 = nn.Linear(64, 32)
        self.fc3 = nn.Linear(32, 1)

    def forward(self, x):
        x = torch.relu(self.fc1(x))
        x = torch.relu(self.fc2(x))
        return torch.sigmoid(self.fc3(x))

model = Net()
```
- Models are defined as a **class** inheriting from `nn.Module`. `__init__` declares the layers; `forward()` defines how data flows through them — this explicit control is the main structural difference from Keras.

---

## 5. Loss Function & Optimizer

### TensorFlow (Keras)
```python
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)
```
- One line — `compile()` wires up the optimizer, loss, and tracked metrics all at once, ready for `.fit()`.

### PyTorch
```python
criterion = nn.BCELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
```
- Defined as **separate objects** you'll use manually inside your own training loop (§6). `model.parameters()` tells the optimizer which weights it's allowed to update.

---

## 6. Training

### TensorFlow (Keras) — training loop is automatic
```python
history = model.fit(
    X_train, y_train,
    validation_split=0.2,
    epochs=20,
    batch_size=32
)
```
- One call runs the entire loop: forward pass, loss calculation, backpropagation, weight updates, repeated for every batch and epoch.

### PyTorch — training loop is manual
```python
from torch.utils.data import DataLoader, TensorDataset

train_loader = DataLoader(TensorDataset(X_train, y_train), batch_size=32, shuffle=True)

for epoch in range(20):
    for X_batch, y_batch in train_loader:
        optimizer.zero_grad()          # clear old gradients
        outputs = model(X_batch)       # forward pass
        loss = criterion(outputs, y_batch)  # compute loss
        loss.backward()                # backpropagation
        optimizer.step()               # update weights
```
- Each step is written explicitly:
  - `optimizer.zero_grad()` — resets gradients from the previous batch (PyTorch accumulates gradients by default, so this is required every step).
  - `loss.backward()` — computes gradients via backpropagation.
  - `optimizer.step()` — applies the gradients to update the model's weights.
- This is more code, but it's exactly why PyTorch is preferred for research — every step is visible and easy to customize.

---

## 7. Evaluation & Prediction

### TensorFlow (Keras)
```python
loss, accuracy = model.evaluate(X_test, y_test)
predictions = model.predict(X_test)
```
- `.evaluate()` and `.predict()` handle batching internally, same as `.fit()`.

### PyTorch
```python
model.eval()                 # switch to evaluation mode (disables dropout, etc.)
with torch.no_grad():        # disable gradient tracking (saves memory/computation)
    predictions = model(X_test)
```
- `model.eval()` — important: some layers (Dropout, BatchNorm) behave differently during training vs. inference; forgetting this is a common PyTorch bug.
- `torch.no_grad()` — since you're not training, there's no need to track gradients — skipping this wastes memory and compute.

---

## 8. Using a GPU

### TensorFlow
```python
tf.config.list_physical_devices('GPU')
```
- TensorFlow automatically uses an available GPU for `.fit()`/`.predict()` — no manual device placement usually needed.

### PyTorch
```python
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model.to(device)
X_batch = X_batch.to(device)
```
- PyTorch requires **explicitly moving** the model and every tensor to the GPU (`.to(device)`) — nothing runs on GPU automatically. More manual, but makes it clear exactly what's running where.

---

## 9. Save & Load the Model

### TensorFlow (Keras)
```python
model.save("model.keras")
loaded_model = keras.models.load_model("model.keras")
```
- Saves architecture + weights + optimizer state together in one file — ready to reload and use immediately.

### PyTorch
```python
torch.save(model.state_dict(), "model.pth")

model = Net()                                  # must recreate the architecture first
model.load_state_dict(torch.load("model.pth"))
model.eval()
```
- PyTorch typically saves only the **weights** (`state_dict`), not the architecture — you need the original model class defined to reload into. (You *can* save the whole model object with `torch.save(model, "model.pth")`, but saving just the weights is the recommended, more portable practice.)

---

## Side-by-Side Comparison

| Concept | TensorFlow/Keras | PyTorch |
|---|---|---|
| Model definition | `Sequential` / Functional API | `class` inheriting `nn.Module` |
| Training loop | Automatic (`.fit()`) | Manual (`for` loop, explicit backward/step) |
| Debugging | Harder to step into (more abstraction) | Easier — standard Python debugging works throughout |
| GPU usage | Automatic | Manual (`.to(device)` everywhere) |
| Saving | Whole model in one file by default | Usually just weights (`state_dict`), needs architecture defined separately |
| Deployment tooling | TensorFlow Serving, TF Lite, TF.js — mature, production-focused | TorchServe, ONNX export — improving, historically less production-focused |
| Common use case today | Production pipelines, mobile/edge deployment | Research, most new pretrained models (including Hugging Face defaults) |
| Learning curve | Gentler to start | Steeper at first, but more transparent once learned |

---

## Quick Reference — Equivalent Commands

| Task | TensorFlow/Keras | PyTorch |
|---|---|---|
| Create a tensor | `tf.constant(...)` | `torch.tensor(...)` |
| Dense/Linear layer | `keras.layers.Dense(64)` | `nn.Linear(in_features, 64)` |
| ReLU activation | `activation='relu'` | `torch.relu(x)` or `nn.ReLU()` |
| Define model | `keras.Sequential([...])` | `class Net(nn.Module): ...` |
| Set loss + optimizer | `model.compile(...)` | `criterion = ...`, `optimizer = ...` |
| Train | `model.fit(X, y, epochs=...)` | manual `for` loop over epochs/batches |
| Evaluate | `model.evaluate(X, y)` | `model.eval()` + manual loop |
| Predict | `model.predict(X)` | `model(X)` inside `torch.no_grad()` |
| Save | `model.save(path)` | `torch.save(model.state_dict(), path)` |
| Load | `keras.models.load_model(path)` | `model.load_state_dict(torch.load(path))` |
