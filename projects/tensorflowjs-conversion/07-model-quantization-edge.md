# Project 7: Convert and Quantize Models for Edge Devices

## Overview
Master model optimization techniques for deploying TensorFlow.js models on resource-constrained edge devices like smartphones, tablets, and IoT devices. Learn to apply quantization (float16, uint8), pruning, and other optimization strategies to reduce model size while maintaining acceptable accuracy, enabling efficient inference on low-power devices.

## Difficulty Level
Advanced

## Learning Objectives
- Understand different quantization techniques (float16, uint8, int8)
- Apply post-training quantization to TensorFlow models
- Convert quantized models to TensorFlow.js format
- Benchmark model performance (size, speed, accuracy)
- Implement adaptive model selection based on device capabilities
- Optimize for WebGL and WASM backends
- Test across different device tiers (high-end, mid-range, low-end)

## Technical Stack
- **Backend**: Python 3.8+, TensorFlow 2.x
- **Conversion**: tensorflowjs_converter with quantization flags
- **Frontend**: HTML5, JavaScript ES6+, TensorFlow.js
- **Optimization**: Model pruning, quantization-aware training
- **Benchmarking**: TensorFlow.js profiling tools
- **Testing**: Multiple device emulation, real device testing

## Project Requirements

### 1. Model Preparation
- Train baseline model for comparison
- Implement post-training quantization
- Apply quantization-aware training (optional)
- Create multiple quantized versions (float16, uint8)

### 2. Conversion Pipeline
- Convert original model to TFJS
- Convert quantized models to TFJS
- Compare file sizes and model artifacts
- Validate numerical accuracy

### 3. Performance Benchmarking
- Measure model load times
- Benchmark inference speed
- Track memory usage
- Compare accuracy across quantization levels
- Test on different backends (WebGL, WASM, CPU)

### 4. Adaptive Loading
- Detect device capabilities
- Select appropriate model version
- Implement progressive loading
- Provide fallback options

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
mkdir model-quantization-edge
cd model-quantization-edge

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install tensorflow tensorflow-model-optimization tensorflowjs numpy matplotlib
```

### Step 2: Train Baseline Model
```python
# train_baseline.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import json

# Load MNIST dataset
(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()

# Normalize
x_train = x_train.astype('float32') / 255.0
x_test = x_test.astype('float32') / 255.0

# Reshape
x_train = x_train.reshape(-1, 28, 28, 1)
x_test = x_test.reshape(-1, 28, 28, 1)

# Build model
def create_model():
    model = keras.Sequential([
        keras.layers.Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)),
        keras.layers.MaxPooling2D((2, 2)),
        keras.layers.Conv2D(64, (3, 3), activation='relu'),
        keras.layers.MaxPooling2D((2, 2)),
        keras.layers.Conv2D(64, (3, 3), activation='relu'),
        keras.layers.Flatten(),
        keras.layers.Dense(64, activation='relu'),
        keras.layers.Dropout(0.5),
        keras.layers.Dense(10, activation='softmax')
    ])
    return model

# Create and train model
model = create_model()

model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

print("Training baseline model...")
history = model.fit(
    x_train, y_train,
    batch_size=128,
    epochs=10,
    validation_split=0.1,
    verbose=1
)

# Evaluate
test_loss, test_acc = model.evaluate(x_test, y_test, verbose=0)
print(f'\nBaseline Test Accuracy: {test_acc:.4f}')

# Save model
model.save('mnist_baseline.h5')

# Save as SavedModel for quantization
tf.saved_model.save(model, 'mnist_baseline_savedmodel')

print("Baseline model saved!")

# Get model size
import os
def get_model_size(path):
    total = 0
    for dirpath, dirnames, filenames in os.walk(path):
        for f in filenames:
            fp = os.path.join(dirpath, f)
            total += os.path.getsize(fp)
    return total / (1024 * 1024)  # MB

baseline_size = get_model_size('mnist_baseline_savedmodel')
print(f"Baseline model size: {baseline_size:.2f} MB")

# Save metrics
metrics = {
    'baseline': {
        'accuracy': float(test_acc),
        'loss': float(test_loss),
        'size_mb': baseline_size
    }
}

with open('model_metrics.json', 'w') as f:
    json.dump(metrics, f, indent=2)
```

### Step 3: Apply Post-Training Quantization
```python
# quantize_models.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import json
import os

# Load test data for representative dataset
(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()
x_test = x_test.astype('float32') / 255.0
x_test = x_test.reshape(-1, 28, 28, 1)

# Load saved model
model = keras.models.load_model('mnist_baseline.h5')

def get_model_size(path):
    total = 0
    for dirpath, dirnames, filenames in os.walk(path):
        for f in filenames:
            fp = os.path.join(dirpath, f)
            total += os.path.getsize(fp)
    return total / (1024 * 1024)

# === Float16 Quantization ===
print("\n=== Float16 Quantization ===")

converter_f16 = tf.lite.TFLiteConverter.from_keras_model(model)
converter_f16.optimizations = [tf.lite.Optimize.DEFAULT]
converter_f16.target_spec.supported_types = [tf.float16]

tflite_f16_model = converter_f16.convert()

with open('mnist_float16.tflite', 'wb') as f:
    f.write(tflite_f16_model)

f16_size = os.path.getsize('mnist_float16.tflite') / (1024 * 1024)
print(f"Float16 model size: {f16_size:.2f} MB")
print(f"Size reduction: {((1 - f16_size/get_model_size('mnist_baseline_savedmodel')) * 100):.1f}%")

# === INT8 Quantization with representative dataset ===
print("\n=== INT8 Quantization ===")

def representative_dataset():
    for i in range(100):
        yield [x_test[i:i+1].astype(np.float32)]

converter_int8 = tf.lite.TFLiteConverter.from_keras_model(model)
converter_int8.optimizations = [tf.lite.Optimize.DEFAULT]
converter_int8.representative_dataset = representative_dataset
converter_int8.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter_int8.inference_input_type = tf.uint8
converter_int8.inference_output_type = tf.uint8

tflite_int8_model = converter_int8.convert()

with open('mnist_int8.tflite', 'wb') as f:
    f.write(tflite_int8_model)

int8_size = os.path.getsize('mnist_int8.tflite') / (1024 * 1024)
print(f"INT8 model size: {int8_size:.2f} MB")
print(f"Size reduction: {((1 - int8_size/get_model_size('mnist_baseline_savedmodel')) * 100):.1f}%")

# === Evaluate quantized models ===
print("\n=== Evaluating Quantized Models ===")

def evaluate_tflite_model(model_path, x_test, y_test):
    interpreter = tf.lite.Interpreter(model_path=model_path)
    interpreter.allocate_tensors()

    input_details = interpreter.get_input_details()
    output_details = interpreter.get_output_details()

    correct = 0
    total = len(x_test)

    for i in range(total):
        input_data = x_test[i:i+1].astype(input_details[0]['dtype'])
        interpreter.set_tensor(input_details[0]['index'], input_data)
        interpreter.invoke()
        output_data = interpreter.get_tensor(output_details[0]['index'])

        predicted = np.argmax(output_data[0])
        if predicted == y_test[i]:
            correct += 1

    accuracy = correct / total
    return accuracy

f16_acc = evaluate_tflite_model('mnist_float16.tflite', x_test, y_test)
print(f"Float16 accuracy: {f16_acc:.4f}")

int8_acc = evaluate_tflite_model('mnist_int8.tflite', x_test, y_test)
print(f"INT8 accuracy: {int8_acc:.4f}")

# Update metrics
with open('model_metrics.json', 'r') as f:
    metrics = json.load(f)

metrics['float16'] = {
    'accuracy': float(f16_acc),
    'size_mb': f16_size,
    'reduction_pct': (1 - f16_size/metrics['baseline']['size_mb']) * 100
}

metrics['int8'] = {
    'accuracy': float(int8_acc),
    'size_mb': int8_size,
    'reduction_pct': (1 - int8_size/metrics['baseline']['size_mb']) * 100
}

with open('model_metrics.json', 'w') as f:
    json.dump(metrics, f, indent=2)

print("\nMetrics saved to model_metrics.json")
```

### Step 4: Convert All Versions to TensorFlow.js
```bash
#!/bin/bash
# convert_all_models.sh

echo "Converting models to TensorFlow.js..."

# Baseline (no quantization)
echo "Converting baseline model..."
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    ./mnist_baseline_savedmodel \
    ./tfjs_models/baseline

# Float16 quantization
echo "Converting with float16 quantization..."
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --quantize_float16 \
    ./mnist_baseline_savedmodel \
    ./tfjs_models/float16

# UINT8 quantization
echo "Converting with uint8 quantization..."
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --quantize_uint8 \
    ./mnist_baseline_savedmodel \
    ./tfjs_models/uint8

echo "Comparing sizes..."
du -sh tfjs_models/*/

echo "Conversion complete!"
```

### Step 5: Create Benchmark Application
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Model Quantization Benchmark</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            padding: 30px;
            box-shadow: 0 10px 50px rgba(0,0,0,0.3);
        }

        h1 {
            color: #667eea;
            text-align: center;
            margin-bottom: 10px;
        }

        .subtitle {
            text-align: center;
            color: #666;
            margin-bottom: 30px;
        }

        .model-cards {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 20px;
            margin: 30px 0;
        }

        .model-card {
            background: #f5f5f5;
            padding: 20px;
            border-radius: 15px;
            border: 3px solid transparent;
            transition: all 0.3s;
        }

        .model-card:hover {
            border-color: #667eea;
            transform: translateY(-5px);
        }

        .model-card.selected {
            border-color: #4CAF50;
            background: #e8f5e9;
        }

        .model-card h3 {
            color: #333;
            margin-bottom: 15px;
        }

        .metric {
            display: flex;
            justify-content: space-between;
            margin: 10px 0;
            padding: 8px;
            background: white;
            border-radius: 5px;
        }

        .metric-label {
            font-weight: 600;
            color: #666;
        }

        .metric-value {
            font-weight: 700;
            color: #667eea;
        }

        .status-badge {
            display: inline-block;
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 600;
            margin-top: 10px;
        }

        .status-badge.loading {
            background: #fff3cd;
            color: #856404;
        }

        .status-badge.ready {
            background: #d4edda;
            color: #155724;
        }

        .status-badge.error {
            background: #f8d7da;
            color: #721c24;
        }

        .controls {
            display: flex;
            gap: 15px;
            justify-content: center;
            margin: 30px 0;
        }

        button {
            padding: 15px 30px;
            font-size: 16px;
            font-weight: 600;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            transition: all 0.3s;
        }

        .btn-primary {
            background: #4CAF50;
            color: white;
        }

        .btn-primary:hover {
            background: #45a049;
        }

        .btn-secondary {
            background: #2196F3;
            color: white;
        }

        button:disabled {
            background: #ccc;
            cursor: not-allowed;
        }

        .canvas-container {
            text-align: center;
            margin: 30px 0;
        }

        #drawCanvas {
            border: 3px solid #667eea;
            border-radius: 10px;
            cursor: crosshair;
            background: white;
        }

        .results {
            background: #f5f5f5;
            padding: 20px;
            border-radius: 15px;
            margin: 20px 0;
        }

        .prediction-grid {
            display: grid;
            grid-template-columns: repeat(5, 1fr);
            gap: 10px;
            margin-top: 15px;
        }

        .prediction-item {
            background: white;
            padding: 15px;
            border-radius: 10px;
            text-align: center;
        }

        .prediction-item.top {
            background: #e8f5e9;
            border: 2px solid #4CAF50;
        }

        .digit {
            font-size: 24px;
            font-weight: 700;
            color: #333;
        }

        .probability {
            font-size: 14px;
            color: #667eea;
            margin-top: 5px;
        }

        .comparison-table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
        }

        .comparison-table th,
        .comparison-table td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }

        .comparison-table th {
            background: #667eea;
            color: white;
            font-weight: 600;
        }

        .comparison-table tr:hover {
            background: #f5f5f5;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>⚡ Model Quantization Benchmark</h1>
        <p class="subtitle">Compare performance of different quantization levels</p>

        <div id="deviceInfo" style="text-align: center; padding: 15px; background: #e3f2fd; border-radius: 10px; margin-bottom: 20px;">
            Loading device information...
        </div>

        <div class="model-cards" id="modelCards"></div>

        <div class="controls">
            <button class="btn-primary" onclick="loadAllModels()" id="loadBtn">
                📥 Load All Models
            </button>
            <button class="btn-secondary" onclick="runBenchmark()" id="benchmarkBtn" disabled>
                🚀 Run Benchmark
            </button>
            <button class="btn-secondary" onclick="clearCanvas()">
                🗑️ Clear
            </button>
        </div>

        <div class="canvas-container">
            <h3>Draw a digit (0-9):</h3>
            <canvas id="drawCanvas" width="280" height="280"></canvas>
        </div>

        <div class="results" id="results" style="display: none;">
            <h3>Predictions:</h3>
            <div class="prediction-grid" id="predictions"></div>
        </div>

        <div class="results">
            <h3>Performance Comparison:</h3>
            <table class="comparison-table" id="comparisonTable">
                <thead>
                    <tr>
                        <th>Model</th>
                        <th>Size (KB)</th>
                        <th>Load Time (ms)</th>
                        <th>Inference Time (ms)</th>
                        <th>Memory (MB)</th>
                        <th>Accuracy</th>
                    </tr>
                </thead>
                <tbody id="comparisonBody"></tbody>
            </table>
        </div>
    </div>

    <script src="benchmark.js"></script>
</body>
</html>
```

### Step 6: Implement Benchmark Logic
```javascript
// benchmark.js
const MODEL_CONFIGS = [
    {
        name: 'Baseline (No Quantization)',
        path: './tfjs_models/baseline/model.json',
        type: 'baseline',
        description: 'Original model without optimization'
    },
    {
        name: 'Float16 Quantized',
        path: './tfjs_models/float16/model.json',
        type: 'float16',
        description: '~50% size reduction, minimal accuracy loss'
    },
    {
        name: 'UINT8 Quantized',
        path: './tfjs_models/uint8/model.json',
        type: 'uint8',
        description: '~75% size reduction, acceptable accuracy loss'
    }
];

const models = {};
const benchmarkResults = {};

// Device detection
async function getDeviceInfo() {
    const info = {
        platform: navigator.platform,
        cores: navigator.hardwareConcurrency || 'unknown',
        memory: navigator.deviceMemory ? `${navigator.deviceMemory}GB` : 'unknown',
        backend: await tf.getBackend(),
        webgl: tf.ENV.getBool('WEBGL_VERSION') > 0,
        wasm: typeof WebAssembly !== 'undefined'
    };

    document.getElementById('deviceInfo').innerHTML = `
        <strong>Device:</strong> ${info.platform} |
        <strong>CPU Cores:</strong> ${info.cores} |
        <strong>Memory:</strong> ${info.memory} |
        <strong>Backend:</strong> ${info.backend} |
        <strong>WebGL:</strong> ${info.webgl ? '✓' : '✗'} |
        <strong>WASM:</strong> ${info.wasm ? '✓' : '✗'}
    `;

    return info;
}

// Initialize model cards
function initializeModelCards() {
    const container = document.getElementById('modelCards');

    MODEL_CONFIGS.forEach(config => {
        const card = document.createElement('div');
        card.className = 'model-card';
        card.id = `card-${config.type}`;

        card.innerHTML = `
            <h3>${config.name}</h3>
            <p style="color: #666; font-size: 14px; margin-bottom: 15px;">
                ${config.description}
            </p>
            <div class="metric">
                <span class="metric-label">Status:</span>
                <span class="status-badge loading">Not Loaded</span>
            </div>
            <div class="metric">
                <span class="metric-label">Size:</span>
                <span class="metric-value" id="size-${config.type}">-</span>
            </div>
            <div class="metric">
                <span class="metric-label">Load Time:</span>
                <span class="metric-value" id="loadtime-${config.type}">-</span>
            </div>
            <div class="metric">
                <span class="metric-label">Inference:</span>
                <span class="metric-value" id="inference-${config.type}">-</span>
            </div>
        `;

        container.appendChild(card);
    });
}

// Load all models
async function loadAllModels() {
    document.getElementById('loadBtn').disabled = true;

    for (const config of MODEL_CONFIGS) {
        await loadModel(config);
    }

    document.getElementById('benchmarkBtn').disabled = false;
    document.getElementById('loadBtn').textContent = '✓ All Models Loaded';
}

// Load single model
async function loadModel(config) {
    const card = document.getElementById(`card-${config.type}`);
    const statusBadge = card.querySelector('.status-badge');

    statusBadge.textContent = 'Loading...';
    statusBadge.className = 'status-badge loading';

    try {
        const startTime = performance.now();

        const model = await tf.loadGraphModel(config.path);

        const loadTime = performance.now() - startTime;

        // Warm up
        await tf.tidy(() => {
            const warmup = tf.zeros([1, 28, 28, 1]);
            model.predict(warmup);
        });

        models[config.type] = model;

        // Update UI
        statusBadge.textContent = 'Ready';
        statusBadge.className = 'status-badge ready';
        document.getElementById(`loadtime-${config.type}`).textContent = `${loadTime.toFixed(0)}ms`;

        card.classList.add('selected');

        console.log(`${config.name} loaded in ${loadTime.toFixed(0)}ms`);

    } catch (error) {
        console.error(`Error loading ${config.name}:`, error);
        statusBadge.textContent = 'Error';
        statusBadge.className = 'status-badge error';
    }
}

// Run benchmark
async function runBenchmark() {
    const iterations = 100;

    for (const config of MODEL_CONFIGS) {
        if (!models[config.type]) continue;

        console.log(`Benchmarking ${config.name}...`);

        const times = [];

        for (let i = 0; i < iterations; i++) {
            const startTime = performance.now();

            await tf.tidy(() => {
                const input = tf.randomNormal([1, 28, 28, 1]);
                models[config.type].predict(input);
            });

            times.push(performance.now() - startTime);
        }

        const avgTime = times.reduce((a, b) => a + b, 0) / times.length;
        const memory = tf.memory();

        benchmarkResults[config.type] = {
            avgInferenceTime: avgTime,
            memory: memory.numBytes / (1024 * 1024)
        };

        document.getElementById(`inference-${config.type}`).textContent = `${avgTime.toFixed(2)}ms`;

        console.log(`${config.name}: ${avgTime.toFixed(2)}ms avg`);
    }

    updateComparisonTable();
    alert('Benchmark complete! Check the comparison table.');
}

// Update comparison table
function updateComparisonTable() {
    const tbody = document.getElementById('comparisonBody');
    tbody.innerHTML = '';

    MODEL_CONFIGS.forEach(config => {
        const result = benchmarkResults[config.type];
        if (!result) return;

        const row = tbody.insertRow();
        row.innerHTML = `
            <td><strong>${config.name}</strong></td>
            <td>-</td>
            <td>${document.getElementById(`loadtime-${config.type}`).textContent}</td>
            <td>${result.avgInferenceTime.toFixed(2)}ms</td>
            <td>${result.memory.toFixed(2)}MB</td>
            <td>-</td>
        `;
    });
}

// Canvas drawing
const canvas = document.getElementById('drawCanvas');
const ctx = canvas.getContext('2d');
let isDrawing = false;

canvas.addEventListener('mousedown', startDrawing);
canvas.addEventListener('mousemove', draw);
canvas.addEventListener('mouseup', stopDrawing);
canvas.addEventListener('mouseout', stopDrawing);

// Touch events
canvas.addEventListener('touchstart', (e) => {
    e.preventDefault();
    const touch = e.touches[0];
    const mouseEvent = new MouseEvent('mousedown', {
        clientX: touch.clientX,
        clientY: touch.clientY
    });
    canvas.dispatchEvent(mouseEvent);
});

canvas.addEventListener('touchmove', (e) => {
    e.preventDefault();
    const touch = e.touches[0];
    const mouseEvent = new MouseEvent('mousemove', {
        clientX: touch.clientX,
        clientY: touch.clientY
    });
    canvas.dispatchEvent(mouseEvent);
});

canvas.addEventListener('touchend', (e) => {
    e.preventDefault();
    const mouseEvent = new MouseEvent('mouseup', {});
    canvas.dispatchEvent(mouseEvent);
});

function startDrawing(e) {
    isDrawing = true;
    draw(e);
}

function draw(e) {
    if (!isDrawing) return;

    const rect = canvas.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;

    ctx.lineWidth = 20;
    ctx.lineCap = 'round';
    ctx.strokeStyle = 'black';

    ctx.lineTo(x, y);
    ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(x, y);
}

function stopDrawing() {
    isDrawing = false;
    ctx.beginPath();

    // Run prediction
    predictDigit();
}

function clearCanvas() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    document.getElementById('results').style.display = 'none';
}

// Predict digit
async function predictDigit() {
    // Get image data from canvas
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    // Preprocess
    const tensor = tf.tidy(() => {
        // Convert to grayscale and resize to 28x28
        let img = tf.browser.fromPixels(imageData, 1);
        img = tf.image.resizeBilinear(img, [28, 28]);
        img = img.div(255.0);
        img = img.expandDims(0);
        return img;
    });

    // Get predictions from all models
    const predictions = {};

    for (const config of MODEL_CONFIGS) {
        if (!models[config.type]) continue;

        const pred = await models[config.type].predict(tensor).data();
        predictions[config.type] = Array.from(pred);
    }

    tensor.dispose();

    // Display results (using baseline model)
    if (predictions.baseline) {
        displayPredictions(predictions.baseline);
    }
}

// Display predictions
function displayPredictions(predictions) {
    const resultsDiv = document.getElementById('results');
    const predictionsDiv = document.getElementById('predictions');

    resultsDiv.style.display = 'block';
    predictionsDiv.innerHTML = '';

    const topPredictions = predictions
        .map((prob, digit) => ({ digit, prob }))
        .sort((a, b) => b.prob - a.prob)
        .slice(0, 5);

    topPredictions.forEach((pred, index) => {
        const item = document.createElement('div');
        item.className = 'prediction-item' + (index === 0 ? ' top' : '');
        item.innerHTML = `
            <div class="digit">${pred.digit}</div>
            <div class="probability">${(pred.prob * 100).toFixed(1)}%</div>
        `;
        predictionsDiv.appendChild(item);
    });
}

// Initialize
getDeviceInfo();
initializeModelCards();
```

## Expected Outputs

1. **Quantized Models**:
   - Float16: ~50% size reduction
   - UINT8: ~75% size reduction
   - Minimal accuracy loss (<2%)

2. **Performance Metrics**:
   - Baseline: ~1-2 MB, 20-50ms inference
   - Float16: ~500KB-1MB, 15-40ms inference
   - UINT8: ~250-500KB, 10-30ms inference

3. **Device Compatibility**:
   - Works on low-end devices
   - Adaptive model loading
   - Fallback options available

## Bonus Challenges

- [ ] Implement quantization-aware training
- [ ] Add model pruning before quantization
- [ ] Create auto-selection based on device tier
- [ ] Implement progressive model loading
- [ ] Add knowledge distillation
- [ ] Create benchmark dashboard
- [ ] Test on real IoT devices
- [ ] Implement mixed precision models
- [ ] Add A/B testing framework
- [ ] Create size/accuracy tradeoff visualizer

## Resources

- [TensorFlow Model Optimization](https://www.tensorflow.org/model_optimization)
- [Post-Training Quantization](https://www.tensorflow.org/lite/performance/post_training_quantization)
- [TensorFlow.js Quantization](https://www.tensorflow.org/js/guide/conversion#quantization)
- [Model Optimization Toolkit](https://www.tensorflow.org/model_optimization/guide)

## Success Criteria

- All quantization levels convert successfully
- Size reductions match expectations (50%, 75%)
- Accuracy loss is acceptable (<2%)
- Performance improves on low-end devices
- Benchmark tool works correctly
- Device detection and adaptation functional
- Models load and run on target devices
- Clear performance comparison available
