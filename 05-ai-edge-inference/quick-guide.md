# 📦 Edge AI with TensorFlow Lite — Quick Guide

A simple, practical guide to **running AI on edge (IoT) devices** using TensorFlow Lite, covering:

*   What TensorFlow Lite is
*   What inference means
*   How models are trained, converted, and optimized

***

# 🚀 Big Picture

    Data → Train Model → Convert → Optimize → Deploy → Inference

*   Train on powerful machines
*   Shrink the model
*   Run it directly on devices
*   Get real-time predictions

***

# 🧠 Core Concepts

## TensorFlow Lite (TFLite)

A **lightweight runtime** that allows AI models to run on small devices:

*   Phones 📱
*   Wearables ⌚
*   Cameras 📷
*   IoT systems 🌡️

👉 Think of it as a **mini AI engine inside devices**

***

## Inference

**Inference = using a trained model to make predictions**

Example:

    Input → Model → Output
    Image → Model → "Person detected"

*   No learning happens
*   Only prediction happens
*   Happens **in real-time on the device**

***

# ⚡ Why Edge AI?

*   ⚡ Fast (no cloud delay)
*   🔒 Private (data stays local)
*   🌐 Works offline
*   🔋 Low power usage

***

# 🧑‍💻 Step 1: Train the Model (TensorFlow)

Models are trained using **TensorFlow/Keras**, not TFLite.

```python
import tensorflow as tf
from tensorflow import keras
import numpy as np

# Dummy data
X = np.array([[0], [1], [2], [3]], dtype=float)
y = np.array([[0], [2], [4], [6]], dtype=float)

# Model
model = keras.Sequential([
    keras.layers.Dense(1, input_shape=[1])
])

model.compile(optimizer='adam', loss='mse')
model.fit(X, y, epochs=100)
```

***

# 💾 Step 2: Save the Model

```python
model.save("model.h5")
```

***

# 🔄 Step 3: Convert to TensorFlow Lite

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_keras_model(model)
tflite_model = converter.convert()

with open("model.tflite", "wb") as f:
    f.write(tflite_model)
```

✅ Now you have a `.tflite` model

***

# ⚙️ Step 4: Optimize / Compress the Model

## ✅ Dynamic Quantization (Easy)

```python
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()
```

*   Smaller size 📦
*   Faster ⚡
*   Minimal accuracy loss

***

## ✅ Full Integer Quantization (Best for Edge)

```python
def representative_data():
    for _ in range(100):
        yield [X.astype(np.float32)]

converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_data

converter.target_spec.supported_ops = [
    tf.lite.OpsSet.TFLITE_BUILTINS_INT8
]
```

*   Very small model
*   Lower power use
*   Ideal for microcontrollers

***

## ✅ Optional: Pruning

Removes unnecessary weights to shrink model further.

***

# 📱 Step 5: Run Inference on Device

```python
interpreter = tf.lite.Interpreter(model_path="model.tflite")
interpreter.allocate_tensors()

interpreter.set_tensor(input_index, input_data)
interpreter.invoke()
output = interpreter.get_tensor(output_index)
```

***

# 🧩 What Happens During Inference

    Input data → Model → Prediction

Examples:

*   Camera → “Person detected”
*   ECG → “Abnormal rhythm”
*   Sensor → “Anomaly detected”

***

# 🏥 Real-World Example (Healthcare)

**Wearable ECG Monitor**

    Signal → TFLite Model → Output: Normal / Arrhythmia

*   Runs locally
*   Real-time alerts
*   Energy efficient

***

# ⚙️ How Models Are Made Smaller

### Quantization

*   Convert float → int (less memory)

### Pruning

*   Remove weak connections

### Optimization

*   Speed up execution on device hardware

***

# ✅ Quick Summary

| Concept         | Meaning                       |
| --------------- | ----------------------------- |
| TensorFlow Lite | AI runtime for edge devices   |
| Training        | Model learns from data        |
| Conversion      | Convert to `.tflite`          |
| Optimization    | Reduce size and improve speed |
| Inference       | Generate predictions          |

***

# 🧠 Final Takeaway

> Train a model → shrink it → deploy it → run inference directly on devices

This enables:

*   Real-time AI
*   Offline intelligence
*   Efficient, private computing

***

# ⚡ One-Line Summary

**TensorFlow Lite lets small devices run trained AI models locally, where inference turns real-world input into instant predictions.**
