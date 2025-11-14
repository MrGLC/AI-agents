# Project 4: Build Browser-Based Object Detection with TensorFlow.js

## Overview
Create a real-time object detection application that runs entirely in the browser using TensorFlow.js. Convert a COCO-SSD or custom YOLO model to detect and localize multiple objects in images and video streams, with bounding boxes and confidence scores displayed in real-time.

## Difficulty Level
Advanced

## Learning Objectives
- Convert object detection models (COCO-SSD, YOLO) to TensorFlow.js
- Implement bounding box drawing and visualization
- Handle multiple object detections simultaneously
- Optimize for real-time video processing
- Implement non-maximum suppression (NMS) in JavaScript
- Work with complex model outputs (boxes, scores, classes)
- Optimize performance for 30+ FPS detection

## Technical Stack
- **Model**: COCO-SSD or YOLOv5 (TensorFlow version)
- **Conversion**: tensorflowjs_converter
- **Frontend**: HTML5 Canvas, JavaScript ES6+
- **ML Framework**: TensorFlow.js
- **Video**: WebRTC getUserMedia API
- **Visualization**: Canvas 2D rendering
- **Performance**: Web Workers for parallel processing

## Project Requirements

### 1. Model Preparation
- Use pre-trained COCO-SSD model or convert custom detector
- Support detection of 80+ COCO classes
- Handle variable input sizes
- Optimize for browser inference speed

### 2. Detection Pipeline
- Load and initialize object detection model
- Preprocess images/video frames
- Run inference and parse outputs
- Apply confidence threshold filtering
- Implement non-maximum suppression
- Draw bounding boxes with labels

### 3. Real-Time Video Detection
- Access webcam via getUserMedia
- Process video frames at 15-30 FPS
- Display detections with minimal latency
- Handle multiple objects in frame
- Smooth bounding box updates

### 4. User Interface
- Video preview with detection overlay
- Confidence threshold slider
- Class filter (select which objects to detect)
- FPS counter and performance metrics
- Screenshot/recording functionality

## Step-by-Step Implementation

### Step 1: Setup Project
```bash
mkdir object-detection-browser
cd object-detection-browser

# Create project structure
mkdir models css js
touch index.html js/app.js js/detector.js css/style.css
```

### Step 2: Option A - Use Pre-built COCO-SSD Model
```javascript
// detector.js - Using pre-built COCO-SSD
class ObjectDetector {
    constructor() {
        this.model = null;
        this.isModelLoaded = false;
    }

    async loadModel() {
        console.log('Loading COCO-SSD model...');

        // Load pre-built COCO-SSD from NPM/CDN
        // This model is already in TFJS format
        this.model = await cocoSsd.load({
            base: 'mobilenet_v2'  // or 'lite_mobilenet_v2' for faster inference
        });

        this.isModelLoaded = true;
        console.log('Model loaded successfully');
    }

    async detect(imageElement, scoreThreshold = 0.5) {
        if (!this.isModelLoaded) {
            throw new Error('Model not loaded');
        }

        const predictions = await this.model.detect(imageElement);

        // Filter by confidence threshold
        return predictions.filter(pred => pred.score >= scoreThreshold);
    }
}
```

### Step 2: Option B - Convert Custom YOLOv5 Model
```python
# convert_yolo.py
import tensorflow as tf
import numpy as np

# Assuming you have a YOLOv5 model in TensorFlow format
# This example shows the conversion process

def convert_yolo_to_tfjs():
    # Load your TensorFlow YOLO model
    model_path = 'yolov5_saved_model'

    # If coming from PyTorch YOLOv5:
    # 1. Export PyTorch to ONNX
    # 2. Convert ONNX to TensorFlow (using onnx-tf)
    # 3. Save as SavedModel

    # Load the TensorFlow model
    model = tf.saved_model.load(model_path)

    # Test inference
    dummy_input = tf.random.uniform([1, 640, 640, 3])
    output = model(dummy_input)

    print(f"Model output shape: {output.shape}")

    # Convert to TensorFlow.js
    # Use command line:
    # tensorflowjs_converter \
    #     --input_format=tf_saved_model \
    #     --output_format=tfjs_graph_model \
    #     --quantize_float16 \
    #     ./yolov5_saved_model \
    #     ./yolo_tfjs

# COCO class names
COCO_CLASSES = [
    'person', 'bicycle', 'car', 'motorcycle', 'airplane', 'bus', 'train',
    'truck', 'boat', 'traffic light', 'fire hydrant', 'stop sign',
    'parking meter', 'bench', 'bird', 'cat', 'dog', 'horse', 'sheep',
    'cow', 'elephant', 'bear', 'zebra', 'giraffe', 'backpack', 'umbrella',
    'handbag', 'tie', 'suitcase', 'frisbee', 'skis', 'snowboard',
    'sports ball', 'kite', 'baseball bat', 'baseball glove', 'skateboard',
    'surfboard', 'tennis racket', 'bottle', 'wine glass', 'cup', 'fork',
    'knife', 'spoon', 'bowl', 'banana', 'apple', 'sandwich', 'orange',
    'broccoli', 'carrot', 'hot dog', 'pizza', 'donut', 'cake', 'chair',
    'couch', 'potted plant', 'bed', 'dining table', 'toilet', 'tv',
    'laptop', 'mouse', 'remote', 'keyboard', 'cell phone', 'microwave',
    'oven', 'toaster', 'sink', 'refrigerator', 'book', 'clock', 'vase',
    'scissors', 'teddy bear', 'hair drier', 'toothbrush'
]

import json
with open('coco_classes.json', 'w') as f:
    json.dump(COCO_CLASSES, f)
```

### Step 3: Create HTML Interface
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Object Detection - TensorFlow.js</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd@2.2.3"></script>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>🎯 Real-Time Object Detection</h1>
            <p>Powered by TensorFlow.js</p>
        </header>

        <div class="main-content">
            <div class="video-container">
                <video id="webcam" autoplay playsinline></video>
                <canvas id="canvas"></canvas>
            </div>

            <div class="controls">
                <div class="control-group">
                    <button id="startBtn" class="btn btn-primary">
                        📹 Start Webcam
                    </button>
                    <button id="stopBtn" class="btn btn-danger" disabled>
                        ⏹️ Stop
                    </button>
                    <button id="screenshotBtn" class="btn btn-secondary" disabled>
                        📸 Screenshot
                    </button>
                </div>

                <div class="control-group">
                    <label for="confidenceSlider">
                        Confidence Threshold: <span id="confidenceValue">0.50</span>
                    </label>
                    <input type="range" id="confidenceSlider" min="0" max="100" value="50">
                </div>

                <div class="control-group">
                    <label>
                        <input type="checkbox" id="showLabels" checked>
                        Show Labels
                    </label>
                    <label>
                        <input type="checkbox" id="showConfidence" checked>
                        Show Confidence
                    </label>
                </div>

                <div class="stats">
                    <div class="stat-item">
                        <span class="stat-label">FPS:</span>
                        <span id="fps" class="stat-value">0</span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-label">Detections:</span>
                        <span id="detectionCount" class="stat-value">0</span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-label">Inference:</span>
                        <span id="inferenceTime" class="stat-value">0ms</span>
                    </div>
                </div>
            </div>
        </div>

        <div id="status" class="status">Loading model...</div>

        <div class="detected-objects">
            <h3>Detected Objects:</h3>
            <div id="objectsList"></div>
        </div>
    </div>

    <script src="js/detector.js"></script>
    <script src="js/app.js"></script>
</body>
</html>
```

### Step 4: Create Stylesheet
```css
/* css/style.css */
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

header {
    text-align: center;
    margin-bottom: 30px;
}

h1 {
    color: #667eea;
    font-size: 32px;
    margin-bottom: 10px;
}

.main-content {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 20px;
    margin-bottom: 20px;
}

.video-container {
    position: relative;
    background: #000;
    border-radius: 10px;
    overflow: hidden;
    aspect-ratio: 4/3;
}

#webcam {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

#canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.controls {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.control-group {
    background: #f5f5f5;
    padding: 15px;
    border-radius: 10px;
}

.control-group label {
    display: block;
    margin-bottom: 10px;
    font-weight: 600;
    color: #333;
}

.btn {
    width: 100%;
    padding: 12px 20px;
    border: none;
    border-radius: 8px;
    font-size: 16px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s;
    margin-bottom: 10px;
}

.btn-primary {
    background: #4CAF50;
    color: white;
}

.btn-primary:hover {
    background: #45a049;
}

.btn-danger {
    background: #f44336;
    color: white;
}

.btn-secondary {
    background: #2196F3;
    color: white;
}

.btn:disabled {
    background: #ccc;
    cursor: not-allowed;
}

input[type="range"] {
    width: 100%;
    margin-top: 10px;
}

.stats {
    background: #e3f2fd;
    padding: 15px;
    border-radius: 10px;
}

.stat-item {
    display: flex;
    justify-content: space-between;
    margin: 8px 0;
}

.stat-label {
    font-weight: 600;
    color: #666;
}

.stat-value {
    font-weight: 700;
    color: #2196F3;
}

.status {
    text-align: center;
    padding: 15px;
    background: #fff3cd;
    border-radius: 10px;
    margin: 20px 0;
    font-weight: 600;
}

.detected-objects {
    background: #f5f5f5;
    padding: 20px;
    border-radius: 10px;
    margin-top: 20px;
}

#objectsList {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 10px;
    margin-top: 15px;
}

.object-tag {
    background: #667eea;
    color: white;
    padding: 8px 12px;
    border-radius: 20px;
    font-size: 14px;
    text-align: center;
    font-weight: 600;
}

@media (max-width: 768px) {
    .main-content {
        grid-template-columns: 1fr;
    }
}
```

### Step 5: Implement Detection Logic
```javascript
// js/app.js
let detector = null;
let webcamStream = null;
let isDetecting = false;
let animationFrameId = null;

const webcamElement = document.getElementById('webcam');
const canvasElement = document.getElementById('canvas');
const ctx = canvasElement.getContext('2d');

const startBtn = document.getElementById('startBtn');
const stopBtn = document.getElementById('stopBtn');
const screenshotBtn = document.getElementById('screenshotBtn');
const confidenceSlider = document.getElementById('confidenceSlider');
const confidenceValue = document.getElementById('confidenceValue');
const statusDiv = document.getElementById('status');

let confidenceThreshold = 0.5;
let showLabels = true;
let showConfidence = true;

// Performance tracking
let frameCount = 0;
let lastTime = performance.now();
let fps = 0;

// Initialize
async function init() {
    statusDiv.textContent = 'Loading model...';

    try {
        detector = new ObjectDetector();
        await detector.loadModel();

        statusDiv.textContent = 'Model loaded! Click "Start Webcam" to begin.';
        statusDiv.style.background = '#d4edda';
        startBtn.disabled = false;

    } catch (error) {
        console.error('Initialization error:', error);
        statusDiv.textContent = 'Error loading model: ' + error.message;
        statusDiv.style.background = '#f8d7da';
    }
}

// Start webcam
async function startWebcam() {
    try {
        webcamStream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment', width: 640, height: 480 },
            audio: false
        });

        webcamElement.srcObject = webcamStream;

        webcamElement.onloadedmetadata = () => {
            canvasElement.width = webcamElement.videoWidth;
            canvasElement.height = webcamElement.videoHeight;

            startBtn.disabled = true;
            stopBtn.disabled = false;
            screenshotBtn.disabled = false;

            isDetecting = true;
            detectFrame();
        };

    } catch (error) {
        console.error('Webcam error:', error);
        alert('Could not access webcam: ' + error.message);
    }
}

// Stop webcam
function stopWebcam() {
    isDetecting = false;

    if (animationFrameId) {
        cancelAnimationFrame(animationFrameId);
    }

    if (webcamStream) {
        webcamStream.getTracks().forEach(track => track.stop());
        webcamStream = null;
    }

    ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);

    startBtn.disabled = false;
    stopBtn.disabled = true;
    screenshotBtn.disabled = true;
}

// Detect objects in frame
async function detectFrame() {
    if (!isDetecting) return;

    const startTime = performance.now();

    try {
        // Run detection
        const predictions = await detector.detect(webcamElement, confidenceThreshold);

        // Clear canvas
        ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);

        // Draw detections
        drawDetections(predictions);

        // Update UI
        updateStats(predictions, performance.now() - startTime);

        // Calculate FPS
        frameCount++;
        const currentTime = performance.now();
        if (currentTime - lastTime >= 1000) {
            fps = frameCount;
            frameCount = 0;
            lastTime = currentTime;
            document.getElementById('fps').textContent = fps;
        }

    } catch (error) {
        console.error('Detection error:', error);
    }

    // Continue detection loop
    animationFrameId = requestAnimationFrame(detectFrame);
}

// Draw bounding boxes and labels
function drawDetections(predictions) {
    predictions.forEach(prediction => {
        const [x, y, width, height] = prediction.bbox;

        // Draw bounding box
        ctx.strokeStyle = '#00FF00';
        ctx.lineWidth = 3;
        ctx.strokeRect(x, y, width, height);

        // Draw label background
        if (showLabels || showConfidence) {
            const label = showLabels ? prediction.class : '';
            const score = showConfidence ? `${(prediction.score * 100).toFixed(0)}%` : '';
            const text = `${label} ${score}`.trim();

            ctx.font = 'bold 16px Arial';
            const textWidth = ctx.measureText(text).width;

            ctx.fillStyle = '#00FF00';
            ctx.fillRect(x, y - 25, textWidth + 10, 25);

            // Draw label text
            ctx.fillStyle = '#000000';
            ctx.fillText(text, x + 5, y - 7);
        }
    });
}

// Update statistics
function updateStats(predictions, inferenceTime) {
    document.getElementById('detectionCount').textContent = predictions.length;
    document.getElementById('inferenceTime').textContent = `${inferenceTime.toFixed(0)}ms`;

    // Update detected objects list
    const objectsList = document.getElementById('objectsList');
    const uniqueObjects = [...new Set(predictions.map(p => p.class))];

    objectsList.innerHTML = uniqueObjects
        .map(obj => `<div class="object-tag">${obj}</div>`)
        .join('');
}

// Take screenshot
function takeScreenshot() {
    const screenshotCanvas = document.createElement('canvas');
    screenshotCanvas.width = canvasElement.width;
    screenshotCanvas.height = canvasElement.height;
    const screenshotCtx = screenshotCanvas.getContext('2d');

    // Draw video frame
    screenshotCtx.drawImage(webcamElement, 0, 0);

    // Draw detections on top
    screenshotCtx.drawImage(canvasElement, 0, 0);

    // Download image
    screenshotCanvas.toBlob(blob => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `detection-${Date.now()}.png`;
        a.click();
        URL.revokeObjectURL(url);
    });
}

// Event listeners
startBtn.addEventListener('click', startWebcam);
stopBtn.addEventListener('click', stopWebcam);
screenshotBtn.addEventListener('click', takeScreenshot);

confidenceSlider.addEventListener('input', (e) => {
    confidenceThreshold = e.target.value / 100;
    confidenceValue.textContent = confidenceThreshold.toFixed(2);
});

document.getElementById('showLabels').addEventListener('change', (e) => {
    showLabels = e.target.checked;
});

document.getElementById('showConfidence').addEventListener('change', (e) => {
    showConfidence = e.target.checked;
});

// Initialize on load
init();
```

## Expected Outputs

1. **Model Performance**:
   - Load time: <10 seconds
   - Inference time: 50-200ms per frame
   - FPS: 15-30 (depending on device)
   - Detection accuracy: >60% mAP on COCO

2. **Detection Capabilities**:
   - Detect 80 COCO object classes
   - Multiple objects in single frame
   - Accurate bounding boxes
   - Confidence scores 0-100%

3. **User Interface**:
   - Real-time video with overlay
   - Smooth bounding box rendering
   - Adjustable confidence threshold
   - FPS and performance metrics
   - Screenshot functionality

4. **Browser Compatibility**:
   - Works in Chrome, Firefox, Safari
   - Supports desktop and mobile
   - Handles different video resolutions

## Bonus Challenges

- [ ] Add object tracking across frames (assign IDs)
- [ ] Implement custom object detection model training
- [ ] Add video file upload and processing
- [ ] Create heatmap of detected object locations
- [ ] Implement zone-based alerting (detect objects in specific areas)
- [ ] Add export functionality for detection data (JSON/CSV)
- [ ] Create multi-camera support
- [ ] Implement background blur for detected persons
- [ ] Add augmented reality effects on detected objects
- [ ] Create time-lapse recording of detections
- [ ] Implement Web Workers for parallel processing
- [ ] Add model selection (different YOLO versions)

## Resources

- [COCO-SSD Model](https://github.com/tensorflow/tfjs-models/tree/master/coco-ssd)
- [TensorFlow.js Object Detection](https://www.tensorflow.org/js/tutorials/transfer/object_detection)
- [COCO Dataset](https://cocodataset.org/)
- [YOLOv5 Documentation](https://github.com/ultralytics/yolov5)
- [WebRTC getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)
- [Canvas API](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)
- [Non-Maximum Suppression](https://learnopencv.com/non-maximum-suppression-theory-and-implementation-in-pytorch/)

## Success Criteria

- Model loads successfully in browser
- Webcam access works on supported devices
- Objects are detected with >50% confidence
- Bounding boxes accurately surround objects
- Real-time performance (>10 FPS minimum)
- UI is responsive and intuitive
- Confidence threshold adjustment works
- Screenshot captures current detections
- Performance metrics display correctly
- Works on both desktop and mobile browsers
- No memory leaks during extended use
- Detection accuracy matches expectations for chosen model
