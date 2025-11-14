# Project 2: Build Async Prediction API with Queue System

## Overview
Build an asynchronous ML prediction API that handles long-running model inference using a task queue system. This architecture is essential for computationally expensive models where predictions take several seconds or minutes.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Implement async/await patterns in FastAPI
- Use Celery for background task processing
- Integrate Redis as message broker and result backend
- Design job status tracking system
- Implement polling and webhook patterns
- Handle task failures and retries
- Monitor queue metrics

## Technical Stack
- **Framework**: FastAPI
- **Task Queue**: Celery
- **Message Broker**: Redis
- **ML Framework**: PyTorch or TensorFlow
- **Database**: Redis (for results) or PostgreSQL
- **Monitoring**: Flower (Celery monitoring)
- **Testing**: pytest-asyncio

## Project Requirements

### 1. API Endpoints
- `POST /predict/async` - Submit prediction job
- `GET /predict/status/{job_id}` - Check job status
- `GET /predict/result/{job_id}` - Retrieve prediction result
- `DELETE /predict/cancel/{job_id}` - Cancel running job
- `GET /jobs` - List all jobs with filtering
- `GET /metrics` - Queue metrics and statistics

### 2. Job States
- PENDING - Job submitted, waiting in queue
- PROCESSING - Job currently being processed
- COMPLETED - Job finished successfully
- FAILED - Job failed with error
- CANCELLED - Job cancelled by user

### 3. Features
- Automatic retry on failure (configurable)
- Job expiration (results auto-delete after N hours)
- Priority queues (high/normal/low priority)
- Progress tracking for long jobs
- Webhook notifications on completion

### 4. Error Handling
- Graceful failure with error messages
- Retry logic with exponential backoff
- Dead letter queue for failed jobs

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Install dependencies
pip install fastapi uvicorn celery redis python-multipart
pip install flower  # For monitoring
pip install torch torchvision  # ML framework
pip install pytest pytest-asyncio httpx

# Start Redis (using Docker)
docker run -d -p 6379:6379 redis:latest

# Or install Redis locally
# Linux: sudo apt-get install redis-server
# Mac: brew install redis
```

### Step 2: Project Structure
```bash
mkdir async_prediction_api
cd async_prediction_api

# Create files
touch main.py celery_app.py tasks.py models.py config.py
touch test_async_api.py
mkdir -p models/weights
```

### Step 3: Configuration (config.py)
```python
from pydantic import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    # API Settings
    API_TITLE = "Async Prediction API"
    API_VERSION = "1.0.0"

    # Redis Settings
    REDIS_HOST = "localhost"
    REDIS_PORT = 6379
    REDIS_DB = 0
    REDIS_URL = f"redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}"

    # Celery Settings
    CELERY_BROKER_URL = REDIS_URL
    CELERY_RESULT_BACKEND = REDIS_URL
    CELERY_TASK_TRACK_STARTED = True
    CELERY_TASK_TIME_LIMIT = 600  # 10 minutes max
    CELERY_RESULT_EXPIRES = 3600  # Results expire after 1 hour

    # Model Settings
    MODEL_NAME = "resnet50"
    BATCH_SIZE = 32

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Pydantic Models (models.py)
```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum

class JobStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class Priority(str, Enum):
    HIGH = "high"
    NORMAL = "normal"
    LOW = "low"

class JobSubmitRequest(BaseModel):
    """Request to submit a prediction job"""
    priority: Priority = Priority.NORMAL
    webhook_url: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = {}

class JobSubmitResponse(BaseModel):
    """Response after submitting job"""
    job_id: str
    status: JobStatus
    message: str
    estimated_time: Optional[int] = None  # seconds

class JobStatusResponse(BaseModel):
    """Job status response"""
    job_id: str
    status: JobStatus
    progress: Optional[float] = None  # 0.0 to 1.0
    created_at: datetime
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    error: Optional[str] = None

class PredictionResult(BaseModel):
    """Prediction result"""
    job_id: str
    status: JobStatus
    predictions: Optional[List[Dict[str, float]]] = None
    top_prediction: Optional[str] = None
    confidence: Optional[float] = None
    processing_time: Optional[float] = None
    error: Optional[str] = None

class QueueMetrics(BaseModel):
    """Queue statistics"""
    pending_jobs: int
    processing_jobs: int
    completed_jobs: int
    failed_jobs: int
    total_jobs: int
    avg_processing_time: Optional[float] = None
```

### Step 5: Celery Configuration (celery_app.py)
```python
from celery import Celery
from config import settings

# Initialize Celery
celery_app = Celery(
    "prediction_tasks",
    broker=settings.CELERY_BROKER_URL,
    backend=settings.CELERY_RESULT_BACKEND
)

# Celery configuration
celery_app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,
    task_time_limit=settings.CELERY_TASK_TIME_LIMIT,
    result_expires=settings.CELERY_RESULT_EXPIRES,
    task_routes={
        'tasks.predict_image': {'queue': 'predictions'},
    },
    task_default_priority=5,
    worker_prefetch_multiplier=1,
)
```

### Step 6: Celery Tasks (tasks.py)
```python
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import io
import time
from celery import Task
from celery_app import celery_app

# Load model once at worker startup
model = None

class PredictionTask(Task):
    """Custom task class with model loading"""

    def __init__(self):
        super().__init__()
        self._model = None

    @property
    def model(self):
        if self._model is None:
            print("Loading model...")
            self._model = models.resnet50(pretrained=True)
            self._model.eval()
            print("Model loaded successfully")
        return self._model

@celery_app.task(
    bind=True,
    base=PredictionTask,
    name='tasks.predict_image',
    max_retries=3,
    default_retry_delay=60
)
def predict_image(self, image_data: bytes, task_id: str):
    """
    Perform image prediction task

    Args:
        image_data: Raw image bytes
        task_id: Unique task identifier

    Returns:
        Dictionary with prediction results
    """
    try:
        # Update task state to show progress
        self.update_state(
            state='PROCESSING',
            meta={'progress': 0.0, 'status': 'Loading image...'}
        )

        # Preprocess image
        transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])

        image = Image.open(io.BytesIO(image_data)).convert('RGB')
        img_tensor = transform(image).unsqueeze(0)

        self.update_state(
            state='PROCESSING',
            meta={'progress': 0.3, 'status': 'Running inference...'}
        )

        # Simulate longer processing time for demo
        time.sleep(2)

        # Make prediction
        start_time = time.time()
        with torch.no_grad():
            outputs = self.model(img_tensor)
            probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

        processing_time = time.time() - start_time

        self.update_state(
            state='PROCESSING',
            meta={'progress': 0.8, 'status': 'Formatting results...'}
        )

        # Get top 5 predictions
        top_probs, top_indices = torch.topk(probabilities, 5)

        # Dummy classes (replace with actual ImageNet classes)
        classes = [f"class_{i}" for i in range(1000)]

        predictions = [
            {classes[idx]: float(prob)}
            for prob, idx in zip(top_probs, top_indices)
        ]

        result = {
            'predictions': predictions,
            'top_prediction': classes[top_indices[0]],
            'confidence': float(top_probs[0]),
            'processing_time': processing_time,
            'task_id': task_id
        }

        self.update_state(
            state='COMPLETED',
            meta={'progress': 1.0, 'status': 'Done'}
        )

        return result

    except Exception as e:
        # Retry on failure
        self.update_state(
            state='FAILURE',
            meta={'error': str(e)}
        )
        raise self.retry(exc=e)

@celery_app.task(name='tasks.cleanup_old_results')
def cleanup_old_results():
    """Periodic task to cleanup expired results"""
    # Implement cleanup logic
    pass
```

### Step 7: FastAPI Application (main.py)
```python
from fastapi import FastAPI, File, UploadFile, HTTPException, Query
from fastapi.responses import JSONResponse
from celery.result import AsyncResult
from typing import Optional, List
import redis
from datetime import datetime

from celery_app import celery_app
from tasks import predict_image
from models import (
    JobSubmitRequest, JobSubmitResponse, JobStatusResponse,
    PredictionResult, QueueMetrics, JobStatus, Priority
)
from config import settings

app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION,
    description="Asynchronous ML prediction API with task queue"
)

# Redis client for metadata storage
redis_client = redis.Redis(
    host=settings.REDIS_HOST,
    port=settings.REDIS_PORT,
    db=settings.REDIS_DB,
    decode_responses=True
)

@app.post("/predict/async", response_model=JobSubmitResponse)
async def submit_prediction_job(
    file: UploadFile = File(...),
    priority: Priority = Query(Priority.NORMAL),
    webhook_url: Optional[str] = None
):
    """
    Submit an async prediction job

    Returns a job_id to track the prediction status
    """
    # Validate file
    if file.content_type not in ["image/jpeg", "image/png", "image/jpg"]:
        raise HTTPException(
            status_code=400,
            detail="Invalid file type. Only JPEG and PNG supported."
        )

    # Read file
    image_data = await file.read()

    # Submit task to Celery
    task = predict_image.apply_async(
        args=[image_data, None],
        priority={"high": 9, "normal": 5, "low": 1}[priority]
    )

    # Store job metadata
    job_metadata = {
        'created_at': datetime.utcnow().isoformat(),
        'status': JobStatus.PENDING,
        'priority': priority,
        'webhook_url': webhook_url or '',
        'filename': file.filename
    }

    redis_client.hmset(f"job:{task.id}", job_metadata)
    redis_client.expire(f"job:{task.id}", settings.CELERY_RESULT_EXPIRES)

    return JobSubmitResponse(
        job_id=task.id,
        status=JobStatus.PENDING,
        message="Job submitted successfully",
        estimated_time=10  # Estimate in seconds
    )

@app.get("/predict/status/{job_id}", response_model=JobStatusResponse)
async def get_job_status(job_id: str):
    """Get the status of a prediction job"""
    task = AsyncResult(job_id, app=celery_app)

    # Get metadata from Redis
    metadata = redis_client.hgetall(f"job:{job_id}")

    if not metadata:
        raise HTTPException(
            status_code=404,
            detail=f"Job {job_id} not found"
        )

    # Map Celery states to our JobStatus
    status_map = {
        'PENDING': JobStatus.PENDING,
        'PROCESSING': JobStatus.PROCESSING,
        'SUCCESS': JobStatus.COMPLETED,
        'FAILURE': JobStatus.FAILED,
        'REVOKED': JobStatus.CANCELLED
    }

    status = status_map.get(task.state, JobStatus.PENDING)

    # Extract progress if available
    progress = None
    if task.state == 'PROCESSING' and task.info:
        progress = task.info.get('progress', 0.0)

    return JobStatusResponse(
        job_id=job_id,
        status=status,
        progress=progress,
        created_at=metadata.get('created_at'),
        started_at=None,  # Could track this in metadata
        completed_at=None,
        error=str(task.info) if task.state == 'FAILURE' else None
    )

@app.get("/predict/result/{job_id}", response_model=PredictionResult)
async def get_prediction_result(job_id: str):
    """Get the prediction result for a completed job"""
    task = AsyncResult(job_id, app=celery_app)

    if task.state == 'PENDING':
        raise HTTPException(
            status_code=202,
            detail="Job is still pending"
        )
    elif task.state == 'PROCESSING':
        raise HTTPException(
            status_code=202,
            detail="Job is still processing"
        )
    elif task.state == 'FAILURE':
        return PredictionResult(
            job_id=job_id,
            status=JobStatus.FAILED,
            error=str(task.info)
        )
    elif task.state == 'SUCCESS':
        result = task.result
        return PredictionResult(
            job_id=job_id,
            status=JobStatus.COMPLETED,
            predictions=result.get('predictions'),
            top_prediction=result.get('top_prediction'),
            confidence=result.get('confidence'),
            processing_time=result.get('processing_time')
        )
    else:
        raise HTTPException(
            status_code=404,
            detail=f"Job {job_id} not found"
        )

@app.delete("/predict/cancel/{job_id}")
async def cancel_job(job_id: str):
    """Cancel a pending or running job"""
    task = AsyncResult(job_id, app=celery_app)

    if task.state in ['SUCCESS', 'FAILURE']:
        raise HTTPException(
            status_code=400,
            detail="Cannot cancel completed job"
        )

    task.revoke(terminate=True)

    return {"message": f"Job {job_id} cancelled", "job_id": job_id}

@app.get("/metrics", response_model=QueueMetrics)
async def get_queue_metrics():
    """Get queue metrics and statistics"""
    inspect = celery_app.control.inspect()

    # Get active tasks
    active = inspect.active() or {}
    active_count = sum(len(tasks) for tasks in active.values())

    # Get reserved tasks (pending)
    reserved = inspect.reserved() or {}
    reserved_count = sum(len(tasks) for tasks in reserved.values())

    # This is simplified - you'd want to track these in Redis
    return QueueMetrics(
        pending_jobs=reserved_count,
        processing_jobs=active_count,
        completed_jobs=0,  # Track in Redis
        failed_jobs=0,  # Track in Redis
        total_jobs=0,
        avg_processing_time=None
    )

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    try:
        # Check Redis connection
        redis_client.ping()
        redis_ok = True
    except:
        redis_ok = False

    # Check Celery workers
    inspect = celery_app.control.inspect()
    workers = inspect.active_queues()
    celery_ok = workers is not None and len(workers) > 0

    return {
        "status": "healthy" if (redis_ok and celery_ok) else "unhealthy",
        "redis": "ok" if redis_ok else "error",
        "celery_workers": len(workers) if workers else 0
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 8: Run the Application
```bash
# Terminal 1: Start Redis
redis-server

# Terminal 2: Start Celery worker
celery -A celery_app worker --loglevel=info -Q predictions

# Terminal 3: Start Flower (monitoring)
celery -A celery_app flower --port=5555

# Terminal 4: Start FastAPI
uvicorn main:app --reload --port 8000
```

### Step 9: Test the API
```python
# test_async_api.py
import pytest
import time
from fastapi.testclient import TestClient
from io import BytesIO
from PIL import Image

from main import app

client = TestClient(app)

def create_test_image():
    img = Image.new('RGB', (224, 224), color='blue')
    img_byte_arr = BytesIO()
    img.save(img_byte_arr, format='JPEG')
    img_byte_arr.seek(0)
    return img_byte_arr

def test_submit_job():
    """Test job submission"""
    test_image = create_test_image()
    files = {"file": ("test.jpg", test_image, "image/jpeg")}

    response = client.post("/predict/async", files=files)
    assert response.status_code == 200

    data = response.json()
    assert "job_id" in data
    assert data["status"] == "pending"

    return data["job_id"]

def test_check_status():
    """Test status checking"""
    job_id = test_submit_job()

    response = client.get(f"/predict/status/{job_id}")
    assert response.status_code == 200

    data = response.json()
    assert data["job_id"] == job_id
    assert data["status"] in ["pending", "processing", "completed"]

def test_get_result():
    """Test getting result"""
    job_id = test_submit_job()

    # Wait for job to complete
    time.sleep(5)

    response = client.get(f"/predict/result/{job_id}")
    assert response.status_code in [200, 202]

    if response.status_code == 200:
        data = response.json()
        assert "predictions" in data

def test_cancel_job():
    """Test job cancellation"""
    job_id = test_submit_job()

    response = client.delete(f"/predict/cancel/{job_id}")
    assert response.status_code == 200
```

## Expected Outputs

### 1. Job Submission Response
```json
{
  "job_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01",
  "status": "pending",
  "message": "Job submitted successfully",
  "estimated_time": 10
}
```

### 2. Status Check Response
```json
{
  "job_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01",
  "status": "processing",
  "progress": 0.5,
  "created_at": "2025-11-14T10:00:00",
  "started_at": "2025-11-14T10:00:05",
  "completed_at": null,
  "error": null
}
```

### 3. Result Response
```json
{
  "job_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01",
  "status": "completed",
  "predictions": [
    {"golden_retriever": 0.87},
    {"labrador": 0.09}
  ],
  "top_prediction": "golden_retriever",
  "confidence": 0.87,
  "processing_time": 2.34,
  "error": null
}
```

## Bonus Challenges

- [ ] Add webhook notifications when jobs complete
- [ ] Implement job prioritization with multiple queues
- [ ] Add batch job submission (multiple images at once)
- [ ] Create admin dashboard to monitor all jobs
- [ ] Implement job results pagination
- [ ] Add job scheduling (run at specific time)
- [ ] Create job chains (multi-step ML pipelines)
- [ ] Add authentication and per-user job limits
- [ ] Implement result caching for duplicate requests
- [ ] Deploy with Docker Compose

## Resources

- [Celery Documentation](https://docs.celeryproject.org/)
- [FastAPI Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)
- [Redis Documentation](https://redis.io/documentation)
- [Flower Monitoring](https://flower.readthedocs.io/)
- [Async Patterns in Python](https://realpython.com/async-io-python/)

## Success Criteria

- [ ] Jobs can be submitted and tracked
- [ ] Status updates work correctly
- [ ] Results can be retrieved when ready
- [ ] Jobs can be cancelled
- [ ] Failed jobs retry automatically
- [ ] Celery workers process tasks successfully
- [ ] Flower monitoring dashboard accessible
- [ ] All tests pass
- [ ] System handles multiple concurrent jobs
