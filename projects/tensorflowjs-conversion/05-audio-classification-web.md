# Project 5: Convert Audio Classification Model for Web

## Overview
Build a browser-based audio classification application that can recognize sounds, music genres, or speech commands in real-time. This project covers converting audio models to TensorFlow.js, implementing audio preprocessing in JavaScript, and creating an interactive web application for microphone-based audio classification.

## Difficulty Level
Advanced

## Learning Objectives
- Train or convert audio classification models to TensorFlow.js
- Implement audio preprocessing (mel-spectrograms, MFCCs) in JavaScript
- Work with Web Audio API for microphone access
- Handle real-time audio buffering and processing
- Visualize audio waveforms and spectrograms
- Optimize audio models for browser inference
- Create responsive audio-based user interfaces

## Technical Stack
- **Model**: YAMNet, Audio Classifier CNN, or Custom Model
- **Backend**: Python, TensorFlow/Keras, librosa
- **Conversion**: tensorflowjs_converter
- **Frontend**: HTML5, JavaScript ES6+, Web Audio API
- **ML Framework**: TensorFlow.js
- **Audio Processing**: AudioContext, MediaRecorder
- **Visualization**: Canvas for waveform/spectrogram display

## Project Requirements

### 1. Model Training/Preparation
- Use pre-trained YAMNet or train custom audio classifier
- Support common audio classification tasks (speech commands, ESC-50, etc.)
- Handle variable-length audio inputs
- Export model with preprocessing configuration

### 2. Audio Preprocessing Pipeline
- Implement audio resampling in JavaScript
- Convert audio to mel-spectrograms or MFCCs
- Match Python preprocessing exactly
- Handle windowing and frame extraction

### 3. Real-Time Classification
- Access microphone via Web Audio API
- Buffer audio data appropriately
- Process audio in real-time or near real-time
- Display top predictions with confidence
- Handle continuous listening mode

### 4. Visualization and UI
- Display live waveform visualization
- Show spectrogram representation
- Present classification results clearly
- Add recording playback functionality
- Implement audio level meter

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
mkdir audio-classification-web
cd audio-classification-web

# Setup Python environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install tensorflow tensorflowjs librosa numpy soundfile matplotlib
```

### Step 2: Train Custom Audio Classifier
```python
# train_audio_classifier.py
import tensorflow as tf
import numpy as np
import librosa
import os
import json

# Audio preprocessing configuration
SAMPLE_RATE = 16000
DURATION = 1.0  # seconds
N_MELS = 128
N_FFT = 2048
HOP_LENGTH = 512

def audio_to_melspectrogram(audio_path):
    """Convert audio file to mel-spectrogram"""
    # Load audio
    audio, sr = librosa.load(audio_path, sr=SAMPLE_RATE, duration=DURATION)

    # Pad if necessary
    if len(audio) < SAMPLE_RATE * DURATION:
        audio = np.pad(audio, (0, int(SAMPLE_RATE * DURATION) - len(audio)))

    # Compute mel-spectrogram
    mel_spec = librosa.feature.melspectrogram(
        y=audio,
        sr=SAMPLE_RATE,
        n_mels=N_MELS,
        n_fft=N_FFT,
        hop_length=HOP_LENGTH
    )

    # Convert to log scale
    mel_spec_db = librosa.power_to_db(mel_spec, ref=np.max)

    # Normalize
    mel_spec_db = (mel_spec_db + 80) / 80  # Normalize to [0, 1]

    return mel_spec_db

# Example: Speech Commands Dataset
# Classes: yes, no, up, down, left, right, on, off, stop, go
CLASSES = ['yes', 'no', 'up', 'down', 'left', 'right', 'on', 'off', 'stop', 'go']

def build_audio_model(num_classes=10):
    """Build CNN for audio classification"""
    model = tf.keras.Sequential([
        tf.keras.layers.Input(shape=(N_MELS, None, 1)),

        # Convolutional blocks
        tf.keras.layers.Conv2D(32, (3, 3), activation='relu', padding='same'),
        tf.keras.layers.MaxPooling2D((2, 2)),
        tf.keras.layers.Dropout(0.25),

        tf.keras.layers.Conv2D(64, (3, 3), activation='relu', padding='same'),
        tf.keras.layers.MaxPooling2D((2, 2)),
        tf.keras.layers.Dropout(0.25),

        tf.keras.layers.Conv2D(128, (3, 3), activation='relu', padding='same'),
        tf.keras.layers.MaxPooling2D((2, 2)),
        tf.keras.layers.Dropout(0.25),

        # Global pooling to handle variable time dimension
        tf.keras.layers.GlobalAveragePooling2D(),

        # Dense layers
        tf.keras.layers.Dense(128, activation='relu'),
        tf.keras.layers.Dropout(0.5),
        tf.keras.layers.Dense(num_classes, activation='softmax')
    ])

    return model

# Build model
model = build_audio_model(num_classes=len(CLASSES))

model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

print(model.summary())

# After training (placeholder - add your training code)
# model.fit(train_data, train_labels, epochs=50, validation_split=0.2)

# Save model
model.save('audio_classifier.h5')
print("Model saved!")

# Save preprocessing config
config = {
    'sample_rate': SAMPLE_RATE,
    'duration': DURATION,
    'n_mels': N_MELS,
    'n_fft': N_FFT,
    'hop_length': HOP_LENGTH,
    'classes': CLASSES
}

with open('audio_config.json', 'w') as f:
    json.dump(config, f, indent=2)

print("Configuration saved!")
```

### Step 3: Convert Model to TensorFlow.js
```bash
# Convert Keras model to TFJS
tensorflowjs_converter \
    --input_format=keras \
    --quantize_float16 \
    audio_classifier.h5 \
    ./tfjs_audio_model

echo "Model converted to TensorFlow.js!"
ls -lh tfjs_audio_model/
```

### Step 4: Create HTML Interface
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Audio Classification - TensorFlow.js</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
            min-height: 100vh;
            padding: 20px;
            color: white;
        }

        .container {
            max-width: 900px;
            margin: 0 auto;
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            border-radius: 20px;
            padding: 30px;
            box-shadow: 0 10px 50px rgba(0,0,0,0.3);
        }

        h1 {
            text-align: center;
            font-size: 36px;
            margin-bottom: 10px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
        }

        .visualizer {
            background: rgba(0, 0, 0, 0.3);
            border-radius: 10px;
            margin: 20px 0;
            overflow: hidden;
        }

        canvas {
            width: 100%;
            display: block;
        }

        .controls {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
            gap: 15px;
            margin: 20px 0;
        }

        button {
            padding: 15px 25px;
            font-size: 16px;
            font-weight: 600;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            transition: all 0.3s;
            text-transform: uppercase;
            letter-spacing: 1px;
        }

        .btn-start {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }

        .btn-stop {
            background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
            color: white;
        }

        .btn-record {
            background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
            color: white;
        }

        button:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(0,0,0,0.3);
        }

        button:disabled {
            background: #555;
            cursor: not-allowed;
            transform: none;
        }

        .status {
            text-align: center;
            padding: 15px;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 10px;
            margin: 20px 0;
            font-weight: 600;
            font-size: 18px;
        }

        .predictions {
            background: rgba(255, 255, 255, 0.1);
            padding: 20px;
            border-radius: 10px;
            margin-top: 20px;
        }

        .prediction-item {
            display: flex;
            align-items: center;
            margin: 12px 0;
            padding: 12px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 8px;
            transition: all 0.3s;
        }

        .prediction-item:hover {
            background: rgba(255, 255, 255, 0.2);
        }

        .prediction-label {
            flex: 1;
            font-size: 18px;
            font-weight: 600;
        }

        .prediction-bar {
            flex: 2;
            height: 30px;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 15px;
            overflow: hidden;
            margin: 0 15px;
        }

        .prediction-fill {
            height: 100%;
            background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
            transition: width 0.3s;
            display: flex;
            align-items: center;
            justify-content: flex-end;
            padding-right: 10px;
        }

        .prediction-percentage {
            font-weight: 700;
            font-size: 16px;
        }

        .audio-level {
            height: 10px;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 5px;
            overflow: hidden;
            margin: 10px 0;
        }

        .audio-level-fill {
            height: 100%;
            background: linear-gradient(90deg, #4facfe 0%, #00f2fe 100%);
            width: 0%;
            transition: width 0.1s;
        }

        @media (max-width: 768px) {
            h1 {
                font-size: 24px;
            }

            .controls {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎵 Audio Classifier</h1>
        <p style="text-align: center; margin-bottom: 20px;">
            Powered by TensorFlow.js - Real-time sound recognition
        </p>

        <div class="status" id="status">Loading model...</div>

        <div class="visualizer">
            <canvas id="waveformCanvas" width="800" height="150"></canvas>
        </div>

        <div class="audio-level">
            <div class="audio-level-fill" id="audioLevel"></div>
        </div>

        <div class="controls">
            <button class="btn-start" id="startBtn" onclick="startListening()" disabled>
                🎤 Start Listening
            </button>
            <button class="btn-stop" id="stopBtn" onclick="stopListening()" disabled>
                ⏹️ Stop
            </button>
            <button class="btn-record" id="recordBtn" onclick="toggleRecording()" disabled>
                🔴 Record Audio
            </button>
        </div>

        <div class="predictions" id="predictions">
            <h3>Predictions will appear here...</h3>
        </div>
    </div>

    <script src="audio-processor.js"></script>
    <script src="app.js"></script>
</body>
</html>
```

### Step 5: Implement Audio Processing
```javascript
// audio-processor.js
class AudioProcessor {
    constructor() {
        this.audioContext = null;
        this.analyser = null;
        this.microphone = null;
        this.scriptProcessor = null;
        this.audioBuffer = [];
        this.config = null;
    }

    async loadConfig() {
        const response = await fetch('audio_config.json');
        this.config = await response.json();
        console.log('Audio config loaded:', this.config);
    }

    async initialize() {
        await this.loadConfig();

        this.audioContext = new (window.AudioContext || window.webkitAudioContext)({
            sampleRate: this.config.sample_rate
        });

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048;

        console.log('Audio processor initialized');
    }

    async startMicrophone() {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });

            this.microphone = this.audioContext.createMediaStreamSource(stream);
            this.microphone.connect(this.analyser);

            // Create script processor for audio data
            this.scriptProcessor = this.audioContext.createScriptProcessor(4096, 1, 1);

            this.scriptProcessor.onaudioprocess = (event) => {
                const inputData = event.inputBuffer.getChannelData(0);
                this.audioBuffer.push(...inputData);

                // Keep buffer at appropriate length
                const maxLength = this.config.sample_rate * this.config.duration;
                if (this.audioBuffer.length > maxLength) {
                    this.audioBuffer = this.audioBuffer.slice(-maxLength);
                }
            };

            this.analyser.connect(this.scriptProcessor);
            this.scriptProcessor.connect(this.audioContext.destination);

            return true;

        } catch (error) {
            console.error('Microphone access error:', error);
            return false;
        }
    }

    stopMicrophone() {
        if (this.scriptProcessor) {
            this.scriptProcessor.disconnect();
            this.scriptProcessor = null;
        }

        if (this.microphone) {
            this.microphone.disconnect();
            this.microphone.mediaStream.getTracks().forEach(track => track.stop());
            this.microphone = null;
        }

        this.audioBuffer = [];
    }

    getWaveformData() {
        const bufferLength = this.analyser.frequencyBinCount;
        const dataArray = new Uint8Array(bufferLength);
        this.analyser.getByteTimeDomainData(dataArray);
        return dataArray;
    }

    getAudioLevel() {
        const bufferLength = this.analyser.frequencyBinCount;
        const dataArray = new Uint8Array(bufferLength);
        this.analyser.getByteTimeDomainData(dataArray);

        let sum = 0;
        for (let i = 0; i < bufferLength; i++) {
            const normalized = (dataArray[i] - 128) / 128;
            sum += normalized * normalized;
        }

        return Math.sqrt(sum / bufferLength);
    }

    audioToMelSpectrogram(audioData) {
        // Simplified mel-spectrogram computation in JavaScript
        // Note: This is a simplified version. For production, consider using a library
        // or pre-computing spectrograms on the server

        const sampleRate = this.config.sample_rate;
        const nFFT = this.config.n_fft;
        const hopLength = this.config.hop_length;
        const nMels = this.config.n_mels;

        // Ensure audio is the right length
        let audio = new Float32Array(audioData);
        const targetLength = sampleRate * this.config.duration;

        if (audio.length < targetLength) {
            const padded = new Float32Array(targetLength);
            padded.set(audio);
            audio = padded;
        } else {
            audio = audio.slice(0, targetLength);
        }

        // For this example, we'll use a simplified approach
        // In production, use proper STFT and mel filterbank
        // You might want to use a library like meyda.js

        return audio;  // Placeholder - implement full mel-spectrogram
    }

    getAudioForInference() {
        if (this.audioBuffer.length === 0) {
            return null;
        }

        const targetLength = this.config.sample_rate * this.config.duration;
        const audio = new Float32Array(targetLength);

        // Get the last N seconds of audio
        const start = Math.max(0, this.audioBuffer.length - targetLength);
        const slice = this.audioBuffer.slice(start);

        audio.set(slice);

        return audio;
    }
}
```

### Step 6: Implement Main Application
```javascript
// app.js
let model = null;
let audioProcessor = null;
let isListening = false;
let classificationInterval = null;
let config = null;

const statusDiv = document.getElementById('status');
const startBtn = document.getElementById('startBtn');
const stopBtn = document.getElementById('stopBtn');
const recordBtn = document.getElementById('recordBtn');
const predictionsDiv = document.getElementById('predictions');

// Waveform visualization
const waveformCanvas = document.getElementById('waveformCanvas');
const waveformCtx = waveformCanvas.getContext('2d');

// Load model
async function loadModel() {
    statusDiv.textContent = 'Loading audio classification model...';

    try {
        // Load config
        const response = await fetch('audio_config.json');
        config = await response.json();

        // Initialize audio processor
        audioProcessor = new AudioProcessor();
        await audioProcessor.initialize();

        // Load TensorFlow.js model
        model = await tf.loadLayersModel('./tfjs_audio_model/model.json');

        console.log('Model loaded successfully');

        // Warm up model
        tf.tidy(() => {
            const warmup = tf.zeros([1, config.n_mels, 32, 1]);
            model.predict(warmup);
        });

        statusDiv.textContent = 'Model ready! Click "Start Listening"';
        startBtn.disabled = false;
        recordBtn.disabled = false;

    } catch (error) {
        console.error('Error loading model:', error);
        statusDiv.textContent = 'Error loading model: ' + error.message;
    }
}

// Start listening
async function startListening() {
    const success = await audioProcessor.startMicrophone();

    if (!success) {
        alert('Could not access microphone');
        return;
    }

    isListening = true;
    startBtn.disabled = true;
    stopBtn.disabled = false;

    statusDiv.textContent = 'Listening...';

    // Start visualization
    visualizeWaveform();

    // Start classification (every 500ms)
    classificationInterval = setInterval(classifyAudio, 500);
}

// Stop listening
function stopListening() {
    isListening = false;
    audioProcessor.stopMicrophone();

    startBtn.disabled = false;
    stopBtn.disabled = true;

    statusDiv.textContent = 'Stopped';

    clearInterval(classificationInterval);
}

// Visualize waveform
function visualizeWaveform() {
    if (!isListening) return;

    const waveformData = audioProcessor.getWaveformData();
    const width = waveformCanvas.width;
    const height = waveformCanvas.height;

    waveformCtx.fillStyle = 'rgba(0, 0, 0, 0.1)';
    waveformCtx.fillRect(0, 0, width, height);

    waveformCtx.lineWidth = 2;
    waveformCtx.strokeStyle = '#00f2fe';
    waveformCtx.beginPath();

    const sliceWidth = width / waveformData.length;
    let x = 0;

    for (let i = 0; i < waveformData.length; i++) {
        const v = waveformData[i] / 128.0;
        const y = v * height / 2;

        if (i === 0) {
            waveformCtx.moveTo(x, y);
        } else {
            waveformCtx.lineTo(x, y);
        }

        x += sliceWidth;
    }

    waveformCtx.lineTo(width, height / 2);
    waveformCtx.stroke();

    // Update audio level
    const level = audioProcessor.getAudioLevel();
    document.getElementById('audioLevel').style.width = `${level * 100}%`;

    requestAnimationFrame(visualizeWaveform);
}

// Classify audio
async function classifyAudio() {
    if (!isListening || !model) return;

    try {
        const audioData = audioProcessor.getAudioForInference();

        if (!audioData) return;

        // Convert audio to model input format
        // Note: Simplified - in production, compute proper mel-spectrogram
        const inputTensor = tf.tidy(() => {
            // This is a simplified version - implement proper preprocessing
            let tensor = tf.tensor(audioData);

            // Reshape for model (add batch, channels)
            // Actual shape depends on your model
            tensor = tensor.reshape([1, config.n_mels, -1, 1]);

            return tensor;
        });

        // Run inference
        const predictions = await model.predict(inputTensor).data();

        inputTensor.dispose();

        // Display results
        displayPredictions(predictions);

    } catch (error) {
        console.error('Classification error:', error);
    }
}

// Display predictions
function displayPredictions(predictions) {
    const topPredictions = Array.from(predictions)
        .map((prob, index) => ({
            label: config.classes[index],
            probability: prob
        }))
        .sort((a, b) => b.probability - a.probability)
        .slice(0, 5);

    predictionsDiv.innerHTML = '<h3>Predictions:</h3>';

    topPredictions.forEach(pred => {
        const percentage = (pred.probability * 100).toFixed(1);

        predictionsDiv.innerHTML += `
            <div class="prediction-item">
                <div class="prediction-label">${pred.label}</div>
                <div class="prediction-bar">
                    <div class="prediction-fill" style="width: ${percentage}%">
                        <span class="prediction-percentage">${percentage}%</span>
                    </div>
                </div>
            </div>
        `;
    });
}

// Toggle recording
let mediaRecorder = null;
let recordedChunks = [];

async function toggleRecording() {
    if (!mediaRecorder || mediaRecorder.state === 'inactive') {
        // Start recording
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        mediaRecorder = new MediaRecorder(stream);

        recordedChunks = [];

        mediaRecorder.ondataavailable = (e) => {
            if (e.data.size > 0) {
                recordedChunks.push(e.data);
            }
        };

        mediaRecorder.onstop = () => {
            const blob = new Blob(recordedChunks, { type: 'audio/webm' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `recording-${Date.now()}.webm`;
            a.click();
            URL.revokeObjectURL(url);
        };

        mediaRecorder.start();
        recordBtn.textContent = '⏺️ Stop Recording';
        recordBtn.style.background = 'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)';

    } else {
        // Stop recording
        mediaRecorder.stop();
        recordBtn.textContent = '🔴 Record Audio';
        recordBtn.style.background = 'linear-gradient(135deg, #4facfe 0%, #00f2fe 100%)';
    }
}

// Initialize
loadModel();
```

## Expected Outputs

1. **Model Performance**:
   - Load time: <5 seconds
   - Inference time: 50-150ms
   - Real-time classification (2-4 Hz update rate)
   - Accuracy: >85% on trained classes

2. **Audio Processing**:
   - Microphone access working
   - Audio buffering at 16kHz
   - Waveform visualization smooth
   - Audio level meter responsive

3. **Classification Results**:
   - Top 5 predictions displayed
   - Confidence scores shown
   - Updates in near real-time
   - Accurate recognition of trained sounds

4. **User Interface**:
   - Responsive controls
   - Visual feedback
   - Recording functionality
   - Clear status messages

## Bonus Challenges

- [ ] Implement proper mel-spectrogram computation in JavaScript
- [ ] Add spectrogram visualization
- [ ] Support audio file upload and classification
- [ ] Add voice activity detection (VAD)
- [ ] Implement continuous recording with timestamped predictions
- [ ] Add background noise reduction
- [ ] Create audio augmentation preview
- [ ] Support multiple languages for speech commands
- [ ] Add speaker identification mode
- [ ] Implement audio emotion recognition
- [ ] Create sound event detection timeline
- [ ] Add export functionality for predictions (CSV/JSON)

## Resources

- [YAMNet Audio Classifier](https://tfhub.dev/google/tfjs-model/yamnet/tfjs/1)
- [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [Speech Commands Dataset](https://www.tensorflow.org/datasets/catalog/speech_commands)
- [ESC-50 Dataset](https://github.com/karolpiczak/ESC-50)
- [librosa Documentation](https://librosa.org/doc/latest/index.html)
- [Meyda.js Audio Feature Extraction](https://meyda.js.org/)
- [TensorFlow.js Audio Tutorial](https://www.tensorflow.org/js/tutorials/transfer/audio_recognizer)

## Success Criteria

- Model converts successfully to TensorFlow.js
- Microphone access works in modern browsers
- Audio preprocessing pipeline functional
- Real-time classification achieves >5 predictions/second
- Waveform visualization displays correctly
- Audio level meter responds to sound
- Classification accuracy matches Python model (>85%)
- Recording and playback functionality works
- No memory leaks during extended use
- Works in Chrome, Firefox, and Safari
- User interface is intuitive and responsive
- Error handling for no microphone access
