# Deep Learning — Summary Notes

End-to-end reference: data prep → building a network → compiling → training → evaluation → saving. Code examples use **Keras/TensorFlow** (the most common starting point). Architectures are split by task type since CNNs, RNNs, and plain ANNs are built for different kinds of data.

---

## 1. What Is Deep Learning (Quick Framing)

Deep learning is a subset of machine learning using **neural networks** — layers of connected nodes ("neurons") that learn patterns by adjusting internal weights through repeated exposure to data. "Deep" just means multiple hidden layers stacked between input and output. Unlike classical ML (Ch. supervised/unsupervised notes), deep learning can automatically learn feature representations from raw data (images, text, audio) rather than needing hand-crafted features.

---

## 2. Load & Preprocess Data

```python
import tensorflow as tf
from tensorflow import keras
```
- Standard imports for building/training networks in Keras (TensorFlow's high-level API).

```python
from sklearn.preprocessing import StandardScaler
X_scaled = StandardScaler().fit_transform(X)
```
- **Scaling is essential** for neural networks — unscaled inputs slow down or destabilize training since gradients get skewed by feature magnitude.

```python
y_categorical = keras.utils.to_categorical(y, num_classes=3)
```
- Converts integer class labels into **one-hot encoded** vectors (e.g. class `2` of 3 → `[0, 0, 1]`) — required for multi-class classification with certain loss functions (see §6).

```python
train_ds = tf.keras.utils.image_dataset_from_directory(
    "data/", image_size=(224, 224), batch_size=32
)
```
- Loads images directly from a folder structure (one subfolder per class) into a ready-to-train dataset — common entry point for CNN/image tasks.

```python
from tensorflow.keras.preprocessing.sequence import pad_sequences
X_padded = pad_sequences(X_sequences, maxlen=100)
```
- Pads/truncates variable-length sequences (e.g. tokenized text) to a fixed length — required before feeding text into RNN/LSTM models.

---

## 3. Building Blocks — Layers & Activation Functions

### Common layer types
```python
keras.layers.Dense(units=64, activation='relu')
```
- **Dense (fully connected) layer** — every neuron connects to every neuron in the previous layer. The core building block of a standard ANN.

```python
keras.layers.Conv2D(filters=32, kernel_size=(3,3), activation='relu')
```
- **Convolutional layer** — slides small filters over the input (e.g. an image) to detect local patterns like edges/textures. Core building block of a CNN.

```python
keras.layers.MaxPooling2D(pool_size=(2,2))
```
- **Pooling layer** — downsamples feature maps (keeps the strongest signal, discards the rest), reducing size and computation while retaining important features.

```python
keras.layers.LSTM(units=64)
```
- **LSTM (Long Short-Term Memory) layer** — a type of recurrent layer that processes sequences step-by-step while retaining relevant information over long sequences. Core building block for sequence/text/time-series data.

```python
keras.layers.Dropout(rate=0.3)
```
- **Dropout layer** — randomly "switches off" a fraction of neurons during each training step, forcing the network not to over-rely on any single neuron. A key regularization technique (see §7).

```python
keras.layers.BatchNormalization()
```
- **Batch Normalization layer** — rescales/recenters activations between layers during training, which speeds up training and adds a mild regularizing effect.

### Common activation functions (used inside layers above)

- **ReLU** (`'relu'`) — outputs the input directly if positive, else 0. Fast, avoids certain training issues, and is the default choice for hidden layers.
- **Sigmoid** (`'sigmoid'`) — squashes output to a range between 0 and 1. Used in the output layer for **binary** classification.
- **Softmax** (`'softmax'`) — converts outputs into a probability distribution across classes (sums to 1). Used in the output layer for **multi-class** classification.
- **Tanh** (`'tanh'`) — squashes output between -1 and 1. Sometimes used in RNN/LSTM layers instead of ReLU.
- **Linear** (`None` / no activation) — outputs the raw value unchanged. Used in the output layer for **regression** tasks.

---

## 4. Model Architectures — By Task Type

### ANN (Artificial Neural Network) — for tabular data
```python
model = keras.Sequential([
    keras.layers.Dense(64, activation='relu', input_shape=(n_features,)),
    keras.layers.Dense(32, activation='relu'),
    keras.layers.Dense(1, activation='sigmoid')   # regression: no activation
])
```
- A stack of Dense layers. Suited to structured/tabular data — the deep learning equivalent of classical ML models like Logistic Regression or Random Forest, but capable of learning more complex non-linear patterns.

### CNN (Convolutional Neural Network) — for images
```python
model = keras.Sequential([
    keras.layers.Conv2D(32, (3,3), activation='relu', input_shape=(224,224,3)),
    keras.layers.MaxPooling2D((2,2)),
    keras.layers.Conv2D(64, (3,3), activation='relu'),
    keras.layers.MaxPooling2D((2,2)),
    keras.layers.Flatten(),
    keras.layers.Dense(64, activation='relu'),
    keras.layers.Dense(num_classes, activation='softmax')
])
```
- Stacks convolution + pooling layers to progressively extract spatial features (edges → shapes → objects), then flattens into Dense layers for final classification. The standard architecture for image tasks.

### RNN / LSTM / GRU — for sequences (text, time-series)
```python
model = keras.Sequential([
    keras.layers.Embedding(input_dim=vocab_size, output_dim=64),
    keras.layers.LSTM(64),
    keras.layers.Dense(1, activation='sigmoid')
])
```
- Processes input step-by-step, carrying forward a "memory" of previous steps — suited to data where order matters (sentences, stock prices, sensor readings). LSTM/GRU are improved variants of a plain RNN that handle longer sequences better by fighting the "vanishing gradient" problem.

### Transfer Learning — reuse a pretrained model
```python
base_model = keras.applications.MobileNetV2(
    input_shape=(224,224,3), include_top=False, weights='imagenet'
)
base_model.trainable = False
model = keras.Sequential([
    base_model,
    keras.layers.GlobalAveragePooling2D(),
    keras.layers.Dense(num_classes, activation='softmax')
])
```
- Starts from a model already trained on a large dataset (e.g. ImageNet) and reuses its learned features, only training a new small head on top. Dramatically reduces training time/data needed — the standard approach when you don't have a huge dataset of your own.

---

## 5. Compiling the Model

```python
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)
```
- `optimizer` — the algorithm that updates weights during training (see §8 for details/options).
- `loss` — the function being minimized during training (see §8 for options by task type).
- `metrics` — additional scores tracked during training/evaluation for human monitoring (doesn't affect training itself, unlike `loss`).

---

## 6. Training

```python
history = model.fit(
    X_train, y_train,
    validation_split=0.2,
    epochs=20,
    batch_size=32
)
```
- `validation_split` — holds out a fraction of training data to monitor performance on unseen data each epoch (helps spot overfitting live during training).
- `epochs` — number of full passes through the entire training dataset.
- `batch_size` — number of samples processed before the model's weights are updated once.

### Callbacks (optional training controls)
```python
early_stop = keras.callbacks.EarlyStopping(monitor='val_loss', patience=5)
checkpoint = keras.callbacks.ModelCheckpoint("best_model.h5", save_best_only=True)

model.fit(X_train, y_train, epochs=50, callbacks=[early_stop, checkpoint])
```
- **EarlyStopping** — automatically stops training once a monitored metric (e.g. validation loss) stops improving for `patience` epochs, preventing wasted training / overfitting.
- **ModelCheckpoint** — saves the model automatically whenever it achieves the best score so far, so you always keep the best version even if later epochs get worse.

---

## 7. Regularization Techniques (Preventing Overfitting)

- **Dropout** — randomly disables neurons during training (see §3). Common rate: `0.2`–`0.5`.
- **L2 Regularization (weight decay)** — penalizes large weights, added directly in a layer: `keras.layers.Dense(64, kernel_regularizer=keras.regularizers.l2(0.01))`.
- **Batch Normalization** — normalizes activations between layers (see §3); has a side-benefit of mild regularization.
- **Early Stopping** — halts training once validation performance stops improving (see §6).
- **Data Augmentation** (images) — artificially expands training data with random transformations (flips, rotations, zooms) so the model generalizes better instead of memorizing exact images.

---

## 8. Loss Functions & Optimizers

### Loss functions (choose based on the task)

| Task | Loss function |
|---|---|
| Binary classification | `'binary_crossentropy'` — penalizes confident wrong predictions between two classes |
| Multi-class classification (one-hot labels) | `'categorical_crossentropy'` — same idea, extended across multiple classes |
| Multi-class classification (integer labels) | `'sparse_categorical_crossentropy'` — same as above, but labels are plain integers instead of one-hot vectors |
| Regression | `'mean_squared_error'` (MSE) or `'mean_absolute_error'` (MAE) — same definitions as in classical regression |

### Optimizers (algorithms that perform gradient descent)

```python
keras.optimizers.SGD(learning_rate=0.01, momentum=0.0)
```
- **SGD (Stochastic Gradient Descent)** — the base optimization algorithm; updates weights using the gradient of the loss, computed on small batches rather than the whole dataset at once.
  `learning_rate` — **Default:** `0.01` · **Range:** small positive float (commonly `0.0001`–`0.1`)
  `momentum` — **Default:** `0.0` · **Range:** `0.0`–`1.0` (adds "inertia" to updates, helping escape shallow local minima and speeding convergence)

```python
keras.optimizers.Adam(learning_rate=0.001, beta_1=0.9, beta_2=0.999)
```
- **Adam** — the most commonly used default optimizer; combines momentum with an adaptive learning rate per parameter, generally converges faster and needs less manual tuning than plain SGD.
  `learning_rate` — **Default:** `0.001` · **Range:** small positive float (commonly `0.0001`–`0.01`)
  `beta_1` / `beta_2` — control how much past gradients influence the current update (momentum-like terms). **Default:** `0.9` / `0.999` · **Range:** `0.0`–`1.0` (rarely changed from default)

```python
keras.optimizers.RMSprop(learning_rate=0.001, rho=0.9)
```
- **RMSprop** — adapts the learning rate for each parameter based on recent gradient magnitudes; historically popular for RNNs, largely superseded by Adam for most use cases today.
  `learning_rate` — **Default:** `0.001` · **Range:** small positive float
  `rho` — decay rate for the moving average of squared gradients. **Default:** `0.9` · **Range:** `0.0`–`1.0`

---

## 9. Evaluation

```python
loss, accuracy = model.evaluate(X_test, y_test)
```
- Runs the model on held-out test data, returning the loss and any tracked metrics (e.g. accuracy) — same evaluation concept as classical ML, just called via `.evaluate()` instead of computing metrics manually.

```python
y_pred = model.predict(X_test)
```
- Generates raw predictions (probabilities for classification, values for regression) — same downstream metric functions from classical ML (`accuracy_score`, `f1_score`, `mean_squared_error`, etc.) can be applied after this.

```python
import matplotlib.pyplot as plt
plt.plot(history.history['loss'], label='train loss')
plt.plot(history.history['val_loss'], label='val loss')
```
- Plotting the loss curves from `history` (returned by `.fit()`) — the standard way to visually check for overfitting (train loss keeps dropping while validation loss rises = overfitting).

---

## 10. Save & Load the Model

```python
model.save("model.h5")               # or model.save("model.keras") — newer format
loaded_model = keras.models.load_model("model.h5")
```
- Saves the entire model — architecture, learned weights, and optimizer state — in one file, ready to reload and use later without retraining.

```python
import mlflow.keras
mlflow.keras.log_model(model, name="model_name")
```
- Alternative: log via MLflow (see the MLflow notes file) for versioning and tracking alongside other experiments.

---

## Summary — Key Hyperparameters (Quick Reference)

| Hyperparameter | What it controls | Default (Keras) | Typical range |
|---|---|---|---|
| `learning_rate` | Step size for each weight update during training | optimizer-dependent (`0.001` for Adam) | `0.0001`–`0.1` |
| `epochs` | Number of full passes through the training data | must be set manually, no fixed default | `10`–`200`+ (paired with early stopping) |
| `batch_size` | Number of samples per weight update | `32` | `16`–`512` (powers of 2 common) |
| `units` (per layer) | Number of neurons in a Dense/LSTM layer | must be set manually | `16`–`512`, shrinking toward the output layer |
| `dropout rate` | Fraction of neurons randomly disabled per step | must be set manually | `0.2`–`0.5` |
| `kernel_size` (CNN) | Size of the sliding filter window | must be set manually | `(3,3)` or `(5,5)` common |
| `filters` (CNN) | Number of feature detectors per conv layer | must be set manually | `16`–`512`, growing with depth |
| `patience` (EarlyStopping) | Epochs to wait for improvement before stopping | must be set manually | `3`–`10` |

---

## Backpropagation & Gradient Descent — In Brief

- **Forward pass** — input data flows through the network layer by layer, producing a prediction.
- **Loss calculation** — the prediction is compared to the true label using the chosen loss function (§8), producing a single error value.
- **Backward pass (backpropagation)** — the error is propagated backward through the network using the chain rule of calculus, calculating how much each individual weight contributed to the error (the gradient).
- **Weight update (gradient descent)** — each weight is nudged slightly in the direction that reduces the error, scaled by the `learning_rate`. This whole forward → loss → backward → update cycle repeats for every batch, across every epoch.
- **Why "stochastic"/"mini-batch"?** — computing gradients on the full dataset every step would be extremely slow; using small batches (`batch_size`) makes each update fast, at the cost of noisier (less precise) gradient estimates — which in practice often helps the model avoid getting stuck in poor solutions.
