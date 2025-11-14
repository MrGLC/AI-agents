# Project 1: Convert Keras Image Classifier to TensorFlow.js

## Overview
Learn how to convert a Keras image classification model to TensorFlow.js format and deploy it in a web browser. This project covers the complete workflow from training a model in Python to running inference entirely in the browser.

## Difficulty Level
Beginner to Intermediate

## Learning Objectives
- Train a CNN image classifier using Keras
- Convert Keras models to TensorFlow.js format
- Understand TFJS model formats (Layers Model vs Graph Model)
- Load and run TFJS models in the browser
- Handle image preprocessing in JavaScript
- Implement real-time inference with proper tensor management

## Technical Stack
- **Backend**: Python 3.8+, TensorFlow 2.x, Keras
- **Conversion**: tensorflowjs_converter
- **Frontend**: HTML5, JavaScript ES6+, TensorFlow.js
- **Dataset**: CIFAR-10 or Fashion-MNIST
- **Server**: Python http.server or Node.js http-server

## Project Requirements

### 1. Model Training (Python)
- Train a CNN classifier on CIFAR-10 or Fashion-MNIST
- Achieve >70% validation accuracy
- Save model in Keras H5 format
- Document model architecture and performance

### 2. Model Conversion
- Install tensorflowjs converter
- Convert H5 model to TFJS Layers Model format
- Verify conversion produces model.json and weight shards
- Compare model size before and after conversion

### 3. Browser Integration
- Create HTML interface with image upload
- Load TFJS model in browser
- Implement image preprocessing pipeline
- Display predictions with confidence scores
- Proper tensor memory management

### 4. Performance Optimization
- Warm up model on first load
- Implement loading indicator
- Use tf.tidy() for memory management
- Cache model in browser storage

## Step-by-Step Implementation

### Step 1: Setup Python Environment
```bash
# Create virtual environment
python -m venv tfjs-env
source tfjs-env/bin/activate  # On Windows: tfjs-env\Scripts\activate

# Install dependencies
pip install tensorflow tensorflowjs numpy matplotlib pillow
```

### Step 2: Train Keras Model
```python
import tensorflow as tf
from tensorflow import keras
import numpy as np

# Load CIFAR-10 dataset
(x_train, y_train), (x_test, y_test) = keras.datasets.cifar10.load_data()

# Normalize pixel values
x_train = x_train.astype('float32') / 255.0
x_test = x_test.astype('float32') / 255.0

# Define model architecture
model = keras.Sequential([
    keras.layers.Conv2D(32, (3, 3), activation='relu', input_shape=(32, 32, 3)),
    keras.layers.MaxPooling2D((2, 2)),
    keras.layers.Conv2D(64, (3, 3), activation='relu'),
    keras.layers.MaxPooling2D((2, 2)),
    keras.layers.Conv2D(64, (3, 3), activation='relu'),
    keras.layers.Flatten(),
    keras.layers.Dense(64, activation='relu'),
    keras.layers.Dropout(0.5),
    keras.layers.Dense(10, activation='softmax')
])

# Compile model
model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Train model
history = model.fit(
    x_train, y_train,
    epochs=10,
    batch_size=64,
    validation_split=0.2,
    verbose=1
)

# Evaluate model
test_loss, test_acc = model.evaluate(x_test, y_test, verbose=0)
print(f'Test accuracy: {test_acc:.4f}')

# Save model
model.save('cifar10_model.h5')
print('Model saved as cifar10_model.h5')
```

### Step 3: Convert Model to TensorFlow.js
```bash
# Install tensorflowjs converter
pip install tensorflowjs

# Convert Keras H5 to TFJS Layers Model
tensorflowjs_converter \
    --input_format=keras \
    cifar10_model.h5 \
    ./tfjs_model

# Check output files
ls -lh tfjs_model/
# Should see: model.json, group1-shard1of1.bin
```

### Step 4: Create HTML Interface
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CIFAR-10 Classifier - TensorFlow.js</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        #imageCanvas {
            border: 2px solid #333;
            margin: 20px 0;
        }
        #predictions {
            margin-top: 20px;
        }
        .prediction-bar {
            margin: 10px 0;
        }
        .bar {
            height: 30px;
            background-color: #4CAF50;
            color: white;
            padding: 5px;
            border-radius: 4px;
        }
        #loading {
            display: none;
            color: #666;
        }
    </style>
</head>
<body>
    <h1>CIFAR-10 Image Classifier</h1>
    <p>Upload an image to classify (airplane, automobile, bird, cat, deer, dog, frog, horse, ship, truck)</p>

    <input type="file" id="imageUpload" accept="image/*">
    <div id="loading">Loading model...</div>

    <canvas id="imageCanvas" width="200" height="200"></canvas>

    <div id="predictions"></div>

    <script src="app.js"></script>
</body>
</html>
```

### Step 5: Implement JavaScript Inference
```javascript
// app.js
const CIFAR10_CLASSES = [
    'airplane', 'automobile', 'bird', 'cat', 'deer',
    'dog', 'frog', 'horse', 'ship', 'truck'
];

let model = null;

// Load the model
async function loadModel() {
    const loadingDiv = document.getElementById('loading');
    loadingDiv.style.display = 'block';

    try {
        model = await tf.loadLayersModel('./tfjs_model/model.json');
        console.log('Model loaded successfully');

        // Warm up the model
        tf.tidy(() => {
            const warmup = tf.zeros([1, 32, 32, 3]);
            model.predict(warmup);
        });

        loadingDiv.textContent = 'Model ready!';
        setTimeout(() => loadingDiv.style.display = 'none', 2000);
    } catch (error) {
        console.error('Error loading model:', error);
        loadingDiv.textContent = 'Error loading model';
    }
}

// Preprocess image
function preprocessImage(imageElement) {
    return tf.tidy(() => {
        // Convert image to tensor
        let tensor = tf.browser.fromPixels(imageElement);

        // Resize to 32x32
        tensor = tf.image.resizeBilinear(tensor, [32, 32]);

        // Normalize to [0, 1]
        tensor = tensor.div(255.0);

        // Add batch dimension
        tensor = tensor.expandDims(0);

        return tensor;
    });
}

// Make prediction
async function predict(imageElement) {
    if (!model) {
        alert('Model not loaded yet!');
        return;
    }

    const preprocessed = preprocessImage(imageElement);

    const predictions = await model.predict(preprocessed).data();

    // Clean up tensor
    preprocessed.dispose();

    // Get top 5 predictions
    const topPredictions = Array.from(predictions)
        .map((prob, index) => ({
            class: CIFAR10_CLASSES[index],
            probability: prob
        }))
        .sort((a, b) => b.probability - a.probability)
        .slice(0, 5);

    displayPredictions(topPredictions);
}

// Display predictions
function displayPredictions(predictions) {
    const predictionsDiv = document.getElementById('predictions');
    predictionsDiv.innerHTML = '<h2>Predictions:</h2>';

    predictions.forEach(pred => {
        const percentage = (pred.probability * 100).toFixed(2);
        const barWidth = percentage;

        predictionsDiv.innerHTML += `
            <div class="prediction-bar">
                <strong>${pred.class}</strong>: ${percentage}%
                <div class="bar" style="width: ${barWidth}%"></div>
            </div>
        `;
    });
}

// Handle image upload
document.getElementById('imageUpload').addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (event) => {
        const img = new Image();
        img.onload = () => {
            // Draw image on canvas
            const canvas = document.getElementById('imageCanvas');
            const ctx = canvas.getContext('2d');
            ctx.drawImage(img, 0, 0, 200, 200);

            // Make prediction
            predict(img);
        };
        img.src = event.target.result;
    };
    reader.readAsDataURL(file);
});

// Load model on page load
loadModel();
```

### Step 6: Serve and Test
```bash
# Serve files using Python
python -m http.server 8000

# Or using Node.js
# npx http-server -p 8000

# Open browser to http://localhost:8000
```

### Step 7: Verify Conversion Quality
```python
# verify_conversion.py
import tensorflow as tf
import numpy as np
import tensorflowjs as tfjs

# Load original Keras model
keras_model = tf.keras.models.load_model('cifar10_model.h5')

# Create test input
test_input = np.random.rand(1, 32, 32, 3).astype(np.float32)

# Get Keras prediction
keras_pred = keras_model.predict(test_input)

print("Keras predictions:", keras_pred[0])
print("\nNote: Compare these with TFJS predictions in browser console")
print("They should be nearly identical (within floating point precision)")
```

## Expected Outputs

1. **Trained Model Files**:
   - `cifar10_model.h5` (~2-5 MB)
   - Training accuracy >70%

2. **Converted TFJS Model**:
   - `tfjs_model/model.json` (architecture)
   - `tfjs_model/group1-shard1of1.bin` (weights)
   - Total size similar to H5 file

3. **Working Web Application**:
   - Model loads in browser (<5 seconds)
   - Image upload and display works
   - Predictions appear in <2 seconds
   - Top 5 classes shown with probabilities
   - Memory usage stable (no leaks)

4. **Browser Console Output**:
   ```
   Model loaded successfully
   Prediction complete
   Memory: X tensors, Y MB
   ```

## Bonus Challenges

- [ ] Add quantization (float16 or uint8) to reduce model size
- [ ] Implement model caching using IndexedDB
- [ ] Add webcam support for real-time classification
- [ ] Create a comparison view showing original vs preprocessed image
- [ ] Add data augmentation preview before prediction
- [ ] Implement batch prediction for multiple images
- [ ] Add model performance metrics display (inference time, FPS)
- [ ] Create a confusion matrix from test predictions
- [ ] Deploy to GitHub Pages or Netlify
- [ ] Add Progressive Web App (PWA) features

## Resources

- [TensorFlow.js Documentation](https://www.tensorflow.org/js)
- [tensorflowjs_converter Guide](https://www.tensorflow.org/js/guide/conversion)
- [TFJS Layers Model API](https://js.tensorflow.org/api/latest/#loadLayersModel)
- [Keras Model Saving](https://www.tensorflow.org/guide/keras/save_and_serialize)
- [CIFAR-10 Dataset](https://www.cs.toronto.edu/~kriz/cifar.html)
- [TensorFlow.js Examples](https://github.com/tensorflow/tfjs-examples)
- [Browser ML Best Practices](https://www.tensorflow.org/js/guide/platform_environment)

## Success Criteria

- Keras model trains successfully with >70% accuracy
- Model converts without errors or warnings
- Converted model size is reasonable (<10 MB)
- Web page loads model successfully
- Image upload and display works correctly
- Predictions are accurate and match Keras model output
- Inference completes in <2 seconds
- No memory leaks (stable tensor count)
- Application works in Chrome, Firefox, and Safari
- Code is well-commented and organized
