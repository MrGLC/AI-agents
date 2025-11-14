# Project 04: Svelte App with TensorFlow.js Inference

## Overview
Build a lightweight, high-performance Svelte application that runs machine learning inference directly in the browser using TensorFlow.js. This project demonstrates client-side ML with no backend required, utilizing Svelte's reactive programming model and minimal bundle size for optimal performance.

## Learning Objectives
- Master Svelte's reactive declarations and stores
- Implement TensorFlow.js for browser-based ML
- Handle model loading and inference lifecycle
- Optimize WebGL and WASM backends for performance
- Build custom Svelte actions and transitions
- Manage memory with tensor disposal
- Implement progressive model loading
- Create real-time inference with webcam/file inputs

## Difficulty Level
**Intermediate to Advanced** - Requires understanding of Svelte, TensorFlow.js, and browser performance optimization.

## Technical Stack
- **Frontend**: Svelte 4+ with SvelteKit
- **ML Framework**: TensorFlow.js (WebGL/WASM backends)
- **State Management**: Svelte Stores
- **Build Tool**: Vite
- **UI Components**: Custom Svelte components
- **Model Format**: TensorFlow.js Graph Model or Layers Model
- **Testing**: Vitest, Playwright
- **Visualization**: D3.js with Svelte

## Requirements

### UI/UX Requirements
1. Lightweight, fast-loading interface
2. Model loading progress indicators
3. Multiple input methods (file upload, webcam, manual input)
4. Real-time inference visualization
5. Performance metrics display (FPS, inference time)
6. Model comparison interface
7. Smooth transitions and animations
8. Mobile-optimized responsive design

### TensorFlow.js Requirements
1. Support for multiple backend types (WebGL, WASM, CPU)
2. Model preloading and caching
3. Batch inference support
4. Tensor memory management
5. Model warmup for consistent performance
6. Backend fallback mechanism
7. Model quantization support

### Performance Requirements
1. Bundle size < 500KB (excluding ML models)
2. First inference < 1 second after load
3. Subsequent inference < 100ms
4. Smooth 60 FPS rendering
5. Efficient memory usage with cleanup
6. Service worker for model caching

### Supported Model Types
1. Image classification (MobileNet, ResNet)
2. Object detection (COCO-SSD)
3. Pose estimation (PoseNet, MoveNet)
4. Text sentiment analysis
5. Custom trained models

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create SvelteKit project
npm create svelte@latest ml-tfjs-inference
# Choose: Skeleton project, TypeScript, ESLint, Prettier

cd ml-tfjs-inference

# Install dependencies
npm install @tensorflow/tfjs @tensorflow/tfjs-vis
npm install @tensorflow-models/mobilenet @tensorflow-models/coco-ssd
npm install d3 @sveltejs/adapter-static
npm install -D @types/d3

npm install
```

### Step 2: TensorFlow.js Setup and Model Store

Create `src/lib/stores/tfjs.ts`:

```typescript
import { writable, derived, get } from 'svelte/store';
import * as tf from '@tensorflow/tfjs';
import type { GraphModel, LayersModel } from '@tensorflow/tfjs';

export interface ModelConfig {
  name: string;
  url: string;
  type: 'graph' | 'layers';
  inputShape?: number[];
  outputShape?: number[];
  preprocessor?: (input: any) => tf.Tensor;
  postprocessor?: (output: tf.Tensor) => any;
}

export interface LoadingState {
  isLoading: boolean;
  progress: number;
  stage: string;
}

export interface InferenceResult {
  output: any;
  inferenceTime: number;
  timestamp: number;
}

// Backend store
export const backend = writable<string>('webgl');
export const backendInitialized = writable<boolean>(false);

// Model store
export const loadedModels = writable<Map<string, GraphModel | LayersModel>>(
  new Map()
);
export const loadingState = writable<LoadingState>({
  isLoading: false,
  progress: 0,
  stage: 'idle',
});

// Inference store
export const inferenceResults = writable<InferenceResult[]>([]);
export const currentInference = writable<InferenceResult | null>(null);

// Performance metrics
export const performanceMetrics = writable({
  fps: 0,
  avgInferenceTime: 0,
  memoryUsage: 0,
  numTensors: 0,
});

// TensorFlow.js utilities
export const tfjsStore = {
  async initializeBackend(preferredBackend: string = 'webgl') {
    try {
      loadingState.set({
        isLoading: true,
        progress: 0,
        stage: 'Initializing TensorFlow.js',
      });

      await tf.ready();

      // Try to set preferred backend
      const success = await tf.setBackend(preferredBackend);

      if (!success) {
        console.warn(`Failed to set ${preferredBackend}, trying fallback`);
        await tf.setBackend('cpu');
      }

      const currentBackend = tf.getBackend();
      backend.set(currentBackend);
      backendInitialized.set(true);

      console.log(`TensorFlow.js backend: ${currentBackend}`);

      loadingState.set({
        isLoading: false,
        progress: 100,
        stage: 'Ready',
      });

      return currentBackend;
    } catch (error) {
      console.error('Failed to initialize TensorFlow.js:', error);
      loadingState.set({
        isLoading: false,
        progress: 0,
        stage: 'Error',
      });
      throw error;
    }
  },

  async loadModel(config: ModelConfig) {
    const { name, url, type } = config;

    loadingState.set({
      isLoading: true,
      progress: 0,
      stage: `Loading model: ${name}`,
    });

    try {
      let model: GraphModel | LayersModel;

      if (type === 'graph') {
        model = await tf.loadGraphModel(url, {
          onProgress: (fraction) => {
            loadingState.update((state) => ({
              ...state,
              progress: fraction * 100,
            }));
          },
        });
      } else {
        model = await tf.loadLayersModel(url, {
          onProgress: (fraction) => {
            loadingState.update((state) => ({
              ...state,
              progress: fraction * 100,
            }));
          },
        });
      }

      // Warmup inference
      if (config.inputShape) {
        const warmupTensor = tf.zeros(config.inputShape);
        const warmupOutput = model.predict(warmupTensor) as tf.Tensor;
        warmupTensor.dispose();
        warmupOutput.dispose();
      }

      loadedModels.update((models) => {
        models.set(name, model);
        return models;
      });

      loadingState.set({
        isLoading: false,
        progress: 100,
        stage: `Model ${name} loaded`,
      });

      return model;
    } catch (error) {
      console.error(`Failed to load model ${name}:`, error);
      loadingState.set({
        isLoading: false,
        progress: 0,
        stage: 'Error loading model',
      });
      throw error;
    }
  },

  async predict(modelName: string, input: tf.Tensor | tf.Tensor[]) {
    const models = get(loadedModels);
    const model = models.get(modelName);

    if (!model) {
      throw new Error(`Model ${modelName} not loaded`);
    }

    const startTime = performance.now();

    const output = model.predict(input) as tf.Tensor;
    const result = await output.data();

    const inferenceTime = performance.now() - startTime;

    const inferenceResult: InferenceResult = {
      output: Array.from(result),
      inferenceTime,
      timestamp: Date.now(),
    };

    currentInference.set(inferenceResult);
    inferenceResults.update((results) => [inferenceResult, ...results].slice(0, 100));

    // Update performance metrics
    tfjsStore.updatePerformanceMetrics(inferenceTime);

    return { output, inferenceResult };
  },

  updatePerformanceMetrics(inferenceTime: number) {
    performanceMetrics.update((metrics) => {
      const results = get(inferenceResults);
      const avgTime =
        results.reduce((sum, r) => sum + r.inferenceTime, 0) / results.length;

      return {
        ...metrics,
        avgInferenceTime: avgTime,
        numTensors: tf.memory().numTensors,
        memoryUsage: tf.memory().numBytes,
      };
    });
  },

  dispose() {
    const models = get(loadedModels);
    models.forEach((model) => model.dispose());
    loadedModels.set(new Map());
  },

  getMemoryInfo() {
    return tf.memory();
  },

  profile(fn: () => void) {
    return tf.profile(fn);
  },
};
```

### Step 3: Image Classification Component

Create `src/lib/components/ImageClassifier.svelte`:

```svelte
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as tf from '@tensorflow/tfjs';
  import * as mobilenet from '@tensorflow-models/mobilenet';
  import { fade, fly } from 'svelte/transition';
  import { tfjsStore, loadingState, currentInference } from '../stores/tfjs';

  let model: mobilenet.MobileNet | null = null;
  let imageElement: HTMLImageElement;
  let fileInput: HTMLInputElement;
  let selectedImage: string | null = null;
  let predictions: Array<{ className: string; probability: number }> = [];
  let isProcessing = false;
  let inferenceTime = 0;

  onMount(async () => {
    await tfjsStore.initializeBackend('webgl');
    await loadModel();
  });

  onDestroy(() => {
    if (model) {
      model.dispose();
    }
  });

  async function loadModel() {
    try {
      model = await mobilenet.load({
        version: 2,
        alpha: 1.0,
      });
      console.log('MobileNet model loaded');
    } catch (error) {
      console.error('Failed to load model:', error);
    }
  }

  function handleFileSelect(event: Event) {
    const target = event.target as HTMLInputElement;
    const file = target.files?.[0];

    if (file && file.type.startsWith('image/')) {
      const reader = new FileReader();
      reader.onload = (e) => {
        selectedImage = e.target?.result as string;
      };
      reader.readAsDataURL(file);
    }
  }

  async function classifyImage() {
    if (!model || !imageElement) return;

    isProcessing = true;
    predictions = [];

    try {
      const startTime = performance.now();

      // Classify image
      const results = await model.classify(imageElement, 5);
      inferenceTime = performance.now() - startTime;

      predictions = results.map((r) => ({
        className: r.className,
        probability: r.probability,
      }));

      tfjsStore.updatePerformanceMetrics(inferenceTime);
    } catch (error) {
      console.error('Classification error:', error);
    } finally {
      isProcessing = false;
    }
  }

  function clearImage() {
    selectedImage = null;
    predictions = [];
    inferenceTime = 0;
    if (fileInput) {
      fileInput.value = '';
    }
  }

  // Reactive statement - classify when image loads
  $: if (imageElement && selectedImage && model) {
    imageElement.onload = () => classifyImage();
  }
</script>

<div class="classifier-container">
  <div class="upload-section">
    <h2>Image Classification</h2>
    <p class="subtitle">Upload an image to classify using MobileNet v2</p>

    <div class="file-input-wrapper">
      <input
        type="file"
        accept="image/*"
        on:change={handleFileSelect}
        bind:this={fileInput}
        id="file-input"
      />
      <label for="file-input" class="file-label">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
          <polyline points="17 8 12 3 7 8" />
          <line x1="12" y1="3" x2="12" y2="15" />
        </svg>
        Choose Image
      </label>
    </div>

    {#if $loadingState.isLoading}
      <div class="loading" transition:fade>
        <div class="spinner"></div>
        <p>{$loadingState.stage}</p>
        <div class="progress-bar">
          <div
            class="progress-fill"
            style="width: {$loadingState.progress}%"
          ></div>
        </div>
      </div>
    {/if}
  </div>

  {#if selectedImage}
    <div class="preview-section" transition:fly={{ y: 20, duration: 300 }}>
      <div class="image-container">
        <img
          bind:this={imageElement}
          src={selectedImage}
          alt="Selected"
          class="preview-image"
        />
        {#if isProcessing}
          <div class="processing-overlay">
            <div class="spinner"></div>
            <p>Classifying...</p>
          </div>
        {/if}
      </div>

      <div class="controls">
        <button on:click={classifyImage} disabled={isProcessing} class="btn-primary">
          {isProcessing ? 'Processing...' : 'Classify Again'}
        </button>
        <button on:click={clearImage} class="btn-secondary">Clear</button>
      </div>

      {#if inferenceTime > 0}
        <div class="inference-info" transition:fade>
          <span>Inference time: {inferenceTime.toFixed(2)}ms</span>
        </div>
      {/if}
    </div>
  {/if}

  {#if predictions.length > 0}
    <div class="results-section" transition:fly={{ y: 20, duration: 300 }}>
      <h3>Predictions</h3>
      <div class="predictions-list">
        {#each predictions as prediction, index}
          <div
            class="prediction-item"
            transition:fly={{ x: -20, duration: 300, delay: index * 50 }}
          >
            <div class="prediction-header">
              <span class="rank">#{index + 1}</span>
              <span class="class-name">{prediction.className}</span>
              <span class="probability">
                {(prediction.probability * 100).toFixed(2)}%
              </span>
            </div>
            <div class="confidence-bar">
              <div
                class="confidence-fill"
                style="width: {prediction.probability * 100}%"
              ></div>
            </div>
          </div>
        {/each}
      </div>
    </div>
  {/if}
</div>

<style>
  .classifier-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
  }

  .upload-section {
    text-align: center;
    margin-bottom: 30px;
  }

  h2 {
    font-size: 28px;
    margin-bottom: 10px;
    color: #2d3748;
  }

  .subtitle {
    color: #718096;
    margin-bottom: 20px;
  }

  .file-input-wrapper {
    margin: 20px 0;
  }

  #file-input {
    display: none;
  }

  .file-label {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    padding: 12px 24px;
    background: #4299e1;
    color: white;
    border-radius: 8px;
    cursor: pointer;
    transition: background 0.2s;
    font-weight: 500;
  }

  .file-label:hover {
    background: #3182ce;
  }

  .loading {
    margin-top: 20px;
  }

  .spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #e2e8f0;
    border-top-color: #4299e1;
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin: 0 auto 10px;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .progress-bar {
    width: 100%;
    height: 4px;
    background: #e2e8f0;
    border-radius: 2px;
    overflow: hidden;
    margin-top: 10px;
  }

  .progress-fill {
    height: 100%;
    background: #4299e1;
    transition: width 0.3s ease;
  }

  .preview-section {
    margin-bottom: 30px;
  }

  .image-container {
    position: relative;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    margin-bottom: 20px;
  }

  .preview-image {
    width: 100%;
    height: auto;
    display: block;
  }

  .processing-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.7);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    color: white;
  }

  .controls {
    display: flex;
    gap: 10px;
    justify-content: center;
    margin-bottom: 10px;
  }

  .btn-primary,
  .btn-secondary {
    padding: 10px 20px;
    border-radius: 6px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s;
    border: none;
  }

  .btn-primary {
    background: #4299e1;
    color: white;
  }

  .btn-primary:hover:not(:disabled) {
    background: #3182ce;
  }

  .btn-primary:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .btn-secondary {
    background: #e2e8f0;
    color: #2d3748;
  }

  .btn-secondary:hover {
    background: #cbd5e0;
  }

  .inference-info {
    text-align: center;
    color: #718096;
    font-size: 14px;
  }

  .results-section {
    background: white;
    border-radius: 12px;
    padding: 24px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .results-section h3 {
    margin-bottom: 20px;
    color: #2d3748;
  }

  .predictions-list {
    display: flex;
    flex-direction: column;
    gap: 15px;
  }

  .prediction-item {
    background: #f7fafc;
    padding: 15px;
    border-radius: 8px;
  }

  .prediction-header {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 10px;
  }

  .rank {
    background: #4299e1;
    color: white;
    width: 28px;
    height: 28px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    font-weight: bold;
  }

  .class-name {
    flex: 1;
    font-weight: 500;
    color: #2d3748;
  }

  .probability {
    font-weight: 600;
    color: #4299e1;
  }

  .confidence-bar {
    height: 8px;
    background: #e2e8f0;
    border-radius: 4px;
    overflow: hidden;
  }

  .confidence-fill {
    height: 100%;
    background: linear-gradient(90deg, #4299e1, #3182ce);
    transition: width 0.5s ease;
  }
</style>
```

### Step 4: Object Detection Component

Create `src/lib/components/ObjectDetector.svelte`:

```svelte
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as cocoSsd from '@tensorflow-models/coco-ssd';
  import { tfjsStore } from '../stores/tfjs';

  let model: cocoSsd.ObjectDetection | null = null;
  let videoElement: HTMLVideoElement;
  let canvasElement: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null;
  let isDetecting = false;
  let fps = 0;
  let detectionCount = 0;
  let stream: MediaStream | null = null;

  onMount(async () => {
    await tfjsStore.initializeBackend('webgl');
    await loadModel();
  });

  onDestroy(() => {
    stopDetection();
    if (model) {
      model.dispose();
    }
  });

  async function loadModel() {
    model = await cocoSsd.load();
    console.log('COCO-SSD model loaded');
  }

  async function startWebcam() {
    try {
      stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 640, height: 480 },
      });

      videoElement.srcObject = stream;
      videoElement.onloadedmetadata = () => {
        videoElement.play();
        canvasElement.width = videoElement.videoWidth;
        canvasElement.height = videoElement.videoHeight;
        ctx = canvasElement.getContext('2d');
        startDetection();
      };
    } catch (error) {
      console.error('Error accessing webcam:', error);
    }
  }

  async function startDetection() {
    if (!model || !ctx) return;

    isDetecting = true;
    let lastTime = performance.now();
    let frameCount = 0;

    async function detect() {
      if (!isDetecting || !model || !ctx) return;

      // Draw video frame
      ctx.drawImage(videoElement, 0, 0);

      // Detect objects
      const predictions = await model.detect(canvasElement);
      detectionCount = predictions.length;

      // Draw predictions
      predictions.forEach((prediction) => {
        const [x, y, width, height] = prediction.bbox;

        // Draw bounding box
        ctx.strokeStyle = '#00ff00';
        ctx.lineWidth = 3;
        ctx.strokeRect(x, y, width, height);

        // Draw label background
        ctx.fillStyle = '#00ff00';
        const textWidth = ctx.measureText(
          `${prediction.class} ${(prediction.score * 100).toFixed(0)}%`
        ).width;
        ctx.fillRect(x, y - 25, textWidth + 10, 25);

        // Draw label text
        ctx.fillStyle = '#000000';
        ctx.font = '16px Arial';
        ctx.fillText(
          `${prediction.class} ${(prediction.score * 100).toFixed(0)}%`,
          x + 5,
          y - 7
        );
      });

      // Calculate FPS
      frameCount++;
      const currentTime = performance.now();
      if (currentTime - lastTime >= 1000) {
        fps = frameCount;
        frameCount = 0;
        lastTime = currentTime;
      }

      requestAnimationFrame(detect);
    }

    detect();
  }

  function stopDetection() {
    isDetecting = false;
    if (stream) {
      stream.getTracks().forEach((track) => track.stop());
      stream = null;
    }
  }
</script>

<div class="detector-container">
  <h2>Real-time Object Detection</h2>
  <p class="subtitle">Detect objects using your webcam with COCO-SSD</p>

  <div class="video-container">
    <video bind:this={videoElement} style="display: none;"></video>
    <canvas bind:this={canvasElement} class="detection-canvas"></canvas>

    {#if isDetecting}
      <div class="stats">
        <div class="stat-item">
          <span class="stat-label">FPS:</span>
          <span class="stat-value">{fps}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Objects:</span>
          <span class="stat-value">{detectionCount}</span>
        </div>
      </div>
    {/if}
  </div>

  <div class="controls">
    {#if !isDetecting}
      <button on:click={startWebcam} class="btn-start">Start Detection</button>
    {:else}
      <button on:click={stopDetection} class="btn-stop">Stop Detection</button>
    {/if}
  </div>
</div>

<style>
  .detector-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
    text-align: center;
  }

  h2 {
    font-size: 28px;
    margin-bottom: 10px;
    color: #2d3748;
  }

  .subtitle {
    color: #718096;
    margin-bottom: 20px;
  }

  .video-container {
    position: relative;
    margin-bottom: 20px;
  }

  .detection-canvas {
    width: 100%;
    border-radius: 12px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .stats {
    position: absolute;
    top: 10px;
    left: 10px;
    display: flex;
    gap: 15px;
  }

  .stat-item {
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 8px 12px;
    border-radius: 6px;
    font-size: 14px;
  }

  .stat-label {
    font-weight: 500;
    margin-right: 5px;
  }

  .stat-value {
    font-weight: bold;
    color: #00ff00;
  }

  .controls {
    display: flex;
    justify-content: center;
    gap: 10px;
  }

  .btn-start,
  .btn-stop {
    padding: 12px 24px;
    border-radius: 8px;
    font-weight: 500;
    cursor: pointer;
    border: none;
    transition: all 0.2s;
  }

  .btn-start {
    background: #48bb78;
    color: white;
  }

  .btn-start:hover {
    background: #38a169;
  }

  .btn-stop {
    background: #f56565;
    color: white;
  }

  .btn-stop:hover {
    background: #e53e3e;
  }
</style>
```

### Step 5: Performance Monitor Component

Create `src/lib/components/PerformanceMonitor.svelte`:

```svelte
<script lang="ts">
  import { performanceMetrics } from '../stores/tfjs';
  import * as tf from '@tensorflow/tfjs';

  function formatBytes(bytes: number): string {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return `${(bytes / Math.pow(k, i)).toFixed(2)} ${sizes[i]}`;
  }

  function getBackendInfo() {
    return {
      backend: tf.getBackend(),
      flags: tf.env().getFlags(),
    };
  }

  $: backendInfo = getBackendInfo();
</script>

<div class="performance-monitor">
  <h3>Performance Metrics</h3>

  <div class="metrics-grid">
    <div class="metric-card">
      <div class="metric-label">Avg Inference Time</div>
      <div class="metric-value">
        {$performanceMetrics.avgInferenceTime.toFixed(2)}ms
      </div>
    </div>

    <div class="metric-card">
      <div class="metric-label">Active Tensors</div>
      <div class="metric-value">{$performanceMetrics.numTensors}</div>
    </div>

    <div class="metric-card">
      <div class="metric-label">Memory Usage</div>
      <div class="metric-value">
        {formatBytes($performanceMetrics.memoryUsage)}
      </div>
    </div>

    <div class="metric-card">
      <div class="metric-label">Backend</div>
      <div class="metric-value">{backendInfo.backend}</div>
    </div>
  </div>
</div>

<style>
  .performance-monitor {
    background: #f7fafc;
    padding: 20px;
    border-radius: 12px;
    margin-top: 20px;
  }

  h3 {
    margin-bottom: 15px;
    color: #2d3748;
  }

  .metrics-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    gap: 15px;
  }

  .metric-card {
    background: white;
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  .metric-label {
    font-size: 12px;
    color: #718096;
    margin-bottom: 5px;
  }

  .metric-value {
    font-size: 20px;
    font-weight: bold;
    color: #2d3748;
  }
</style>
```

## Expected Outputs

1. **Browser-based ML Application**:
   - No backend required
   - Fast model loading
   - Real-time inference

2. **Multiple Demos**:
   - Image classification
   - Object detection
   - Pose estimation (bonus)

3. **Performance**:
   - < 500KB bundle size
   - < 100ms inference
   - Smooth 60 FPS

4. **Developer Experience**:
   - Simple Svelte components
   - Reactive state management
   - Clean code structure

## Bonus Challenges

1. **Custom Model**: Train and deploy custom TensorFlow.js model
2. **Model Quantization**: Implement 8-bit quantized models
3. **Multi-backend**: Compare WebGL vs WASM performance
4. **Transfer Learning**: Implement in-browser training
5. **PWA**: Add offline support with service workers
6. **WebWorker**: Run inference in Web Worker
7. **Model Caching**: Smart caching strategies
8. **Batch Inference**: Optimize for batch processing

## Resources

- [Svelte Documentation](https://svelte.dev/docs)
- [SvelteKit Documentation](https://kit.svelte.dev/docs)
- [TensorFlow.js](https://www.tensorflow.org/js)
- [TensorFlow.js Models](https://github.com/tensorflow/tfjs-models)
- [Svelte Stores](https://svelte.dev/docs#run-time-svelte-store)

## Success Criteria

### Functionality (40%)
- [ ] Model loading working
- [ ] Inference functional
- [ ] Multiple input methods
- [ ] Real-time updates
- [ ] Memory management

### Performance (30%)
- [ ] Bundle size < 500KB
- [ ] Fast inference times
- [ ] Proper tensor cleanup
- [ ] Smooth animations
- [ ] Efficient rendering

### Code Quality (20%)
- [ ] Clean Svelte components
- [ ] Reactive declarations
- [ ] Store patterns
- [ ] TypeScript typing
- [ ] Error handling

### User Experience (10%)
- [ ] Intuitive interface
- [ ] Loading indicators
- [ ] Responsive design
- [ ] Performance metrics
- [ ] Clear feedback
