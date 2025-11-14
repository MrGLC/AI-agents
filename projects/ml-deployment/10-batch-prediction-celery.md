# Project 10: Deploy Batch Prediction Pipeline with Celery

## Overview

Build a scalable batch prediction pipeline using Celery for distributed task processing, Redis as message broker, and PostgreSQL for result storage. This project demonstrates how to handle large-scale ML inference jobs, process millions of predictions asynchronously, and manage long-running ML workflows.

## Learning Objectives

- Implement asynchronous batch processing with Celery
- Design distributed ML inference pipelines
- Handle large datasets efficiently
- Implement task queues and worker pools
- Monitor job progress and status
- Implement retry logic and error handling
- Optimize throughput with parallel processing
- Manage result storage and retrieval
- Implement job scheduling and periodic tasks
- Handle task prioritization and routing

## Difficulty Level

**Advanced** - Requires understanding of distributed systems, message queues, and async processing.

## Technical Stack

- **Task Queue**: Celery 5.3+
- **Message Broker**: Redis
- **Result Backend**: PostgreSQL
- **ML Framework**: scikit-learn, XGBoost
- **Web Framework**: FastAPI
- **Monitoring**: Flower (Celery monitoring)
- **Database**: PostgreSQL
- **Container**: Docker, Docker Compose
- **File Storage**: MinIO (S3-compatible)

## Requirements

### Model Requirements
- Support batch predictions (1K-1M samples)
- Efficient data loading and preprocessing
- Memory-efficient inference
- Support for multiple model types

### Pipeline Requirements
- Asynchronous task processing
- Progress tracking
- Job scheduling
- Result aggregation
- Error handling and retries
- Priority queues

### API Requirements
- POST /jobs - Submit batch job
- GET /jobs/{id} - Get job status
- GET /jobs/{id}/results - Download results
- GET /jobs - List all jobs
- DELETE /jobs/{id} - Cancel job
- GET /workers - Worker status

## Step-by-Step Implementation

### Step 1: Train Model

Create `train_model.py`:

```python
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import joblib
import json
from datetime import datetime
import os

def train_model():
    """Train model for batch predictions"""

    X, y = make_classification(
        n_samples=10000,
        n_features=30,
        n_informative=20,
        n_classes=2,
        random_state=42
    )

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    print("Training model...")
    model = GradientBoostingClassifier(
        n_estimators=100,
        max_depth=5,
        random_state=42
    )
    model.fit(X_train, y_train)

    accuracy = model.score(X_test, y_test)
    print(f"Accuracy: {accuracy:.4f}")

    # Save
    os.makedirs('models', exist_ok=True)
    joblib.dump(model, 'models/batch_model.joblib')

    metadata = {
        'model_type': 'GradientBoostingClassifier',
        'n_features': 30,
        'accuracy': float(accuracy),
        'created_at': datetime.now().isoformat()
    }

    with open('models/metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    print("Model saved")

if __name__ == "__main__":
    train_model()
```

### Step 2: Create Celery Tasks

Create `tasks.py`:

```python
from celery import Celery, Task
from celery.signals import worker_ready
import joblib
import numpy as np
import pandas as pd
import json
import time
import psycopg2
from psycopg2.extras import execute_values
import logging
import os
from typing import List, Dict

logger = logging.getLogger(__name__)

# Configure Celery
app = Celery(
    'batch_prediction',
    broker='redis://redis:6379/0',
    backend='redis://redis:6379/0'
)

app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
    task_routes={
        'tasks.predict_batch': {'queue': 'batch_predictions'},
        'tasks.predict_chunk': {'queue': 'predictions'},
        'tasks.aggregate_results': {'queue': 'aggregation'}
    }
)

# Database configuration
DB_CONFIG = {
    'host': os.environ.get('DB_HOST', 'postgres'),
    'port': 5432,
    'database': os.environ.get('DB_NAME', 'predictions'),
    'user': os.environ.get('DB_USER', 'postgres'),
    'password': os.environ.get('DB_PASSWORD', 'postgres')
}

# Global model cache
model = None

class PredictionTask(Task):
    """Base task with model loading"""

    def __init__(self):
        super().__init__()
        self._model = None

    @property
    def model(self):
        if self._model is None:
            self._model = joblib.load('models/batch_model.joblib')
            logger.info("Model loaded in worker")
        return self._model

@worker_ready.connect
def on_worker_ready(**kwargs):
    """Load model when worker starts"""
    logger.info("Worker ready, loading model...")

def get_db_connection():
    """Get database connection"""
    return psycopg2.connect(**DB_CONFIG)

def init_db():
    """Initialize database schema"""
    conn = get_db_connection()
    with conn.cursor() as cur:
        # Jobs table
        cur.execute("""
            CREATE TABLE IF NOT EXISTS batch_jobs (
                id SERIAL PRIMARY KEY,
                job_id VARCHAR(255) UNIQUE NOT NULL,
                total_samples INTEGER,
                status VARCHAR(50) DEFAULT 'pending',
                progress FLOAT DEFAULT 0.0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                completed_at TIMESTAMP
            )
        """)

        # Predictions table
        cur.execute("""
            CREATE TABLE IF NOT EXISTS predictions (
                id SERIAL PRIMARY KEY,
                job_id VARCHAR(255) NOT NULL,
                sample_id INTEGER,
                prediction INTEGER,
                probability FLOAT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (job_id) REFERENCES batch_jobs(job_id)
            )
        """)

        # Create index
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_job_id ON predictions(job_id)
        """)

        conn.commit()
    conn.close()

@app.task(bind=True, base=PredictionTask)
def predict_chunk(self, job_id: str, chunk_data: List[List[float]], chunk_id: int):
    """Predict a chunk of data"""

    logger.info(f"Processing chunk {chunk_id} for job {job_id}")

    try:
        # Convert to numpy array
        X = np.array(chunk_data)

        # Make predictions
        predictions = self.model.predict(X)
        probabilities = self.model.predict_proba(X)

        # Store results in database
        conn = get_db_connection()
        with conn.cursor() as cur:
            # Prepare data for bulk insert
            values = [
                (job_id, chunk_id * len(chunk_data) + i, int(pred), float(prob[1]))
                for i, (pred, prob) in enumerate(zip(predictions, probabilities))
            ]

            execute_values(
                cur,
                """
                INSERT INTO predictions (job_id, sample_id, prediction, probability)
                VALUES %s
                """,
                values
            )

            conn.commit()
        conn.close()

        logger.info(f"Chunk {chunk_id} completed: {len(chunk_data)} predictions")

        return {
            'chunk_id': chunk_id,
            'samples_processed': len(chunk_data),
            'status': 'completed'
        }

    except Exception as e:
        logger.error(f"Error processing chunk {chunk_id}: {e}")
        raise

@app.task(bind=True)
def predict_batch(self, job_id: str, data: List[List[float]], chunk_size: int = 1000):
    """Orchestrate batch prediction by splitting into chunks"""

    logger.info(f"Starting batch job {job_id} with {len(data)} samples")

    try:
        # Create job record
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                INSERT INTO batch_jobs (job_id, total_samples, status)
                VALUES (%s, %s, %s)
            """, (job_id, len(data), 'processing'))
            conn.commit()
        conn.close()

        # Split data into chunks
        chunks = [
            data[i:i + chunk_size]
            for i in range(0, len(data), chunk_size)
        ]

        logger.info(f"Split into {len(chunks)} chunks")

        # Create subtasks
        from celery import group
        job = group(
            predict_chunk.s(job_id, chunk, chunk_id)
            for chunk_id, chunk in enumerate(chunks)
        )

        # Execute in parallel
        result = job.apply_async()

        # Wait for completion with progress tracking
        total_chunks = len(chunks)
        while not result.ready():
            completed = sum(1 for r in result.results if r.ready())
            progress = (completed / total_chunks) * 100

            # Update progress
            conn = get_db_connection()
            with conn.cursor() as cur:
                cur.execute("""
                    UPDATE batch_jobs
                    SET progress = %s
                    WHERE job_id = %s
                """, (progress, job_id))
                conn.commit()
            conn.close()

            self.update_state(
                state='PROGRESS',
                meta={'progress': progress, 'completed': completed, 'total': total_chunks}
            )

            time.sleep(1)

        # Mark as completed
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                UPDATE batch_jobs
                SET status = %s, progress = %s, completed_at = CURRENT_TIMESTAMP
                WHERE job_id = %s
            """, ('completed', 100.0, job_id))
            conn.commit()
        conn.close()

        logger.info(f"Batch job {job_id} completed")

        return {
            'job_id': job_id,
            'total_samples': len(data),
            'status': 'completed'
        }

    except Exception as e:
        logger.error(f"Error in batch job {job_id}: {e}")

        # Mark as failed
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                UPDATE batch_jobs
                SET status = %s
                WHERE job_id = %s
            """, ('failed', job_id))
            conn.commit()
        conn.close()

        raise

@app.task
def aggregate_results(job_id: str):
    """Aggregate and summarize results"""

    logger.info(f"Aggregating results for job {job_id}")

    conn = get_db_connection()
    with conn.cursor() as cur:
        cur.execute("""
            SELECT
                COUNT(*) as total,
                AVG(prediction) as avg_prediction,
                AVG(probability) as avg_probability,
                COUNT(CASE WHEN prediction = 1 THEN 1 END) as positive_count
            FROM predictions
            WHERE job_id = %s
        """, (job_id,))

        result = cur.fetchone()

    conn.close()

    return {
        'job_id': job_id,
        'total_predictions': result[0],
        'average_prediction': float(result[1]),
        'average_probability': float(result[2]),
        'positive_predictions': result[3]
    }

@app.task
def cleanup_old_jobs(days: int = 7):
    """Clean up old completed jobs"""

    logger.info(f"Cleaning up jobs older than {days} days")

    conn = get_db_connection()
    with conn.cursor() as cur:
        cur.execute("""
            DELETE FROM batch_jobs
            WHERE status = 'completed'
            AND completed_at < NOW() - INTERVAL '%s days'
        """, (days,))
        deleted = cur.rowcount
        conn.commit()
    conn.close()

    logger.info(f"Deleted {deleted} old jobs")
    return {'deleted_jobs': deleted}

# Periodic task configuration
app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'tasks.cleanup_old_jobs',
        'schedule': 86400.0,  # 24 hours
        'args': (7,)
    }
}

if __name__ == "__main__":
    init_db()
```

### Step 3: Create API

Create `app.py`:

```python
from fastapi import FastAPI, HTTPException, UploadFile, File, BackgroundTasks
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field
from typing import List, Optional
import pandas as pd
import io
import json
import uuid
import logging
from datetime import datetime
from tasks import predict_batch, aggregate_results, get_db_connection, init_db
from celery.result import AsyncResult
from tasks import app as celery_app

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Batch Prediction API with Celery",
    description="Scalable batch ML predictions using Celery",
    version="1.0.0"
)

class BatchJobCreate(BaseModel):
    data: List[List[float]]
    chunk_size: int = Field(default=1000, ge=100, le=10000)

class JobStatus(BaseModel):
    job_id: str
    status: str
    progress: float
    total_samples: Optional[int]
    created_at: str
    completed_at: Optional[str]

@app.on_event("startup")
async def startup_event():
    """Initialize database on startup"""
    init_db()
    logger.info("Database initialized")

@app.get("/")
def root():
    """Root endpoint"""
    return {
        "service": "Batch Prediction API",
        "task_queue": "Celery",
        "broker": "Redis"
    }

@app.post("/jobs")
async def create_batch_job(job_request: BatchJobCreate):
    """Submit a new batch prediction job"""

    try:
        # Generate job ID
        job_id = str(uuid.uuid4())

        # Submit to Celery
        task = predict_batch.apply_async(
            args=[job_id, job_request.data, job_request.chunk_size],
            task_id=job_id
        )

        logger.info(f"Created job {job_id} with {len(job_request.data)} samples")

        return {
            'job_id': job_id,
            'status': 'submitted',
            'total_samples': len(job_request.data),
            'message': 'Job submitted successfully'
        }

    except Exception as e:
        logger.error(f"Error creating job: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/jobs/csv")
async def create_job_from_csv(file: UploadFile = File(...)):
    """Submit batch job from CSV file"""

    try:
        # Read CSV
        contents = await file.read()
        df = pd.read_csv(io.BytesIO(contents))

        # Convert to list
        data = df.values.tolist()

        # Generate job ID
        job_id = str(uuid.uuid4())

        # Submit to Celery
        task = predict_batch.apply_async(
            args=[job_id, data, 1000],
            task_id=job_id
        )

        logger.info(f"Created job {job_id} from CSV with {len(data)} samples")

        return {
            'job_id': job_id,
            'status': 'submitted',
            'total_samples': len(data),
            'message': 'Job submitted from CSV'
        }

    except Exception as e:
        logger.error(f"Error processing CSV: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/jobs/{job_id}", response_model=JobStatus)
async def get_job_status(job_id: str):
    """Get job status and progress"""

    try:
        # Get task info from Celery
        task = AsyncResult(job_id, app=celery_app)

        # Get job info from database
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                SELECT job_id, total_samples, status, progress, created_at, completed_at
                FROM batch_jobs
                WHERE job_id = %s
            """, (job_id,))
            result = cur.fetchone()
        conn.close()

        if not result:
            # Check if task exists in Celery
            if task.state == 'PENDING':
                return JobStatus(
                    job_id=job_id,
                    status='pending',
                    progress=0.0,
                    total_samples=None,
                    created_at=datetime.now().isoformat(),
                    completed_at=None
                )
            raise HTTPException(status_code=404, detail="Job not found")

        return JobStatus(
            job_id=result[0],
            status=result[2],
            progress=float(result[3]),
            total_samples=result[1],
            created_at=result[4].isoformat(),
            completed_at=result[5].isoformat() if result[5] else None
        )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error getting job status: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/jobs/{job_id}/results")
async def get_job_results(job_id: str, format: str = "json"):
    """Download job results"""

    try:
        # Check if job is completed
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                SELECT status FROM batch_jobs WHERE job_id = %s
            """, (job_id,))
            result = cur.fetchone()

        if not result:
            raise HTTPException(status_code=404, detail="Job not found")

        if result[0] != 'completed':
            raise HTTPException(status_code=400, detail="Job not completed yet")

        # Get predictions
        with conn.cursor() as cur:
            cur.execute("""
                SELECT sample_id, prediction, probability
                FROM predictions
                WHERE job_id = %s
                ORDER BY sample_id
            """, (job_id,))
            predictions = cur.fetchall()
        conn.close()

        if format == "csv":
            # Return as CSV
            df = pd.DataFrame(
                predictions,
                columns=['sample_id', 'prediction', 'probability']
            )
            stream = io.StringIO()
            df.to_csv(stream, index=False)

            return StreamingResponse(
                iter([stream.getvalue()]),
                media_type="text/csv",
                headers={"Content-Disposition": f"attachment; filename={job_id}.csv"}
            )

        else:
            # Return as JSON
            results = [
                {
                    'sample_id': p[0],
                    'prediction': p[1],
                    'probability': p[2]
                }
                for p in predictions
            ]

            return {
                'job_id': job_id,
                'total_predictions': len(results),
                'results': results
            }

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error getting results: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/jobs/{job_id}/summary")
async def get_job_summary(job_id: str):
    """Get job summary statistics"""

    try:
        # Trigger aggregation task
        task = aggregate_results.delay(job_id)
        result = task.get(timeout=10)

        return result

    except Exception as e:
        logger.error(f"Error getting summary: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/jobs")
async def list_jobs(limit: int = 100):
    """List all jobs"""

    try:
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                SELECT job_id, total_samples, status, progress, created_at
                FROM batch_jobs
                ORDER BY created_at DESC
                LIMIT %s
            """, (limit,))
            jobs = cur.fetchall()
        conn.close()

        return {
            'jobs': [
                {
                    'job_id': j[0],
                    'total_samples': j[1],
                    'status': j[2],
                    'progress': float(j[3]),
                    'created_at': j[4].isoformat()
                }
                for j in jobs
            ],
            'count': len(jobs)
        }

    except Exception as e:
        logger.error(f"Error listing jobs: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/jobs/{job_id}")
async def cancel_job(job_id: str):
    """Cancel a running job"""

    try:
        # Revoke Celery task
        celery_app.control.revoke(job_id, terminate=True)

        # Update database
        conn = get_db_connection()
        with conn.cursor() as cur:
            cur.execute("""
                UPDATE batch_jobs
                SET status = 'cancelled'
                WHERE job_id = %s
            """, (job_id,))
            conn.commit()
        conn.close()

        return {
            'job_id': job_id,
            'status': 'cancelled',
            'message': 'Job cancelled successfully'
        }

    except Exception as e:
        logger.error(f"Error cancelling job: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/workers")
async def get_worker_status():
    """Get Celery worker status"""

    try:
        inspect = celery_app.control.inspect()

        stats = inspect.stats()
        active = inspect.active()
        registered = inspect.registered()

        return {
            'workers': list(stats.keys()) if stats else [],
            'active_tasks': active or {},
            'registered_tasks': registered or {},
            'stats': stats or {}
        }

    except Exception as e:
        logger.error(f"Error getting worker status: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    """Health check"""
    try:
        # Check database
        conn = get_db_connection()
        conn.close()
        db_healthy = True
    except:
        db_healthy = False

    # Check Celery
    try:
        inspect = celery_app.control.inspect()
        workers = inspect.stats()
        celery_healthy = workers is not None and len(workers) > 0
    except:
        celery_healthy = False

    return {
        'status': 'healthy' if db_healthy and celery_healthy else 'unhealthy',
        'database': db_healthy,
        'celery_workers': celery_healthy,
        'timestamp': datetime.now().isoformat()
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 4: Create Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: batch-postgres
    environment:
      POSTGRES_DB: predictions
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: batch-redis
    ports:
      - "6379:6379"
    restart: unless-stopped

  worker:
    build: .
    container_name: celery-worker
    command: celery -A tasks worker --loglevel=info --concurrency=4
    environment:
      - DB_HOST=postgres
      - DB_NAME=predictions
      - DB_USER=postgres
      - DB_PASSWORD=postgres
    depends_on:
      - postgres
      - redis
    volumes:
      - ./models:/app/models
    restart: unless-stopped

  beat:
    build: .
    container_name: celery-beat
    command: celery -A tasks beat --loglevel=info
    environment:
      - DB_HOST=postgres
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  flower:
    build: .
    container_name: celery-flower
    command: celery -A tasks flower --port=5555
    ports:
      - "5555:5555"
    environment:
      - DB_HOST=postgres
    depends_on:
      - redis
    restart: unless-stopped

  api:
    build: .
    container_name: batch-api
    command: uvicorn app:app --host 0.0.0.0 --port 8000
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=postgres
      - DB_NAME=predictions
      - DB_USER=postgres
      - DB_PASSWORD=postgres
    depends_on:
      - postgres
      - redis
    volumes:
      - ./models:/app/models
    restart: unless-stopped

volumes:
  postgres_data:
```

### Step 5: Create Requirements and Dockerfile

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
celery[redis]==5.3.4
flower==2.0.1
scikit-learn==1.3.2
xgboost==2.0.3
numpy==1.24.3
pandas==2.1.4
joblib==1.3.2
psycopg2-binary==2.9.9
redis==5.0.1
python-multipart==0.0.6
```

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY tasks.py .
COPY models/ models/

EXPOSE 8000 5555

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 6: Create Test Script

Create `test_batch.py`:

```python
import requests
import numpy as np
import time
import json

BASE_URL = "http://localhost:8000"

def test_batch_pipeline():
    """Test batch prediction pipeline"""

    print("Testing Batch Prediction Pipeline\n")

    # Create test data
    print("1. Creating batch job with 10,000 samples...")
    data = np.random.rand(10000, 30).tolist()

    response = requests.post(
        f"{BASE_URL}/jobs",
        json={'data': data, 'chunk_size': 1000}
    )
    result = response.json()
    job_id = result['job_id']

    print(f"Job created: {job_id}")
    print(f"Total samples: {result['total_samples']}\n")

    # Monitor progress
    print("2. Monitoring job progress...")
    while True:
        response = requests.get(f"{BASE_URL}/jobs/{job_id}")
        status = response.json()

        print(f"Status: {status['status']}, Progress: {status['progress']:.1f}%")

        if status['status'] == 'completed':
            break
        elif status['status'] == 'failed':
            print("Job failed!")
            return

        time.sleep(2)

    print("\n3. Getting results summary...")
    response = requests.get(f"{BASE_URL}/jobs/{job_id}/summary")
    summary = response.json()

    print(json.dumps(summary, indent=2))

    print("\n4. Downloading results...")
    response = requests.get(f"{BASE_URL}/jobs/{job_id}/results?format=json")
    results = response.json()

    print(f"Total predictions: {results['total_predictions']}")
    print(f"First 5 predictions:")
    for pred in results['results'][:5]:
        print(f"  Sample {pred['sample_id']}: {pred['prediction']} (prob={pred['probability']:.4f})")

    print("\n5. Checking worker status...")
    response = requests.get(f"{BASE_URL}/workers")
    workers = response.json()

    print(f"Active workers: {len(workers['workers'])}")

if __name__ == "__main__":
    test_batch_pipeline()
```

## Expected Outputs

### Job Creation
```json
{
  "job_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "status": "submitted",
  "total_samples": 10000,
  "message": "Job submitted successfully"
}
```

### Job Status
```json
{
  "job_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "status": "processing",
  "progress": 67.5,
  "total_samples": 10000,
  "created_at": "2024-11-14T10:00:00",
  "completed_at": null
}
```

### Results Summary
```json
{
  "job_id": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "total_predictions": 10000,
  "average_prediction": 0.51,
  "average_probability": 0.72,
  "positive_predictions": 5100
}
```

## Bonus Challenges

1. **Priority Queues**: Different priorities for urgent jobs
2. **Result Streaming**: Stream results as they complete
3. **Checkpointing**: Resume failed jobs from checkpoint
4. **Dynamic Scaling**: Auto-scale workers based on queue length
5. **Multi-Model**: Support different models in pipeline
6. **Data Validation**: Validate input data before processing
7. **Cost Tracking**: Track compute costs per job
8. **SLA Monitoring**: Track and alert on SLA violations
9. **Distributed Storage**: Use S3/MinIO for large datasets
10. **Pipeline Orchestration**: Multi-stage ML pipelines

## Resources

- [Celery Documentation](https://docs.celeryq.dev/)
- [Flower Documentation](https://flower.readthedocs.io/)
- [Distributed Task Patterns](https://docs.celeryq.dev/en/stable/userguide/canvas.html)

## Success Criteria

- [ ] Model trains successfully
- [ ] Celery workers start and connect
- [ ] Jobs submit successfully
- [ ] Progress tracking works
- [ ] Chunks process in parallel
- [ ] Results stored in database
- [ ] Results downloadable as JSON/CSV
- [ ] Job cancellation works
- [ ] Flower dashboard accessible
- [ ] Worker status endpoint works
- [ ] Periodic cleanup runs
- [ ] Handle 10K+ samples efficiently
- [ ] Fault tolerance with retries

## Project Structure

```
batch-prediction-celery/
├── models/
│   ├── batch_model.joblib
│   └── metadata.json
├── app.py
├── tasks.py
├── train_model.py
├── test_batch.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── README.md
```
