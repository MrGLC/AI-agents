# Project 3: Convert Pre-trained MobileNet for Mobile Browser

## Overview
Learn how to convert and optimize a pre-trained MobileNet model for mobile browser deployment. This project emphasizes model optimization techniques like quantization, size reduction, and mobile-specific performance tuning to create a fast, efficient image classifier that works well on smartphones and tablets.

## Difficulty Level
Intermediate

## Learning Objectives
- Convert pre-trained TensorFlow models to TensorFlow.js
- Apply quantization techniques (float16, uint8) for size reduction
- Optimize models specifically for mobile browsers
- Implement responsive web design for mobile devices
- Handle touch interactions and device orientation
- Optimize loading and inference for slower networks
- Test across different mobile browsers and devices

## Technical Stack
- **Model**: MobileNetV2 (pre-trained on ImageNet)
- **Conversion**: tensorflowjs_converter with quantization
- **Frontend**: HTML5, JavaScript ES6+, TensorFlow.js
- **Mobile Features**: Touch events, device orientation, camera access
- **Optimization**: Service Workers, lazy loading, compression
- **Testing**: Chrome DevTools mobile emulation, real devices

## Project Requirements

### 1. Model Conversion and Optimization
- Download pre-trained MobileNetV2 from TensorFlow Hub
- Convert to TensorFlow.js Graph Model format
- Apply float16 quantization for size reduction
- Compare original vs quantized model sizes and accuracy

### 2. Mobile-Optimized Web Interface
- Responsive design for various screen sizes
- Touch-friendly controls and gestures
- Camera integration for real-time capture
- Offline functionality with Service Workers
- Progressive loading with visual feedback

### 3. Performance Optimization
- Lazy load model only when needed
- Compress model files with gzip
- Cache model in browser storage
- Optimize for 3G/4G network conditions
- Minimize memory usage on mobile devices

### 4. Mobile Testing
- Test on iOS Safari and Chrome
- Test on Android Chrome and Firefox
- Verify performance on different device tiers
- Handle low-memory scenarios gracefully

## Step-by-Step Implementation

### Step 1: Setup and Download Pre-trained Model
```bash
# Create project directory
mkdir mobilenet-mobile
cd mobilenet-mobile

# Create Python virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install tensorflow tensorflow-hub tensorflowjs numpy pillow
```

```python
# download_mobilenet.py
import tensorflow as tf
import tensorflow_hub as hub
import numpy as np
from PIL import Image

# Download MobileNetV2 from TensorFlow Hub
print("Downloading MobileNetV2...")
mobilenet_url = "https://tfhub.dev/google/imagenet/mobilenet_v2_100_224/classification/5"
model = tf.keras.Sequential([
    hub.KerasLayer(mobilenet_url, input_shape=(224, 224, 3))
])

# Test the model
print("Testing model...")
test_image = np.random.rand(1, 224, 224, 3).astype(np.float32)
predictions = model.predict(test_image)
print(f"Model output shape: {predictions.shape}")

# Save as SavedModel format
print("Saving model...")
model.save('mobilenet_savedmodel', save_format='tf')
print("Model saved successfully!")

# Get model size
import os
def get_dir_size(path):
    total = 0
    for dirpath, dirnames, filenames in os.walk(path):
        for f in filenames:
            fp = os.path.join(dirpath, f)
            total += os.path.getsize(fp)
    return total

size_mb = get_dir_size('mobilenet_savedmodel') / (1024 * 1024)
print(f"SavedModel size: {size_mb:.2f} MB")
```

### Step 2: Convert with Different Quantization Levels
```bash
# Convert without quantization (baseline)
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --signature_name=serving_default \
    --saved_model_tags=serve \
    ./mobilenet_savedmodel \
    ./mobilenet_tfjs_original

# Convert with float16 quantization (recommended for mobile)
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --quantize_float16 \
    --signature_name=serving_default \
    --saved_model_tags=serve \
    ./mobilenet_savedmodel \
    ./mobilenet_tfjs_float16

# Convert with uint8 quantization (maximum compression)
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --quantize_uint8 \
    --signature_name=serving_default \
    --saved_model_tags=serve \
    ./mobilenet_savedmodel \
    ./mobilenet_tfjs_uint8

# Compare sizes
echo "Model Sizes:"
du -sh mobilenet_tfjs_original
du -sh mobilenet_tfjs_float16
du -sh mobilenet_tfjs_uint8
```

### Step 3: Download ImageNet Labels
```python
# download_labels.py
import urllib.request
import json

# Download ImageNet labels
labels_url = "https://storage.googleapis.com/download.tensorflow.org/data/ImageNetLabels.txt"
urllib.request.urlretrieve(labels_url, "imagenet_labels.txt")

# Convert to JSON for easier use in JavaScript
with open('imagenet_labels.txt', 'r') as f:
    labels = [line.strip() for line in f.readlines()]

with open('imagenet_labels.json', 'w') as f:
    json.dump(labels, f)

print(f"Downloaded {len(labels)} ImageNet labels")
```

### Step 4: Create Mobile-Optimized HTML
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="mobile-web-app-capable" content="yes">
    <title>MobileNet Image Classifier</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 10px;
            color: #333;
        }

        .container {
            max-width: 600px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            padding: 20px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.3);
        }

        h1 {
            text-align: center;
            color: #667eea;
            margin-bottom: 20px;
            font-size: 24px;
        }

        #imagePreview {
            width: 100%;
            max-height: 300px;
            object-fit: contain;
            border-radius: 10px;
            margin: 10px 0;
            display: none;
            border: 2px solid #ddd;
        }

        .button-group {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 10px;
            margin: 15px 0;
        }

        button {
            padding: 15px;
            font-size: 16px;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            transition: all 0.3s;
            font-weight: 600;
            touch-action: manipulation;
        }

        #uploadBtn {
            background: #667eea;
            color: white;
            grid-column: span 2;
        }

        #cameraBtn {
            background: #4CAF50;
            color: white;
        }

        #classifyBtn {
            background: #FF6B6B;
            color: white;
        }

        button:active {
            transform: scale(0.95);
        }

        button:disabled {
            background: #ccc;
            cursor: not-allowed;
        }

        #status {
            text-align: center;
            padding: 15px;
            margin: 10px 0;
            border-radius: 10px;
            background: #e3f2fd;
            color: #1976d2;
            font-weight: 500;
        }

        #predictions {
            margin-top: 15px;
        }

        .prediction-item {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 12px;
            margin: 8px 0;
            background: #f5f5f5;
            border-radius: 8px;
            border-left: 4px solid #667eea;
        }

        .prediction-item:first-child {
            background: #e8f5e9;
            border-left-color: #4CAF50;
        }

        .prediction-label {
            font-weight: 600;
            flex: 1;
        }

        .prediction-confidence {
            color: #667eea;
            font-weight: 700;
            font-size: 18px;
        }

        #fileInput, #cameraInput {
            display: none;
        }

        .loader {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #667eea;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
            margin: 20px auto;
            display: none;
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }

        @media (max-width: 480px) {
            .container {
                padding: 15px;
                border-radius: 15px;
            }

            h1 {
                font-size: 20px;
            }

            button {
                padding: 12px;
                font-size: 14px;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📱 MobileNet Classifier</h1>

        <div id="status">Loading model...</div>
        <div class="loader" id="loader"></div>

        <img id="imagePreview" alt="Preview">

        <div class="button-group">
            <button id="uploadBtn" onclick="document.getElementById('fileInput').click()">
                📁 Choose Image
            </button>
            <button id="cameraBtn" onclick="document.getElementById('cameraInput').click()">
                📷 Take Photo
            </button>
            <button id="classifyBtn" onclick="classifyImage()" disabled>
                🔍 Classify
            </button>
        </div>

        <input type="file" id="fileInput" accept="image/*">
        <input type="file" id="cameraInput" accept="image/*" capture="environment">

        <div id="predictions"></div>
    </div>

    <script src="mobilenet.js"></script>
</body>
</html>
```

### Step 5: Implement JavaScript with Mobile Optimizations
```javascript
// mobilenet.js
let model = null;
let labels = [];
let currentImage = null;

const statusDiv = document.getElementById('status');
const loaderDiv = document.getElementById('loader');
const imagePreview = document.getElementById('imagePreview');
const classifyBtn = document.getElementById('classifyBtn');
const predictionsDiv = document.getElementById('predictions');

// Load ImageNet labels
async function loadLabels() {
    const response = await fetch('imagenet_labels.json');
    labels = await response.json();
    console.log(`Loaded ${labels.length} labels`);
}

// Load TensorFlow.js model
async function loadModel() {
    statusDiv.textContent = 'Loading model...';
    loaderDiv.style.display = 'block';

    try {
        // Load labels first
        await loadLabels();

        // Check if model is cached
        const cacheKey = 'mobilenet-model-v1';
        const cachedModel = localStorage.getItem(cacheKey);

        // Load float16 quantized model (best balance for mobile)
        const modelPath = './mobilenet_tfjs_float16/model.json';
        model = await tf.loadGraphModel(modelPath);

        console.log('Model loaded successfully');

        // Warm up model
        tf.tidy(() => {
            const warmupInput = tf.zeros([1, 224, 224, 3]);
            model.predict(warmupInput);
        });

        statusDiv.textContent = 'Model ready! Select an image.';
        loaderDiv.style.display = 'none';

        // Store cache flag
        localStorage.setItem(cacheKey, 'loaded');

    } catch (error) {
        console.error('Error loading model:', error);
        statusDiv.textContent = 'Error loading model. Please refresh.';
        statusDiv.style.background = '#ffebee';
        statusDiv.style.color = '#c62828';
        loaderDiv.style.display = 'none';
    }
}

// Preprocess image for MobileNet
function preprocessImage(imageElement) {
    return tf.tidy(() => {
        // Convert to tensor
        let tensor = tf.browser.fromPixels(imageElement);

        // Resize to 224x224
        tensor = tf.image.resizeBilinear(tensor, [224, 224]);

        // Normalize to [-1, 1] (MobileNet preprocessing)
        tensor = tensor.div(127.5).sub(1);

        // Add batch dimension
        tensor = tensor.expandDims(0);

        return tensor;
    });
}

// Classify image
async function classifyImage() {
    if (!model || !currentImage) {
        alert('Please load an image first');
        return;
    }

    classifyBtn.disabled = true;
    statusDiv.textContent = 'Classifying...';
    loaderDiv.style.display = 'block';
    predictionsDiv.innerHTML = '';

    try {
        const startTime = performance.now();

        // Preprocess image
        const preprocessed = preprocessImage(currentImage);

        // Run inference
        const predictions = await model.predict(preprocessed).data();

        // Clean up tensor
        preprocessed.dispose();

        const endTime = performance.now();
        const inferenceTime = (endTime - startTime).toFixed(0);

        // Get top 5 predictions
        const topPredictions = Array.from(predictions)
            .map((prob, index) => ({
                label: labels[index] || `Class ${index}`,
                probability: prob
            }))
            .sort((a, b) => b.probability - a.probability)
            .slice(0, 5);

        // Display results
        displayPredictions(topPredictions, inferenceTime);

        statusDiv.textContent = `Classification complete (${inferenceTime}ms)`;
        loaderDiv.style.display = 'none';

    } catch (error) {
        console.error('Classification error:', error);
        statusDiv.textContent = 'Error during classification';
        statusDiv.style.background = '#ffebee';
        loaderDiv.style.display = 'none';
    }

    classifyBtn.disabled = false;
}

// Display predictions
function displayPredictions(predictions, inferenceTime) {
    predictionsDiv.innerHTML = `
        <h3 style="margin-bottom: 10px;">Top Predictions:</h3>
    `;

    predictions.forEach((pred, index) => {
        const percentage = (pred.probability * 100).toFixed(1);
        const item = document.createElement('div');
        item.className = 'prediction-item';
        item.innerHTML = `
            <span class="prediction-label">${index + 1}. ${pred.label}</span>
            <span class="prediction-confidence">${percentage}%</span>
        `;
        predictionsDiv.appendChild(item);
    });

    // Add performance info
    const perfInfo = document.createElement('div');
    perfInfo.style.cssText = 'text-align: center; margin-top: 15px; color: #666; font-size: 14px;';
    perfInfo.textContent = `⚡ Inference time: ${inferenceTime}ms`;
    predictionsDiv.appendChild(perfInfo);
}

// Handle image selection
function handleImageSelect(e) {
    const file = e.target.files[0];
    if (!file) return;

    // Check file size (mobile optimization)
    const maxSizeMB = 5;
    if (file.size > maxSizeMB * 1024 * 1024) {
        alert(`Image too large. Please select an image smaller than ${maxSizeMB}MB.`);
        return;
    }

    const reader = new FileReader();
    reader.onload = (event) => {
        const img = new Image();
        img.onload = () => {
            currentImage = img;
            imagePreview.src = event.target.result;
            imagePreview.style.display = 'block';
            classifyBtn.disabled = false;
            statusDiv.textContent = 'Image loaded. Tap "Classify" to analyze.';
            predictionsDiv.innerHTML = '';
        };
        img.src = event.target.result;
    };
    reader.readAsDataURL(file);
}

// Event listeners
document.getElementById('fileInput').addEventListener('change', handleImageSelect);
document.getElementById('cameraInput').addEventListener('change', handleImageSelect);

// Handle orientation change (mobile)
window.addEventListener('orientationchange', () => {
    setTimeout(() => {
        window.scrollTo(0, 0);
    }, 100);
});

// Initialize
loadModel();

// Log memory usage (debugging)
setInterval(() => {
    const memInfo = tf.memory();
    console.log(`Tensors: ${memInfo.numTensors}, Memory: ${(memInfo.numBytes / 1024 / 1024).toFixed(2)}MB`);
}, 10000);
```

### Step 6: Add Service Worker for Offline Support
```javascript
// service-worker.js
const CACHE_NAME = 'mobilenet-cache-v1';
const urlsToCache = [
    './',
    './index.html',
    './mobilenet.js',
    './imagenet_labels.json',
    './mobilenet_tfjs_float16/model.json',
    // Add weight shard files
    './mobilenet_tfjs_float16/group1-shard1of1.bin',
    'https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0/dist/tf.min.js'
];

// Install event
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then((cache) => {
                console.log('Opened cache');
                return cache.addAll(urlsToCache);
            })
    );
});

// Fetch event
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request)
            .then((response) => {
                // Cache hit - return response
                if (response) {
                    return response;
                }
                return fetch(event.request);
            }
        )
    );
});

// Activate event
self.addEventListener('activate', (event) => {
    const cacheWhitelist = [CACHE_NAME];
    event.waitUntil(
        caches.keys().then((cacheNames) => {
            return Promise.all(
                cacheNames.map((cacheName) => {
                    if (cacheWhitelist.indexOf(cacheName) === -1) {
                        return caches.delete(cacheName);
                    }
                })
            );
        })
    );
});
```

### Step 7: Create Manifest for PWA
```json
{
    "name": "MobileNet Image Classifier",
    "short_name": "MobileNet",
    "description": "AI-powered image classification in your browser",
    "start_url": "./",
    "display": "standalone",
    "background_color": "#667eea",
    "theme_color": "#667eea",
    "orientation": "portrait",
    "icons": [
        {
            "src": "icon-192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

## Expected Outputs

1. **Converted Models**:
   - Original: ~14 MB
   - Float16: ~7 MB (50% reduction)
   - Uint8: ~3.5 MB (75% reduction)

2. **Performance Metrics (Mobile)**:
   - Model load time: <5 seconds on 4G
   - Inference time: 100-500ms (depending on device)
   - Memory usage: <100MB
   - Offline functionality working

3. **Mobile Features**:
   - Responsive design on all screen sizes
   - Touch-friendly interface
   - Camera integration working
   - Smooth animations and transitions
   - Works in portrait and landscape

4. **Accuracy**:
   - Top-1 accuracy: ~65-70% (ImageNet)
   - Float16 vs original: <1% accuracy drop
   - Top-5 accuracy: ~85-90%

## Bonus Challenges

- [ ] Implement progressive model loading (load smaller model first)
- [ ] Add image preprocessing visualization
- [ ] Create batch classification for multiple images
- [ ] Add haptic feedback on touch devices
- [ ] Implement swipe gestures for image navigation
- [ ] Add share functionality for results
- [ ] Create performance benchmarking tool
- [ ] Add WebGL backend selection
- [ ] Implement WASM backend fallback
- [ ] Add dark mode support
- [ ] Create installable PWA with offline support
- [ ] Add localization for multiple languages

## Resources

- [TensorFlow Hub MobileNet](https://tfhub.dev/google/imagenet/mobilenet_v2_100_224/classification/5)
- [TensorFlow.js Model Quantization](https://www.tensorflow.org/js/guide/conversion#quantization)
- [Web Performance Optimization](https://web.dev/performance/)
- [Progressive Web Apps](https://web.dev/progressive-web-apps/)
- [Mobile Web Best Practices](https://developers.google.com/web/fundamentals/design-and-ux/principles)
- [Service Workers Guide](https://developers.google.com/web/fundamentals/primers/service-workers)
- [ImageNet Classes](http://image-net.org/challenges/LSVRC/2012/browse-synsets)

## Success Criteria

- Model converts successfully with float16 quantization
- Size reduction of ~50% from original
- Web app loads and runs on mobile browsers (iOS Safari, Android Chrome)
- Inference time <500ms on mid-range mobile devices
- Camera integration works on both iOS and Android
- Responsive design works on screens 320px-768px width
- Service Worker caches all necessary files
- App works offline after initial load
- No memory leaks (stable tensor count over time)
- Touch interactions are smooth and responsive
- App is installable as PWA
- Works both online and offline
