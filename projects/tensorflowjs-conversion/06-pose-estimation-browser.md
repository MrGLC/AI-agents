# Project 6: Build In-Browser Pose Estimation Application

## Overview
Create a real-time human pose estimation application that runs entirely in the browser using TensorFlow.js. Detect and track body keypoints, skeletal structure, and movements using pre-trained PoseNet or MoveNet models, enabling applications like fitness tracking, gesture recognition, and motion analysis.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Work with pre-trained pose estimation models (PoseNet, MoveNet)
- Convert and optimize pose models for browser deployment
- Implement keypoint detection and tracking
- Draw skeletal overlays on video streams
- Calculate angles and distances between keypoints
- Build fitness/exercise tracking applications
- Optimize for real-time performance (30+ FPS)

## Technical Stack
- **Model**: MoveNet or PoseNet (pre-trained)
- **Frontend**: HTML5 Canvas, JavaScript ES6+
- **ML Framework**: TensorFlow.js
- **Video**: WebRTC getUserMedia API
- **Visualization**: Canvas 2D rendering with skeletal overlay
- **Math**: Angle calculation, distance measurement
- **Performance**: RequestAnimationFrame, Web Workers

## Project Requirements

### 1. Model Integration
- Load pre-trained MoveNet or PoseNet model
- Support single-pose and multi-pose detection
- Handle different confidence thresholds
- Optimize model selection for performance

### 2. Pose Detection Pipeline
- Access webcam for video input
- Run real-time pose inference
- Extract keypoint coordinates
- Calculate confidence scores
- Track poses across frames

### 3. Visualization
- Draw keypoints on detected body parts
- Render skeletal structure connecting keypoints
- Color-code by confidence level
- Display keypoint labels
- Add pose confidence indicators

### 4. Advanced Features
- Calculate joint angles (elbow, knee, etc.)
- Measure distances between keypoints
- Count exercise repetitions
- Detect specific poses or gestures
- Provide real-time feedback

## Step-by-Step Implementation

### Step 1: Setup Project Structure
```bash
mkdir pose-estimation-app
cd pose-estimation-app

# Create project structure
mkdir css js
touch index.html js/app.js js/pose-detector.js js/pose-analyzer.js css/style.css
```

### Step 2: Create Pose Detector Class
```javascript
// js/pose-detector.js
class PoseDetector {
    constructor() {
        this.model = null;
        this.modelType = 'MoveNet';  // or 'PoseNet'
    }

    async loadModel(modelType = 'MoveNet') {
        this.modelType = modelType;

        console.log(`Loading ${modelType} model...`);

        if (modelType === 'MoveNet') {
            // Load MoveNet (more accurate, single pose)
            this.model = await poseDetection.createDetector(
                poseDetection.SupportedModels.MoveNet,
                {
                    modelType: poseDetection.movenet.modelType.SINGLEPOSE_LIGHTNING,
                    enableSmoothing: true
                }
            );
        } else {
            // Load PoseNet (multi-pose support)
            this.model = await poseDetection.createDetector(
                poseDetection.SupportedModels.PoseNet,
                {
                    architecture: 'MobileNetV1',
                    outputStride: 16,
                    inputResolution: { width: 640, height: 480 },
                    multiplier: 0.75
                }
            );
        }

        console.log('Model loaded successfully');
    }

    async detectPose(videoElement) {
        if (!this.model) {
            throw new Error('Model not loaded');
        }

        const poses = await this.model.estimatePoses(videoElement, {
            maxPoses: 1,
            flipHorizontal: false,
            scoreThreshold: 0.5
        });

        return poses;
    }

    getKeypoint(pose, name) {
        return pose.keypoints.find(kp => kp.name === name);
    }
}

// Keypoint names for reference
const KEYPOINT_NAMES = [
    'nose',
    'left_eye', 'right_eye',
    'left_ear', 'right_ear',
    'left_shoulder', 'right_shoulder',
    'left_elbow', 'right_elbow',
    'left_wrist', 'right_wrist',
    'left_hip', 'right_hip',
    'left_knee', 'right_knee',
    'left_ankle', 'right_ankle'
];

// Skeleton connections
const POSE_CONNECTIONS = [
    ['left_shoulder', 'right_shoulder'],
    ['left_shoulder', 'left_elbow'],
    ['left_elbow', 'left_wrist'],
    ['right_shoulder', 'right_elbow'],
    ['right_elbow', 'right_wrist'],
    ['left_shoulder', 'left_hip'],
    ['right_shoulder', 'right_hip'],
    ['left_hip', 'right_hip'],
    ['left_hip', 'left_knee'],
    ['left_knee', 'left_ankle'],
    ['right_hip', 'right_knee'],
    ['right_knee', 'right_ankle']
];
```

### Step 3: Create Pose Analyzer Class
```javascript
// js/pose-analyzer.js
class PoseAnalyzer {
    constructor() {
        this.previousPoses = [];
        this.exerciseCounter = {
            squats: 0,
            pushups: 0,
            jumping_jacks: 0
        };
        this.exerciseState = null;
    }

    // Calculate angle between three points
    calculateAngle(pointA, pointB, pointC) {
        const radians = Math.atan2(pointC.y - pointB.y, pointC.x - pointB.x) -
                       Math.atan2(pointA.y - pointB.y, pointA.x - pointB.x);

        let angle = Math.abs(radians * 180.0 / Math.PI);

        if (angle > 180.0) {
            angle = 360.0 - angle;
        }

        return angle;
    }

    // Calculate distance between two points
    calculateDistance(pointA, pointB) {
        return Math.sqrt(
            Math.pow(pointB.x - pointA.x, 2) +
            Math.pow(pointB.y - pointA.y, 2)
        );
    }

    // Get joint angles from pose
    getJointAngles(pose) {
        const getKp = (name) => pose.keypoints.find(kp => kp.name === name);

        const leftElbow = getKp('left_elbow');
        const leftShoulder = getKp('left_shoulder');
        const leftWrist = getKp('left_wrist');
        const leftHip = getKp('left_hip');
        const leftKnee = getKp('left_knee');
        const leftAnkle = getKp('left_ankle');

        const rightElbow = getKp('right_elbow');
        const rightShoulder = getKp('right_shoulder');
        const rightWrist = getKp('right_wrist');
        const rightHip = getKp('right_hip');
        const rightKnee = getKp('right_knee');
        const rightAnkle = getKp('right_ankle');

        const angles = {};

        // Left arm angle
        if (leftShoulder?.score > 0.5 && leftElbow?.score > 0.5 && leftWrist?.score > 0.5) {
            angles.leftElbow = this.calculateAngle(leftShoulder, leftElbow, leftWrist);
        }

        // Right arm angle
        if (rightShoulder?.score > 0.5 && rightElbow?.score > 0.5 && rightWrist?.score > 0.5) {
            angles.rightElbow = this.calculateAngle(rightShoulder, rightElbow, rightWrist);
        }

        // Left leg angle
        if (leftHip?.score > 0.5 && leftKnee?.score > 0.5 && leftAnkle?.score > 0.5) {
            angles.leftKnee = this.calculateAngle(leftHip, leftKnee, leftAnkle);
        }

        // Right leg angle
        if (rightHip?.score > 0.5 && rightKnee?.score > 0.5 && rightAnkle?.score > 0.5) {
            angles.rightKnee = this.calculateAngle(rightHip, rightKnee, rightAnkle);
        }

        return angles;
    }

    // Count squats
    countSquats(pose) {
        const angles = this.getJointAngles(pose);

        if (!angles.leftKnee && !angles.rightKnee) return;

        const kneeAngle = Math.min(angles.leftKnee || 180, angles.rightKnee || 180);

        // Squat down position (knees bent)
        if (kneeAngle < 90 && this.exerciseState !== 'down') {
            this.exerciseState = 'down';
        }

        // Squat up position (knees straight)
        if (kneeAngle > 160 && this.exerciseState === 'down') {
            this.exerciseCounter.squats++;
            this.exerciseState = 'up';
        }
    }

    // Detect specific poses
    detectPose(pose) {
        const angles = this.getJointAngles(pose);

        // T-Pose detection
        if (angles.leftElbow > 160 && angles.rightElbow > 160) {
            const leftShoulder = pose.keypoints.find(kp => kp.name === 'left_shoulder');
            const leftWrist = pose.keypoints.find(kp => kp.name === 'left_wrist');
            const rightShoulder = pose.keypoints.find(kp => kp.name === 'right_shoulder');
            const rightWrist = pose.keypoints.find(kp => kp.name === 'right_wrist');

            if (leftShoulder && leftWrist && rightShoulder && rightWrist) {
                const leftArmAngle = Math.abs(leftWrist.y - leftShoulder.y);
                const rightArmAngle = Math.abs(rightWrist.y - rightShoulder.y);

                if (leftArmAngle < 50 && rightArmAngle < 50) {
                    return 'T-Pose';
                }
            }
        }

        // Standing detection
        const leftKnee = pose.keypoints.find(kp => kp.name === 'left_knee');
        const rightKnee = pose.keypoints.find(kp => kp.name === 'right_knee');

        if (angles.leftKnee > 160 && angles.rightKnee > 160) {
            return 'Standing';
        }

        // Sitting/Squatting detection
        if (angles.leftKnee < 100 || angles.rightKnee < 100) {
            return 'Squatting';
        }

        return 'Unknown';
    }

    // Reset counters
    resetCounters() {
        this.exerciseCounter = {
            squats: 0,
            pushups: 0,
            jumping_jacks: 0
        };
        this.exerciseState = null;
    }
}
```

### Step 4: Create HTML Interface
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Pose Estimation - TensorFlow.js</title>

    <!-- TensorFlow.js -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>

    <!-- Pose Detection Model -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/pose-detection@2.0.0"></script>

    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>🧘 Pose Estimation & Exercise Tracker</h1>
            <p>Real-time body pose detection powered by TensorFlow.js</p>
        </header>

        <div class="main-content">
            <div class="video-section">
                <div class="video-container">
                    <video id="webcam" autoplay playsinline></video>
                    <canvas id="canvas"></canvas>
                </div>

                <div class="controls">
                    <button id="startBtn" class="btn btn-primary">
                        📹 Start Camera
                    </button>
                    <button id="stopBtn" class="btn btn-danger" disabled>
                        ⏹️ Stop
                    </button>
                    <button id="screenshotBtn" class="btn btn-secondary" disabled>
                        📸 Screenshot
                    </button>
                </div>
            </div>

            <div class="info-panel">
                <div class="status-box">
                    <h3>Status</h3>
                    <div id="status">Loading model...</div>
                </div>

                <div class="stats-box">
                    <h3>Performance</h3>
                    <div class="stat-item">
                        <span>FPS:</span>
                        <span id="fps" class="stat-value">0</span>
                    </div>
                    <div class="stat-item">
                        <span>Inference:</span>
                        <span id="inferenceTime" class="stat-value">0ms</span>
                    </div>
                    <div class="stat-item">
                        <span>Confidence:</span>
                        <span id="confidence" class="stat-value">0%</span>
                    </div>
                </div>

                <div class="pose-info-box">
                    <h3>Detected Pose</h3>
                    <div id="detectedPose" class="pose-name">None</div>
                </div>

                <div class="angles-box">
                    <h3>Joint Angles</h3>
                    <div id="angles"></div>
                </div>

                <div class="exercise-box">
                    <h3>Exercise Counter</h3>
                    <div class="exercise-counter">
                        <div class="counter-item">
                            <span class="counter-label">Squats:</span>
                            <span id="squatCount" class="counter-value">0</span>
                        </div>
                        <button id="resetBtn" class="btn btn-small">Reset</button>
                    </div>
                </div>

                <div class="settings-box">
                    <h3>Settings</h3>
                    <label>
                        <input type="checkbox" id="showKeypoints" checked>
                        Show Keypoints
                    </label>
                    <label>
                        <input type="checkbox" id="showSkeleton" checked>
                        Show Skeleton
                    </label>
                    <label>
                        <input type="checkbox" id="showAngles" checked>
                        Show Angles
                    </label>
                </div>
            </div>
        </div>
    </div>

    <script src="js/pose-detector.js"></script>
    <script src="js/pose-analyzer.js"></script>
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
    max-width: 1400px;
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

header p {
    color: #666;
    font-size: 16px;
}

.main-content {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 30px;
}

.video-section {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.video-container {
    position: relative;
    background: #000;
    border-radius: 15px;
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
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 15px;
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

.btn-small {
    padding: 8px 15px;
    font-size: 14px;
    margin-top: 10px;
}

.btn:disabled {
    background: #ccc;
    cursor: not-allowed;
}

.info-panel {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.status-box,
.stats-box,
.pose-info-box,
.angles-box,
.exercise-box,
.settings-box {
    background: #f5f5f5;
    padding: 20px;
    border-radius: 10px;
}

h3 {
    color: #333;
    margin-bottom: 15px;
    font-size: 18px;
}

#status {
    padding: 10px;
    background: #fff3cd;
    border-radius: 5px;
    text-align: center;
    font-weight: 600;
}

.stat-item {
    display: flex;
    justify-content: space-between;
    margin: 10px 0;
    padding: 8px;
    background: white;
    border-radius: 5px;
}

.stat-value {
    font-weight: 700;
    color: #667eea;
}

.pose-name {
    font-size: 24px;
    font-weight: 700;
    color: #4CAF50;
    text-align: center;
    padding: 15px;
    background: white;
    border-radius: 10px;
}

#angles {
    background: white;
    padding: 15px;
    border-radius: 5px;
    min-height: 100px;
}

.angle-item {
    display: flex;
    justify-content: space-between;
    margin: 8px 0;
    padding: 5px 0;
    border-bottom: 1px solid #eee;
}

.exercise-counter {
    background: white;
    padding: 15px;
    border-radius: 5px;
}

.counter-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-size: 20px;
}

.counter-label {
    font-weight: 600;
    color: #333;
}

.counter-value {
    font-size: 32px;
    font-weight: 700;
    color: #667eea;
}

.settings-box label {
    display: block;
    margin: 10px 0;
    cursor: pointer;
    user-select: none;
}

.settings-box input[type="checkbox"] {
    margin-right: 10px;
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
let poseDetector = null;
let poseAnalyzer = null;
let webcamStream = null;
let isRunning = false;
let animationFrameId = null;

const webcamElement = document.getElementById('webcam');
const canvasElement = document.getElementById('canvas');
const ctx = canvasElement.getContext('2d');

const startBtn = document.getElementById('startBtn');
const stopBtn = document.getElementById('stopBtn');
const screenshotBtn = document.getElementById('screenshotBtn');
const resetBtn = document.getElementById('resetBtn');
const statusDiv = document.getElementById('status');

// Settings
let showKeypoints = true;
let showSkeleton = true;
let showAngles = true;

// Performance tracking
let frameCount = 0;
let lastTime = performance.now();
let fps = 0;

// Initialize
async function init() {
    statusDiv.textContent = 'Loading pose detection model...';

    try {
        poseDetector = new PoseDetector();
        poseAnalyzer = new PoseAnalyzer();

        await poseDetector.loadModel('MoveNet');

        statusDiv.textContent = 'Model loaded! Click "Start Camera"';
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
            video: { facingMode: 'user', width: 640, height: 480 },
            audio: false
        });

        webcamElement.srcObject = webcamStream;

        webcamElement.onloadedmetadata = () => {
            canvasElement.width = webcamElement.videoWidth;
            canvasElement.height = webcamElement.videoHeight;

            startBtn.disabled = true;
            stopBtn.disabled = false;
            screenshotBtn.disabled = false;

            isRunning = true;
            detectPoseLoop();
        };

    } catch (error) {
        console.error('Webcam error:', error);
        alert('Could not access webcam: ' + error.message);
    }
}

// Stop webcam
function stopWebcam() {
    isRunning = false;

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

// Main detection loop
async function detectPoseLoop() {
    if (!isRunning) return;

    const startTime = performance.now();

    try {
        // Detect poses
        const poses = await poseDetector.detectPose(webcamElement);

        // Clear canvas
        ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);

        if (poses && poses.length > 0) {
            const pose = poses[0];

            // Draw pose
            if (showSkeleton) {
                drawSkeleton(pose);
            }

            if (showKeypoints) {
                drawKeypoints(pose);
            }

            // Analyze pose
            const angles = poseAnalyzer.getJointAngles(pose);
            const detectedPose = poseAnalyzer.detectPose(pose);

            // Count exercises
            poseAnalyzer.countSquats(pose);

            // Update UI
            updateUI(pose, angles, detectedPose, performance.now() - startTime);
        }

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

    animationFrameId = requestAnimationFrame(detectPoseLoop);
}

// Draw keypoints
function drawKeypoints(pose) {
    pose.keypoints.forEach(keypoint => {
        if (keypoint.score > 0.3) {
            ctx.beginPath();
            ctx.arc(keypoint.x, keypoint.y, 5, 0, 2 * Math.PI);

            // Color based on confidence
            if (keypoint.score > 0.7) {
                ctx.fillStyle = '#00FF00';
            } else if (keypoint.score > 0.5) {
                ctx.fillStyle = '#FFFF00';
            } else {
                ctx.fillStyle = '#FF0000';
            }

            ctx.fill();

            // Draw label
            ctx.fillStyle = 'white';
            ctx.font = 'bold 12px Arial';
            ctx.fillText(keypoint.name.replace('_', ' '), keypoint.x + 8, keypoint.y + 5);
        }
    });
}

// Draw skeleton
function drawSkeleton(pose) {
    const getKp = (name) => pose.keypoints.find(kp => kp.name === name);

    POSE_CONNECTIONS.forEach(([partA, partB]) => {
        const keypointA = getKp(partA);
        const keypointB = getKp(partB);

        if (keypointA?.score > 0.3 && keypointB?.score > 0.3) {
            ctx.beginPath();
            ctx.moveTo(keypointA.x, keypointA.y);
            ctx.lineTo(keypointB.x, keypointB.y);
            ctx.strokeStyle = '#00FFFF';
            ctx.lineWidth = 3;
            ctx.stroke();
        }
    });
}

// Update UI
function updateUI(pose, angles, detectedPose, inferenceTime) {
    // Inference time
    document.getElementById('inferenceTime').textContent = `${inferenceTime.toFixed(0)}ms`;

    // Average confidence
    const avgConfidence = pose.keypoints.reduce((sum, kp) => sum + kp.score, 0) / pose.keypoints.length;
    document.getElementById('confidence').textContent = `${(avgConfidence * 100).toFixed(0)}%`;

    // Detected pose
    document.getElementById('detectedPose').textContent = detectedPose;

    // Joint angles
    const anglesDiv = document.getElementById('angles');
    anglesDiv.innerHTML = '';

    if (showAngles) {
        Object.entries(angles).forEach(([joint, angle]) => {
            const angleItem = document.createElement('div');
            angleItem.className = 'angle-item';
            angleItem.innerHTML = `
                <span>${joint.replace(/([A-Z])/g, ' $1').trim()}:</span>
                <span>${angle.toFixed(1)}°</span>
            `;
            anglesDiv.appendChild(angleItem);
        });
    }

    // Exercise counters
    document.getElementById('squatCount').textContent = poseAnalyzer.exerciseCounter.squats;
}

// Take screenshot
function takeScreenshot() {
    const screenshotCanvas = document.createElement('canvas');
    screenshotCanvas.width = canvasElement.width;
    screenshotCanvas.height = canvasElement.height;
    const screenshotCtx = screenshotCanvas.getContext('2d');

    screenshotCtx.drawImage(webcamElement, 0, 0);
    screenshotCtx.drawImage(canvasElement, 0, 0);

    screenshotCanvas.toBlob(blob => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `pose-${Date.now()}.png`;
        a.click();
        URL.revokeObjectURL(url);
    });
}

// Event listeners
startBtn.addEventListener('click', startWebcam);
stopBtn.addEventListener('click', stopWebcam);
screenshotBtn.addEventListener('click', takeScreenshot);
resetBtn.addEventListener('click', () => poseAnalyzer.resetCounters());

document.getElementById('showKeypoints').addEventListener('change', (e) => {
    showKeypoints = e.target.checked;
});

document.getElementById('showSkeleton').addEventListener('change', (e) => {
    showSkeleton = e.target.checked;
});

document.getElementById('showAngles').addEventListener('change', (e) => {
    showAngles = e.target.checked;
});

// Initialize
init();
```

## Expected Outputs

1. **Pose Detection**:
   - 17 keypoints detected per person
   - Skeletal structure overlay
   - Real-time tracking at 20-30 FPS
   - Confidence scores for each keypoint

2. **Measurements**:
   - Joint angles (elbows, knees, etc.)
   - Distance calculations
   - Pose classification (standing, squatting, etc.)

3. **Exercise Tracking**:
   - Squat counting
   - Pose feedback
   - Form analysis

4. **Performance**:
   - Inference time: 30-100ms
   - Smooth visualization
   - Minimal latency

## Bonus Challenges

- [ ] Add pushup counter
- [ ] Implement plank timer with form checking
- [ ] Create yoga pose classifier
- [ ] Add multi-person pose detection
- [ ] Implement gesture recognition
- [ ] Create virtual fitness coach with audio feedback
- [ ] Add pose comparison to ideal form
- [ ] Implement motion heatmaps
- [ ] Create AR filters based on poses
- [ ] Add video recording with pose overlay
- [ ] Build pose-based game controls
- [ ] Create workout routine builder

## Resources

- [TensorFlow.js Pose Detection](https://github.com/tensorflow/tfjs-models/tree/master/pose-detection)
- [MoveNet Documentation](https://www.tensorflow.org/hub/tutorials/movenet)
- [PoseNet Documentation](https://github.com/tensorflow/tfjs-models/tree/master/posenet)
- [Pose Estimation Guide](https://www.tensorflow.org/lite/examples/pose_estimation/overview)
- [MediaPipe Pose](https://google.github.io/mediapipe/solutions/pose.html)

## Success Criteria

- Model loads successfully
- Webcam access works
- 17 keypoints detected accurately
- Skeleton overlay renders correctly
- Real-time performance (>15 FPS)
- Joint angles calculated accurately
- Exercise counter works reliably
- UI is responsive and clear
- Works on desktop and mobile browsers
- No memory leaks during extended use
