# Project 10: Build Real-Time Webcam Inference with TensorFlow.js

## Overview
Create a high-performance real-time computer vision application that processes webcam video streams in the browser using TensorFlow.js. This project combines multiple ML models (classification, detection, segmentation) to run simultaneously on live video, demonstrating advanced optimization techniques for achieving 30+ FPS inference in production environments.

## Difficulty Level
Advanced

## Learning Objectives
- Implement multi-model real-time inference pipeline
- Optimize WebGL backend for maximum performance
- Handle video preprocessing efficiently
- Implement frame skipping and adaptive FPS
- Manage GPU memory and tensor lifecycle
- Create smooth UI updates with requestAnimationFrame
- Build production-ready computer vision applications
- Implement model switching and fallback strategies

## Technical Stack
- **Models**: Multiple TFJS models (MobileNet, BodyPix, Face Detection)
- **Frontend**: HTML5, JavaScript ES6+, WebRTC
- **ML Framework**: TensorFlow.js with WebGL backend
- **Video**: getUserMedia, Canvas API
- **Performance**: Web Workers, OffscreenCanvas
- **UI**: Real-time overlays, performance monitoring
- **Optimization**: Tensor pooling, memory management

## Project Requirements

### 1. Multi-Model Architecture
- Load and manage multiple ML models
- Switch between models dynamically
- Run models in sequence or parallel
- Implement model warm-up and caching

### 2. Video Processing Pipeline
- Access webcam with optimal settings
- Implement efficient frame extraction
- Preprocess video frames for models
- Handle different input resolutions
- Manage frame buffer efficiently

### 3. Real-Time Inference
- Achieve 30+ FPS on modern hardware
- Implement adaptive frame rate
- Handle inference queue
- Optimize memory usage
- Minimize latency

### 4. Advanced Features
- Multiple visualization modes
- Performance profiling dashboard
- Recording and screenshot capability
- Filters and effects based on predictions
- Background replacement/blur

## Step-by-Step Implementation

### Step 1: Setup Project Structure
```bash
mkdir realtime-webcam-ml
cd realtime-webcam-ml

# Create project structure
mkdir css js models
touch index.html js/app.js js/model-manager.js js/video-processor.js css/style.css
```

### Step 2: Create Model Manager
```javascript
// js/model-manager.js
class ModelManager {
    constructor() {
        this.models = {
            mobilenet: null,
            bodyPix: null,
            faceMesh: null,
            handPose: null
        };

        this.activeModels = new Set();
        this.isReady = false;
    }

    async loadModel(modelName) {
        console.log(`Loading ${modelName}...`);

        try {
            switch(modelName) {
                case 'mobilenet':
                    this.models.mobilenet = await mobilenet.load({
                        version: 2,
                        alpha: 0.5  // Faster, smaller model
                    });
                    break;

                case 'bodyPix':
                    this.models.bodyPix = await bodyPix.load({
                        architecture: 'MobileNetV1',
                        outputStride: 16,
                        multiplier: 0.75,
                        quantBytes: 2
                    });
                    break;

                case 'faceMesh':
                    this.models.faceMesh = await facemesh.load({
                        maxFaces: 1
                    });
                    break;

                case 'handPose':
                    this.models.handPose = await handpose.load();
                    break;

                default:
                    throw new Error(`Unknown model: ${modelName}`);
            }

            // Warm up model
            await this.warmUpModel(modelName);

            this.activeModels.add(modelName);
            console.log(`${modelName} loaded and ready`);

            return true;

        } catch (error) {
            console.error(`Error loading ${modelName}:`, error);
            return false;
        }
    }

    async warmUpModel(modelName) {
        const model = this.models[modelName];
        if (!model) return;

        console.log(`Warming up ${modelName}...`);

        // Create dummy input
        const dummyInput = tf.zeros([1, 224, 224, 3]);

        try {
            switch(modelName) {
                case 'mobilenet':
                    await model.classify(dummyInput);
                    break;

                case 'bodyPix':
                    await model.segmentPerson(dummyInput);
                    break;

                case 'faceMesh':
                    await model.estimateFaces(dummyInput);
                    break;

                case 'handPose':
                    await model.estimateHands(dummyInput);
                    break;
            }
        } catch (error) {
            console.error(`Warmup failed for ${modelName}:`, error);
        } finally {
            dummyInput.dispose();
        }
    }

    async loadAllModels() {
        const modelNames = ['mobilenet', 'bodyPix'];

        for (const name of modelNames) {
            await this.loadModel(name);
        }

        this.isReady = true;
    }

    getModel(modelName) {
        return this.models[modelName];
    }

    isModelLoaded(modelName) {
        return this.activeModels.has(modelName);
    }
}
```

### Step 3: Create Video Processor
```javascript
// js/video-processor.js
class VideoProcessor {
    constructor() {
        this.video = null;
        this.canvas = null;
        this.ctx = null;
        this.stream = null;
        this.isProcessing = false;
        this.frameCount = 0;
        this.lastFrameTime = 0;
        this.fps = 0;
        this.targetFPS = 30;
        this.skipFrames = 0;
        this.frameSkipCounter = 0;
    }

    async initialize(videoElement, canvasElement) {
        this.video = videoElement;
        this.canvas = canvasElement;
        this.ctx = canvasElement.getContext('2d', {
            alpha: false,
            desynchronized: true
        });

        try {
            // Request webcam with optimal settings
            this.stream = await navigator.mediaDevices.getUserMedia({
                video: {
                    facingMode: 'user',
                    width: { ideal: 640 },
                    height: { ideal: 480 },
                    frameRate: { ideal: 30 }
                },
                audio: false
            });

            this.video.srcObject = this.stream;

            return new Promise((resolve) => {
                this.video.onloadedmetadata = () => {
                    this.video.play();
                    this.canvas.width = this.video.videoWidth;
                    this.canvas.height = this.video.videoHeight;
                    console.log(`Video initialized: ${this.canvas.width}x${this.canvas.height}`);
                    resolve();
                };
            });

        } catch (error) {
            console.error('Error accessing webcam:', error);
            throw error;
        }
    }

    stop() {
        this.isProcessing = false;

        if (this.stream) {
            this.stream.getTracks().forEach(track => track.stop());
            this.stream = null;
        }
    }

    startProcessing(callback) {
        this.isProcessing = true;
        this.processFrame(callback);
    }

    async processFrame(callback) {
        if (!this.isProcessing) return;

        const startTime = performance.now();

        // Frame skipping logic
        this.frameSkipCounter++;
        if (this.frameSkipCounter <= this.skipFrames) {
            requestAnimationFrame(() => this.processFrame(callback));
            return;
        }
        this.frameSkipCounter = 0;

        // Draw current frame to canvas
        this.ctx.drawImage(this.video, 0, 0, this.canvas.width, this.canvas.height);

        try {
            // Execute callback with video element
            await callback(this.video, this.canvas, this.ctx);

        } catch (error) {
            console.error('Frame processing error:', error);
        }

        // Calculate FPS
        this.frameCount++;
        const currentTime = performance.now();

        if (currentTime - this.lastFrameTime >= 1000) {
            this.fps = this.frameCount;
            this.frameCount = 0;
            this.lastFrameTime = currentTime;

            // Adaptive frame skipping
            if (this.fps < this.targetFPS - 5) {
                this.skipFrames = Math.min(this.skipFrames + 1, 3);
            } else if (this.fps > this.targetFPS && this.skipFrames > 0) {
                this.skipFrames--;
            }
        }

        // Continue processing
        requestAnimationFrame(() => this.processFrame(callback));
    }

    getCurrentFrame() {
        return this.ctx.getImageData(0, 0, this.canvas.width, this.canvas.height);
    }

    getFPS() {
        return this.fps;
    }
}
```

### Step 4: Create Main HTML Interface
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Real-Time Webcam ML - TensorFlow.js</title>

    <!-- TensorFlow.js -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>

    <!-- Pre-trained Models -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/mobilenet@2.1.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/body-pix@2.2.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/facemesh@0.0.5"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/handpose@0.0.7"></script>

    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>🎥 Real-Time Webcam ML</h1>
            <p>Multi-model computer vision in your browser</p>
        </header>

        <!-- Performance Dashboard -->
        <div class="performance-dashboard">
            <div class="metric">
                <span class="metric-label">FPS:</span>
                <span id="fps" class="metric-value">0</span>
            </div>
            <div class="metric">
                <span class="metric-label">Inference:</span>
                <span id="inferenceTime" class="metric-value">0ms</span>
            </div>
            <div class="metric">
                <span class="metric-label">Memory:</span>
                <span id="memory" class="metric-value">0 MB</span>
            </div>
            <div class="metric">
                <span class="metric-label">Tensors:</span>
                <span id="tensors" class="metric-value">0</span>
            </div>
            <div class="metric">
                <span class="metric-label">Backend:</span>
                <span id="backend" class="metric-value">-</span>
            </div>
        </div>

        <!-- Main Content -->
        <div class="main-content">
            <div class="video-section">
                <div class="video-container">
                    <video id="webcam" autoplay playsinline></video>
                    <canvas id="canvas"></canvas>
                    <div id="overlay" class="overlay"></div>
                </div>

                <div class="controls">
                    <button id="startBtn" class="btn btn-primary" onclick="start()">
                        📹 Start Camera
                    </button>
                    <button id="stopBtn" class="btn btn-danger" onclick="stop()" disabled>
                        ⏹️ Stop
                    </button>
                    <button id="screenshotBtn" class="btn btn-secondary" onclick="takeScreenshot()" disabled>
                        📸 Screenshot
                    </button>
                </div>
            </div>

            <div class="sidebar">
                <!-- Model Selection -->
                <div class="panel">
                    <h3>Active Models</h3>
                    <div class="model-list">
                        <label class="model-option">
                            <input type="checkbox" id="modelMobileNet" checked>
                            <span>Image Classification</span>
                        </label>
                        <label class="model-option">
                            <input type="checkbox" id="modelBodyPix">
                            <span>Body Segmentation</span>
                        </label>
                        <label class="model-option">
                            <input type="checkbox" id="modelFaceMesh">
                            <span>Face Mesh</span>
                        </label>
                        <label class="model-option">
                            <input type="checkbox" id="modelHandPose">
                            <span>Hand Pose</span>
                        </label>
                    </div>
                </div>

                <!-- Visualization Options -->
                <div class="panel">
                    <h3>Visualization</h3>
                    <div class="viz-options">
                        <label>
                            <input type="checkbox" id="showLabels" checked>
                            Show Labels
                        </label>
                        <label>
                            <input type="checkbox" id="showConfidence" checked>
                            Show Confidence
                        </label>
                        <label>
                            <input type="checkbox" id="blurBackground">
                            Blur Background
                        </label>
                    </div>
                </div>

                <!-- Performance Settings -->
                <div class="panel">
                    <h3>Performance</h3>
                    <div class="settings">
                        <label>
                            Target FPS:
                            <input type="range" id="targetFPS" min="10" max="60" value="30">
                            <span id="targetFPSValue">30</span>
                        </label>
                    </div>
                </div>

                <!-- Predictions Panel -->
                <div class="panel">
                    <h3>Predictions</h3>
                    <div id="predictions" class="predictions-list"></div>
                </div>

                <!-- Status -->
                <div class="panel">
                    <h3>Status</h3>
                    <div id="status" class="status-text">Loading models...</div>
                </div>
            </div>
        </div>
    </div>

    <script src="js/model-manager.js"></script>
    <script src="js/video-processor.js"></script>
    <script src="js/app.js"></script>
</body>
</html>
```

### Step 5: Create Stylesheet
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
    max-width: 1600px;
    margin: 0 auto;
    background: white;
    border-radius: 20px;
    padding: 30px;
    box-shadow: 0 10px 50px rgba(0,0,0,0.3);
}

header {
    text-align: center;
    margin-bottom: 20px;
}

h1 {
    color: #667eea;
    margin-bottom: 5px;
}

header p {
    color: #666;
}

.performance-dashboard {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    gap: 15px;
    margin-bottom: 20px;
}

.metric {
    background: #f5f5f5;
    padding: 15px;
    border-radius: 10px;
    text-align: center;
}

.metric-label {
    display: block;
    font-size: 12px;
    color: #666;
    margin-bottom: 5px;
}

.metric-value {
    display: block;
    font-size: 24px;
    font-weight: 700;
    color: #667eea;
}

.main-content {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 20px;
}

.video-section {
    display: flex;
    flex-direction: column;
    gap: 15px;
}

.video-container {
    position: relative;
    background: #000;
    border-radius: 15px;
    overflow: hidden;
    aspect-ratio: 4/3;
}

#webcam {
    position: absolute;
    top: 0;
    left: 0;
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

.overlay {
    position: absolute;
    top: 10px;
    left: 10px;
    color: white;
    background: rgba(0,0,0,0.7);
    padding: 10px;
    border-radius: 5px;
    font-size: 14px;
}

.controls {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
}

.btn {
    padding: 12px 20px;
    border: none;
    border-radius: 10px;
    font-size: 16px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s;
}

.btn-primary {
    background: #4CAF50;
    color: white;
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

.sidebar {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.panel {
    background: #f5f5f5;
    padding: 20px;
    border-radius: 10px;
}

.panel h3 {
    color: #333;
    margin-bottom: 15px;
    font-size: 16px;
}

.model-list,
.viz-options {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.model-option {
    display: flex;
    align-items: center;
    gap: 10px;
    cursor: pointer;
    padding: 8px;
    background: white;
    border-radius: 5px;
    transition: background 0.3s;
}

.model-option:hover {
    background: #e8e8e8;
}

.model-option input[type="checkbox"] {
    width: 18px;
    height: 18px;
}

.settings label {
    display: block;
    margin: 10px 0;
}

.settings input[type="range"] {
    width: 100%;
    margin-top: 5px;
}

.predictions-list {
    max-height: 200px;
    overflow-y: auto;
}

.prediction-item {
    background: white;
    padding: 8px 12px;
    margin: 5px 0;
    border-radius: 5px;
    display: flex;
    justify-content: space-between;
}

.status-text {
    background: white;
    padding: 15px;
    border-radius: 5px;
    text-align: center;
}

@media (max-width: 1024px) {
    .main-content {
        grid-template-columns: 1fr;
    }
}
```

### Step 6: Implement Main Application Logic
```javascript
// js/app.js
let modelManager = null;
let videoProcessor = null;
let isRunning = false;

// Active models
let activeInference = {
    classification: true,
    segmentation: false,
    faceMesh: false,
    handPose: false
};

// Settings
let showLabels = true;
let showConfidence = true;
let blurBackground = false;

// Initialize application
async function init() {
    updateStatus('Initializing TensorFlow.js...');

    // Set backend
    await tf.setBackend('webgl');
    await tf.ready();

    const backend = tf.getBackend();
    document.getElementById('backend').textContent = backend.toUpperCase();

    // Initialize managers
    modelManager = new ModelManager();
    videoProcessor = new VideoProcessor();

    // Load default models
    updateStatus('Loading models...');
    await modelManager.loadAllModels();

    updateStatus('Ready! Click "Start Camera" to begin.');

    document.getElementById('startBtn').disabled = false;

    // Setup event listeners
    setupEventListeners();

    // Start memory monitoring
    monitorMemory();
}

// Start video processing
async function start() {
    try {
        updateStatus('Starting camera...');

        await videoProcessor.initialize(
            document.getElementById('webcam'),
            document.getElementById('canvas')
        );

        isRunning = true;

        document.getElementById('startBtn').disabled = true;
        document.getElementById('stopBtn').disabled = false;
        document.getElementById('screenshotBtn').disabled = false;

        videoProcessor.startProcessing(processVideoFrame);

        updateStatus('Running...');

    } catch (error) {
        console.error('Error starting:', error);
        updateStatus('Error: ' + error.message);
        alert('Could not access webcam: ' + error.message);
    }
}

// Stop video processing
function stop() {
    isRunning = false;
    videoProcessor.stop();

    document.getElementById('startBtn').disabled = false;
    document.getElementById('stopBtn').disabled = true;
    document.getElementById('screenshotBtn').disabled = true;

    updateStatus('Stopped');
}

// Process each video frame
async function processVideoFrame(video, canvas, ctx) {
    const startTime = performance.now();

    try {
        // Run active models
        await tf.tidy(async () => {
            // Classification
            if (activeInference.classification && modelManager.isModelLoaded('mobilenet')) {
                await runClassification(video);
            }

            // Body Segmentation
            if (activeInference.segmentation && modelManager.isModelLoaded('bodyPix')) {
                await runSegmentation(video, canvas, ctx);
            }

            // Additional models can be added here
        });

    } catch (error) {
        console.error('Processing error:', error);
    }

    // Update performance metrics
    const inferenceTime = performance.now() - startTime;
    updatePerformanceMetrics(inferenceTime);
}

// Run image classification
async function runClassification(video) {
    const model = modelManager.getModel('mobilenet');
    const predictions = await model.classify(video, 3);

    if (showLabels) {
        displayPredictions(predictions);
    }
}

// Run body segmentation
async function runSegmentation(video, canvas, ctx) {
    const model = modelManager.getModel('bodyPix');

    const segmentation = await model.segmentPerson(video, {
        flipHorizontal: false,
        internalResolution: 'medium',
        segmentationThreshold: 0.7
    });

    if (blurBackground) {
        const backgroundBlurAmount = 6;
        const edgeBlurAmount = 2;

        await bodyPix.drawBokehEffect(
            canvas, video, segmentation,
            backgroundBlurAmount,
            edgeBlurAmount
        );
    }
}

// Display classification predictions
function displayPredictions(predictions) {
    const predictionsDiv = document.getElementById('predictions');
    predictionsDiv.innerHTML = '';

    predictions.forEach(pred => {
        const item = document.createElement('div');
        item.className = 'prediction-item';

        const label = pred.className;
        const confidence = showConfidence ?
            `${(pred.probability * 100).toFixed(1)}%` : '';

        item.innerHTML = `
            <span>${label}</span>
            <span>${confidence}</span>
        `;

        predictionsDiv.appendChild(item);
    });
}

// Update performance metrics
function updatePerformanceMetrics(inferenceTime) {
    document.getElementById('fps').textContent = videoProcessor.getFPS();
    document.getElementById('inferenceTime').textContent = `${inferenceTime.toFixed(0)}ms`;
}

// Monitor memory usage
function monitorMemory() {
    setInterval(() => {
        const memInfo = tf.memory();
        document.getElementById('memory').textContent =
            `${(memInfo.numBytes / 1024 / 1024).toFixed(1)} MB`;
        document.getElementById('tensors').textContent = memInfo.numTensors;
    }, 1000);
}

// Setup event listeners
function setupEventListeners() {
    // Model checkboxes
    document.getElementById('modelMobileNet').addEventListener('change', (e) => {
        activeInference.classification = e.target.checked;
    });

    document.getElementById('modelBodyPix').addEventListener('change', async (e) => {
        if (e.target.checked && !modelManager.isModelLoaded('bodyPix')) {
            await modelManager.loadModel('bodyPix');
        }
        activeInference.segmentation = e.target.checked;
    });

    // Visualization options
    document.getElementById('showLabels').addEventListener('change', (e) => {
        showLabels = e.target.checked;
    });

    document.getElementById('showConfidence').addEventListener('change', (e) => {
        showConfidence = e.target.checked;
    });

    document.getElementById('blurBackground').addEventListener('change', (e) => {
        blurBackground = e.target.checked;
    });

    // Target FPS slider
    document.getElementById('targetFPS').addEventListener('input', (e) => {
        const fps = e.target.value;
        document.getElementById('targetFPSValue').textContent = fps;
        videoProcessor.targetFPS = parseInt(fps);
    });
}

// Take screenshot
function takeScreenshot() {
    const canvas = document.getElementById('canvas');

    canvas.toBlob((blob) => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `screenshot-${Date.now()}.png`;
        a.click();
        URL.revokeObjectURL(url);
    });
}

// Update status
function updateStatus(message) {
    document.getElementById('status').textContent = message;
}

// Initialize app
init();
```

## Expected Outputs

1. **Real-Time Performance**:
   - 30+ FPS on modern hardware
   - <50ms inference per frame
   - Smooth video playback
   - No visible lag

2. **Multi-Model Support**:
   - Image classification working
   - Body segmentation functional
   - Models switchable on the fly
   - Efficient model management

3. **Features**:
   - Background blur effect
   - Real-time predictions
   - Performance dashboard
   - Screenshot capability

4. **Resource Usage**:
   - Memory stable (<500MB)
   - GPU utilization optimized
   - Tensor count managed
   - No memory leaks

## Bonus Challenges

- [ ] Add face filters/effects
- [ ] Implement virtual background replacement
- [ ] Create gesture-controlled UI
- [ ] Add AR objects overlay
- [ ] Implement style transfer
- [ ] Create video recording with effects
- [ ] Add multi-person tracking
- [ ] Implement action recognition
- [ ] Create custom ML pipeline
- [ ] Add WebWorker processing
- [ ] Implement model ensemble
- [ ] Create custom shaders for effects

## Resources

- [TensorFlow.js Performance Best Practices](https://www.tensorflow.org/js/guide/platform_environment)
- [WebGL Backend](https://github.com/tensorflow/tfjs/tree/master/tfjs-backend-webgl)
- [Pre-trained Models](https://github.com/tensorflow/tfjs-models)
- [BodyPix Documentation](https://github.com/tensorflow/tfjs-models/tree/master/body-pix)
- [MobileNet](https://github.com/tensorflow/tfjs-models/tree/master/mobilenet)

## Success Criteria

- Application achieves 30+ FPS
- Multiple models run simultaneously
- Webcam access works reliably
- Real-time predictions display correctly
- Background effects work smoothly
- Performance dashboard shows accurate metrics
- Memory usage remains stable
- No tensor leaks during operation
- Works across Chrome, Firefox, Safari
- Adaptive FPS maintains smoothness
- Screenshot functionality works
- UI remains responsive during inference
