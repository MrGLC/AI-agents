# Project 5: Build Real-time Streaming API with WebSockets

## Overview
Create a real-time ML prediction API using WebSockets that supports live video stream processing, continuous predictions, and bidirectional communication. Perfect for applications like live object detection, real-time sentiment analysis, or webcam-based classification.

## Difficulty Level
Advanced

## Learning Objectives
- Implement WebSocket connections in FastAPI
- Handle real-time bidirectional communication
- Process streaming video frames
- Manage WebSocket connection lifecycle
- Implement frame buffering and rate limiting
- Handle multiple concurrent WebSocket connections
- Stream predictions back to clients in real-time

## Technical Stack
- **Framework**: FastAPI with WebSocket support
- **ML Framework**: PyTorch or TensorFlow
- **WebSockets**: fastapi.WebSocket
- **Image Processing**: OpenCV, Pillow
- **Frontend**: HTML5 Canvas + JavaScript
- **Protocol**: WebSocket (ws://)
- **Testing**: websockets library

## Project Requirements

### 1. WebSocket Endpoints
- `WS /ws/predict/stream` - Real-time prediction stream
- `WS /ws/video/analyze` - Video stream analysis
- `WS /ws/monitor` - Monitoring and metrics stream
- `GET /stream/demo` - Demo page with webcam

### 2. Features
- Accept video frames via WebSocket
- Real-time prediction on each frame
- Send predictions back to client
- Support multiple simultaneous connections
- Frame rate control and buffering
- Connection state management

### 3. Prediction Modes
- Single frame predictions
- Object detection with bounding boxes
- Continuous monitoring mode
- Batch frame processing

### 4. Communication Protocol
- JSON message format
- Binary frame support (base64)
- Error handling and reconnection
- Heartbeat/ping-pong for connection health

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn websockets
pip install torch torchvision pillow
pip install opencv-python numpy
pip install pytest pytest-asyncio
```

### Step 2: Project Structure
```bash
mkdir websocket_ml_api
cd websocket_ml_api

touch main.py predictor.py connection_manager.py
touch models.py utils.py config.py
mkdir -p static templates
touch static/webcam.html test_websocket.py
```

### Step 3: Configuration (config.py)
```python
from pydantic import BaseSettings

class Settings(BaseSettings):
    # API Settings
    API_TITLE = "WebSocket ML Streaming API"
    API_VERSION = "1.0.0"

    # WebSocket Settings
    MAX_CONNECTIONS = 100
    PING_INTERVAL = 30  # seconds
    MAX_MESSAGE_SIZE = 10 * 1024 * 1024  # 10MB

    # Prediction Settings
    MAX_FPS = 10  # Maximum frames per second to process
    FRAME_BUFFER_SIZE = 5
    CONFIDENCE_THRESHOLD = 0.5

    # Model Settings
    MODEL_NAME = "resnet50"
    DEVICE = "cuda"  # or "cpu"

settings = Settings()
```

### Step 4: Pydantic Models (models.py)
```python
from pydantic import BaseModel, Field
from typing import List, Dict, Optional, Literal
from datetime import datetime

class FrameMessage(BaseModel):
    """Incoming frame message"""
    type: Literal["frame"] = "frame"
    image: str  # base64 encoded
    timestamp: Optional[float] = None
    frame_id: Optional[int] = None

class PredictionMessage(BaseModel):
    """Outgoing prediction message"""
    type: Literal["prediction"] = "prediction"
    frame_id: Optional[int] = None
    predictions: List[Dict[str, float]]
    top_prediction: str
    confidence: float
    processing_time: float
    timestamp: float

class ErrorMessage(BaseModel):
    """Error message"""
    type: Literal["error"] = "error"
    message: str
    code: Optional[str] = None

class StatusMessage(BaseModel):
    """Status/info message"""
    type: Literal["status"] = "status"
    message: str
    data: Optional[Dict] = {}

class ControlMessage(BaseModel):
    """Control message from client"""
    type: Literal["control"] = "control"
    action: str  # "start", "stop", "pause", "configure"
    params: Optional[Dict] = {}

class BoundingBox(BaseModel):
    """Bounding box for object detection"""
    x: float
    y: float
    width: float
    height: float
    class_name: str
    confidence: float

class DetectionMessage(BaseModel):
    """Object detection result"""
    type: Literal["detection"] = "detection"
    frame_id: Optional[int] = None
    detections: List[BoundingBox]
    processing_time: float
    timestamp: float
```

### Step 5: Real-time Predictor (predictor.py)
```python
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import io
import base64
import time
import asyncio
from typing import Dict, List

from config import settings

class RealtimePredictor:
    """Real-time ML predictor for streaming"""

    def __init__(self):
        self.model = None
        self.device = torch.device(
            settings.DEVICE if torch.cuda.is_available() else 'cpu'
        )
        self.transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])
        # ImageNet classes (simplified)
        self.classes = [f"class_{i}" for i in range(1000)]

    async def load_model(self):
        """Load model asynchronously"""
        if self.model is None:
            print("Loading model...")
            # Run in executor to avoid blocking
            loop = asyncio.get_event_loop()
            self.model = await loop.run_in_executor(
                None,
                lambda: models.resnet50(pretrained=True)
            )
            self.model.to(self.device)
            self.model.eval()
            print(f"Model loaded on {self.device}")

    async def predict_frame(
        self,
        image_base64: str,
        top_k: int = 5
    ) -> Dict:
        """Predict single frame"""
        try:
            # Decode image
            image_data = base64.b64decode(image_base64)
            image = Image.open(io.BytesIO(image_data)).convert('RGB')

            # Preprocess
            img_tensor = self.transform(image).unsqueeze(0).to(self.device)

            # Predict
            start_time = time.time()
            with torch.no_grad():
                outputs = self.model(img_tensor)
                probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

            processing_time = time.time() - start_time

            # Get top K predictions
            top_probs, top_indices = torch.topk(probabilities, top_k)

            predictions = [
                {self.classes[idx]: float(prob)}
                for prob, idx in zip(top_probs, top_indices)
            ]

            return {
                'predictions': predictions,
                'top_prediction': self.classes[top_indices[0]],
                'confidence': float(top_probs[0]),
                'processing_time': processing_time
            }

        except Exception as e:
            raise ValueError(f"Error processing frame: {str(e)}")

# Global predictor instance
predictor = RealtimePredictor()
```

### Step 6: Connection Manager (connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Set
import asyncio
import time
from collections import deque

class ConnectionManager:
    """Manages WebSocket connections"""

    def __init__(self):
        self.active_connections: Dict[str, WebSocket] = {}
        self.connection_metadata: Dict[str, dict] = {}
        self.frame_buffers: Dict[str, deque] = {}

    async def connect(self, websocket: WebSocket, client_id: str):
        """Accept new connection"""
        await websocket.accept()
        self.active_connections[client_id] = websocket
        self.connection_metadata[client_id] = {
            'connected_at': time.time(),
            'frames_processed': 0,
            'last_frame_time': None,
            'status': 'active'
        }
        self.frame_buffers[client_id] = deque(maxlen=5)
        print(f"Client {client_id} connected. Total connections: {len(self.active_connections)}")

    def disconnect(self, client_id: str):
        """Remove connection"""
        if client_id in self.active_connections:
            del self.active_connections[client_id]
        if client_id in self.connection_metadata:
            del self.connection_metadata[client_id]
        if client_id in self.frame_buffers:
            del self.frame_buffers[client_id]
        print(f"Client {client_id} disconnected. Total connections: {len(self.active_connections)}")

    async def send_message(self, client_id: str, message: dict):
        """Send message to specific client"""
        if client_id in self.active_connections:
            try:
                await self.active_connections[client_id].send_json(message)
            except Exception as e:
                print(f"Error sending to {client_id}: {e}")
                self.disconnect(client_id)

    async def broadcast(self, message: dict, exclude: Set[str] = None):
        """Broadcast message to all clients"""
        exclude = exclude or set()
        for client_id in list(self.active_connections.keys()):
            if client_id not in exclude:
                await self.send_message(client_id, message)

    def update_metadata(self, client_id: str, **kwargs):
        """Update client metadata"""
        if client_id in self.connection_metadata:
            self.connection_metadata[client_id].update(kwargs)

    def get_stats(self) -> dict:
        """Get connection statistics"""
        return {
            'total_connections': len(self.active_connections),
            'active_clients': list(self.active_connections.keys()),
            'total_frames_processed': sum(
                meta.get('frames_processed', 0)
                for meta in self.connection_metadata.values()
            )
        }

# Global connection manager
manager = ConnectionManager()
```

### Step 7: Main Application (main.py)
```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
import json
import time
import uuid
import asyncio

from predictor import predictor
from connection_manager import manager
from models import (
    FrameMessage, PredictionMessage, ErrorMessage,
    StatusMessage, ControlMessage
)
from config import settings

app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION
)

# Startup event
@app.on_event("startup")
async def startup_event():
    """Load model on startup"""
    await predictor.load_model()
    print("WebSocket API ready")

@app.get("/")
async def get_home():
    """Home page with info"""
    return {
        "message": "WebSocket ML Streaming API",
        "endpoints": {
            "stream": "ws://localhost:8000/ws/predict/stream",
            "demo": "/demo"
        },
        "connections": manager.get_stats()
    }

@app.get("/demo", response_class=HTMLResponse)
async def get_demo():
    """Demo page with webcam"""
    html_content = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>WebSocket ML Demo</title>
        <style>
            body { font-family: Arial, sans-serif; padding: 20px; }
            #video { border: 2px solid #333; }
            #results { margin-top: 20px; padding: 10px; background: #f0f0f0; }
            .prediction { margin: 5px 0; }
            button { padding: 10px 20px; margin: 5px; font-size: 16px; }
        </style>
    </head>
    <body>
        <h1>Real-time ML Prediction</h1>
        <video id="video" width="640" height="480" autoplay></video>
        <br>
        <button onclick="startStream()">Start</button>
        <button onclick="stopStream()">Stop</button>
        <div id="results">
            <h3>Predictions:</h3>
            <div id="predictions"></div>
        </div>

        <script>
            let ws = null;
            let video = document.getElementById('video');
            let streaming = false;
            let frameInterval = null;

            // Get webcam
            navigator.mediaDevices.getUserMedia({ video: true })
                .then(stream => {
                    video.srcObject = stream;
                });

            function startStream() {
                if (streaming) return;

                // Connect WebSocket
                ws = new WebSocket('ws://localhost:8000/ws/predict/stream');

                ws.onopen = () => {
                    console.log('WebSocket connected');
                    streaming = true;

                    // Send frames every 200ms (5 FPS)
                    frameInterval = setInterval(sendFrame, 200);
                };

                ws.onmessage = (event) => {
                    const data = JSON.parse(event.data);

                    if (data.type === 'prediction') {
                        displayPrediction(data);
                    } else if (data.type === 'error') {
                        console.error('Error:', data.message);
                    }
                };

                ws.onerror = (error) => {
                    console.error('WebSocket error:', error);
                };

                ws.onclose = () => {
                    console.log('WebSocket closed');
                    stopStream();
                };
            }

            function stopStream() {
                streaming = false;
                if (frameInterval) {
                    clearInterval(frameInterval);
                    frameInterval = null;
                }
                if (ws) {
                    ws.close();
                    ws = null;
                }
            }

            function sendFrame() {
                if (!streaming || !ws) return;

                // Capture frame from video
                const canvas = document.createElement('canvas');
                canvas.width = video.videoWidth;
                canvas.height = video.videoHeight;
                const ctx = canvas.getContext('2d');
                ctx.drawImage(video, 0, 0);

                // Convert to base64
                const imageData = canvas.toDataURL('image/jpeg', 0.8);
                const base64 = imageData.split(',')[1];

                // Send to server
                ws.send(JSON.stringify({
                    type: 'frame',
                    image: base64,
                    timestamp: Date.now()
                }));
            }

            function displayPrediction(data) {
                const div = document.getElementById('predictions');
                const html = `
                    <div class="prediction">
                        <strong>${data.top_prediction}</strong>
                        (${(data.confidence * 100).toFixed(1)}%)
                        - ${data.processing_time.toFixed(3)}s
                    </div>
                `;
                div.innerHTML = html;
            }
        </script>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content)

@app.websocket("/ws/predict/stream")
async def websocket_predict_stream(websocket: WebSocket):
    """
    WebSocket endpoint for real-time predictions

    Protocol:
    - Client sends: {"type": "frame", "image": "<base64>"}
    - Server sends: {"type": "prediction", "predictions": [...], ...}
    """
    client_id = str(uuid.uuid4())
    await manager.connect(websocket, client_id)

    # Send welcome message
    await manager.send_message(client_id, {
        'type': 'status',
        'message': 'Connected to ML streaming API',
        'data': {'client_id': client_id}
    })

    try:
        while True:
            # Receive message
            data = await websocket.receive_text()
            message = json.loads(data)

            if message.get('type') == 'frame':
                # Process frame
                try:
                    # Rate limiting - check last frame time
                    metadata = manager.connection_metadata.get(client_id, {})
                    last_time = metadata.get('last_frame_time', 0)
                    current_time = time.time()

                    # Minimum interval between frames
                    min_interval = 1.0 / settings.MAX_FPS
                    if current_time - last_time < min_interval:
                        # Skip frame if too frequent
                        continue

                    # Predict
                    result = await predictor.predict_frame(
                        message.get('image'),
                        top_k=5
                    )

                    # Send prediction back
                    response = PredictionMessage(
                        frame_id=message.get('frame_id'),
                        predictions=result['predictions'],
                        top_prediction=result['top_prediction'],
                        confidence=result['confidence'],
                        processing_time=result['processing_time'],
                        timestamp=current_time
                    )

                    await manager.send_message(
                        client_id,
                        response.dict()
                    )

                    # Update metadata
                    manager.update_metadata(
                        client_id,
                        last_frame_time=current_time,
                        frames_processed=metadata.get('frames_processed', 0) + 1
                    )

                except Exception as e:
                    error_msg = ErrorMessage(
                        message=str(e),
                        code="PREDICTION_ERROR"
                    )
                    await manager.send_message(client_id, error_msg.dict())

            elif message.get('type') == 'control':
                # Handle control messages
                action = message.get('action')
                if action == 'get_stats':
                    stats = manager.get_stats()
                    await manager.send_message(client_id, {
                        'type': 'status',
                        'message': 'Statistics',
                        'data': stats
                    })

    except WebSocketDisconnect:
        manager.disconnect(client_id)
    except Exception as e:
        print(f"Error in WebSocket connection: {e}")
        manager.disconnect(client_id)

@app.websocket("/ws/monitor")
async def websocket_monitor(websocket: WebSocket):
    """
    WebSocket endpoint for monitoring metrics

    Broadcasts statistics every 5 seconds
    """
    client_id = f"monitor_{uuid.uuid4()}"
    await manager.connect(websocket, client_id)

    try:
        while True:
            # Send stats every 5 seconds
            stats = manager.get_stats()
            await manager.send_message(client_id, {
                'type': 'stats',
                'data': stats,
                'timestamp': time.time()
            })
            await asyncio.sleep(5)

    except WebSocketDisconnect:
        manager.disconnect(client_id)

@app.get("/stats")
async def get_stats():
    """Get current connection statistics"""
    return manager.get_stats()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 8: Test WebSocket Client (test_websocket.py)
```python
import asyncio
import websockets
import json
import base64
from PIL import Image
import io

async def test_websocket():
    """Test WebSocket connection"""
    uri = "ws://localhost:8000/ws/predict/stream"

    # Create a test image
    img = Image.new('RGB', (224, 224), color='blue')
    img_byte_arr = io.BytesIO()
    img.save(img_byte_arr, format='JPEG')
    img_base64 = base64.b64encode(img_byte_arr.getvalue()).decode('utf-8')

    async with websockets.connect(uri) as websocket:
        print("Connected to WebSocket")

        # Send 10 frames
        for i in range(10):
            message = {
                'type': 'frame',
                'image': img_base64,
                'frame_id': i
            }
            await websocket.send(json.dumps(message))
            print(f"Sent frame {i}")

            # Receive response
            response = await websocket.recv()
            data = json.loads(response)
            print(f"Received: {data.get('type')} - {data.get('top_prediction', 'N/A')}")

            await asyncio.sleep(0.2)

if __name__ == "__main__":
    asyncio.run(test_websocket())
```

### Step 9: Run the Application
```bash
# Start the server
uvicorn main:app --reload --port 8000

# Open demo page
# Navigate to http://localhost:8000/demo

# Or test with Python client
python test_websocket.py
```

## Expected Outputs

### 1. WebSocket Connection
```
WebSocket connected
Client abc-123 connected. Total connections: 1
```

### 2. Prediction Message (from server)
```json
{
  "type": "prediction",
  "frame_id": 42,
  "predictions": [
    {"golden_retriever": 0.87},
    {"labrador": 0.09}
  ],
  "top_prediction": "golden_retriever",
  "confidence": 0.87,
  "processing_time": 0.023,
  "timestamp": 1699965432.123
}
```

### 3. Status Message
```json
{
  "type": "status",
  "message": "Connected to ML streaming API",
  "data": {
    "client_id": "abc-123-def-456"
  }
}
```

## Bonus Challenges

- [ ] Add object detection with bounding boxes
- [ ] Implement session recording and playback
- [ ] Add multiple model support (client chooses)
- [ ] Create mobile app client
- [ ] Implement frame interpolation for smoother results
- [ ] Add WebRTC for better video streaming
- [ ] Create admin dashboard to monitor all connections
- [ ] Add authentication and per-user limits
- [ ] Implement result caching for similar frames
- [ ] Deploy with WSS (secure WebSocket)
- [ ] Add support for binary WebSocket messages
- [ ] Implement automatic reconnection logic

## Resources

- [FastAPI WebSockets](https://fastapi.tiangolo.com/advanced/websockets/)
- [WebSocket API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [Python websockets Library](https://websockets.readthedocs.io/)
- [Real-time ML Inference](https://pytorch.org/serve/performance_guide.html)
- [WebRTC for Streaming](https://webrtc.org/)

## Success Criteria

- [ ] WebSocket connections establish successfully
- [ ] Frames are processed in real-time
- [ ] Predictions stream back to client
- [ ] Multiple clients can connect simultaneously
- [ ] Frame rate limiting works correctly
- [ ] Graceful connection handling (connect/disconnect)
- [ ] Demo page with webcam works
- [ ] Error handling is robust
- [ ] Low latency (< 100ms per frame)
- [ ] No memory leaks with long-running connections
