# Project 4: Build Batch Prediction API with File Uploads

## Overview
Create an API that handles batch predictions for multiple images or data files, with support for ZIP file uploads, CSV processing, and efficient batch inference. This is essential for processing large datasets efficiently.

## Difficulty Level
Intermediate

## Learning Objectives
- Handle multiple file uploads in FastAPI
- Process ZIP archives and extract contents
- Implement efficient batch prediction with batching
- Stream large file responses
- Use async file I/O for better performance
- Implement progress tracking for batch jobs
- Generate downloadable result files (CSV, JSON)

## Technical Stack
- **Framework**: FastAPI
- **ML Framework**: PyTorch or TensorFlow
- **File Handling**: zipfile, aiofiles
- **Data Processing**: pandas, numpy
- **Storage**: Local filesystem or S3
- **Background Jobs**: Optional Celery for large batches

## Project Requirements

### 1. API Endpoints
- `POST /predict/batch` - Upload multiple images
- `POST /predict/batch-zip` - Upload ZIP file of images
- `POST /predict/batch-csv` - Upload CSV with image URLs
- `GET /batch/{batch_id}/status` - Get batch job status
- `GET /batch/{batch_id}/results` - Download results (CSV/JSON)
- `GET /batch/{batch_id}/summary` - Get batch statistics

### 2. File Upload Support
- Multiple file upload (up to 100 images)
- ZIP file upload (auto-extract)
- CSV file with URLs or base64 images
- File validation and size limits
- Supported formats: JPEG, PNG, etc.

### 3. Batch Processing
- Efficient batching for GPU inference
- Progress tracking
- Partial results on failure
- Result caching

### 4. Output Formats
- JSON response
- CSV download
- Excel file (.xlsx)
- ZIP with annotated images (optional)

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn python-multipart
pip install torch torchvision pillow
pip install pandas openpyxl aiofiles
pip install pytest httpx
```

### Step 2: Project Structure
```bash
mkdir batch_prediction_api
cd batch_prediction_api

touch main.py models.py batch_processor.py
touch utils.py config.py test_batch_api.py
mkdir -p uploads results temp
```

### Step 3: Configuration (config.py)
```python
from pathlib import Path

class Settings:
    # API Settings
    API_TITLE = "Batch Prediction API"
    API_VERSION = "1.0.0"

    # File Settings
    MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB per file
    MAX_BATCH_SIZE = 100  # Maximum files per batch
    MAX_ZIP_SIZE = 100 * 1024 * 1024  # 100MB for ZIP
    ALLOWED_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.bmp'}

    # Storage Settings
    UPLOAD_DIR = Path("uploads")
    RESULTS_DIR = Path("results")
    TEMP_DIR = Path("temp")

    # Model Settings
    BATCH_SIZE = 32  # For batched inference
    MODEL_NAME = "resnet50"

    # Processing Settings
    RESULT_EXPIRY_HOURS = 24

settings = Settings()

# Create directories
settings.UPLOAD_DIR.mkdir(exist_ok=True)
settings.RESULTS_DIR.mkdir(exist_ok=True)
settings.TEMP_DIR.mkdir(exist_ok=True)
```

### Step 4: Pydantic Models (models.py)
```python
from pydantic import BaseModel, Field, validator
from typing import List, Dict, Optional
from datetime import datetime
from enum import Enum

class BatchStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    PARTIAL = "partial"

class ImagePrediction(BaseModel):
    """Individual image prediction"""
    filename: str
    top_prediction: str
    confidence: float
    predictions: List[Dict[str, float]]
    processing_time: float
    error: Optional[str] = None

class BatchJobResponse(BaseModel):
    """Response when batch job is submitted"""
    batch_id: str
    status: BatchStatus
    total_images: int
    message: str
    estimated_time: Optional[int] = None

class BatchStatusResponse(BaseModel):
    """Batch job status"""
    batch_id: str
    status: BatchStatus
    total_images: int
    processed_images: int
    successful: int
    failed: int
    progress: float  # 0.0 to 1.0
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None

class BatchResultSummary(BaseModel):
    """Summary of batch results"""
    batch_id: str
    total_images: int
    successful: int
    failed: int
    avg_confidence: float
    avg_processing_time: float
    top_predictions: Dict[str, int]  # class -> count
    created_at: datetime

class BatchResult(BaseModel):
    """Complete batch results"""
    batch_id: str
    summary: BatchResultSummary
    predictions: List[ImagePrediction]
```

### Step 5: Batch Processor (batch_processor.py)
```python
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import asyncio
from pathlib import Path
from typing import List, Dict
import time
import json
from collections import Counter

from models import ImagePrediction, BatchStatus
from config import settings

class BatchPredictor:
    """Handles batch predictions"""

    def __init__(self):
        self.model = None
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])
        # Dummy classes
        self.classes = [f"class_{i}" for i in range(1000)]

    def load_model(self):
        """Load ML model"""
        if self.model is None:
            print("Loading model...")
            self.model = models.resnet50(pretrained=True)
            self.model.to(self.device)
            self.model.eval()
            print(f"Model loaded on {self.device}")

    def preprocess_images(self, image_paths: List[Path]) -> torch.Tensor:
        """Preprocess batch of images"""
        tensors = []
        for img_path in image_paths:
            try:
                image = Image.open(img_path).convert('RGB')
                tensor = self.transform(image)
                tensors.append(tensor)
            except Exception as e:
                print(f"Error preprocessing {img_path}: {e}")
                # Add dummy tensor for failed images
                tensors.append(torch.zeros(3, 224, 224))

        return torch.stack(tensors)

    async def predict_batch(
        self,
        image_paths: List[Path],
        batch_id: str,
        update_callback=None
    ) -> List[ImagePrediction]:
        """Process batch of images"""
        self.load_model()

        results = []
        total = len(image_paths)

        # Process in batches
        for i in range(0, total, settings.BATCH_SIZE):
            batch_paths = image_paths[i:i + settings.BATCH_SIZE]

            # Preprocess batch
            batch_tensor = self.preprocess_images(batch_paths)
            batch_tensor = batch_tensor.to(self.device)

            # Predict
            start_time = time.time()
            with torch.no_grad():
                outputs = self.model(batch_tensor)
                probabilities = torch.nn.functional.softmax(outputs, dim=1)

            processing_time = time.time() - start_time

            # Process each prediction
            for j, (img_path, probs) in enumerate(zip(batch_paths, probabilities)):
                top_probs, top_indices = torch.topk(probs, 5)

                predictions_dict = [
                    {self.classes[idx]: float(prob)}
                    for prob, idx in zip(top_probs, top_indices)
                ]

                result = ImagePrediction(
                    filename=img_path.name,
                    top_prediction=self.classes[top_indices[0]],
                    confidence=float(top_probs[0]),
                    predictions=predictions_dict,
                    processing_time=processing_time / len(batch_paths)
                )
                results.append(result)

            # Update progress
            if update_callback:
                progress = (i + len(batch_paths)) / total
                await update_callback(batch_id, progress, len(results))

            # Allow other tasks to run
            await asyncio.sleep(0)

        return results

    def save_results(
        self,
        batch_id: str,
        predictions: List[ImagePrediction],
        format: str = "json"
    ) -> Path:
        """Save results to file"""
        result_file = settings.RESULTS_DIR / f"{batch_id}.{format}"

        if format == "json":
            with open(result_file, 'w') as f:
                json.dump(
                    [pred.dict() for pred in predictions],
                    f,
                    indent=2
                )
        elif format == "csv":
            import pandas as pd
            df = pd.DataFrame([
                {
                    'filename': pred.filename,
                    'top_prediction': pred.top_prediction,
                    'confidence': pred.confidence,
                    'processing_time': pred.processing_time,
                    'error': pred.error
                }
                for pred in predictions
            ])
            df.to_csv(result_file, index=False)

        return result_file

# Global predictor instance
batch_predictor = BatchPredictor()
```

### Step 6: Utility Functions (utils.py)
```python
import zipfile
import uuid
from pathlib import Path
from typing import List
import aiofiles
import shutil

from config import settings

def generate_batch_id() -> str:
    """Generate unique batch ID"""
    return str(uuid.uuid4())

async def save_upload_file(upload_file, destination: Path):
    """Save uploaded file asynchronously"""
    async with aiofiles.open(destination, 'wb') as f:
        content = await upload_file.read()
        await f.write(content)

def extract_zip(zip_path: Path, extract_to: Path) -> List[Path]:
    """Extract ZIP file and return image paths"""
    image_paths = []

    with zipfile.ZipFile(zip_path, 'r') as zip_ref:
        zip_ref.extractall(extract_to)

    # Find all image files
    for ext in settings.ALLOWED_EXTENSIONS:
        image_paths.extend(extract_to.glob(f"**/*{ext}"))

    return image_paths

def validate_image_file(filename: str) -> bool:
    """Validate image file extension"""
    return Path(filename).suffix.lower() in settings.ALLOWED_EXTENSIONS

def cleanup_batch_files(batch_id: str):
    """Cleanup temporary files for a batch"""
    batch_dir = settings.TEMP_DIR / batch_id
    if batch_dir.exists():
        shutil.rmtree(batch_dir)
```

### Step 7: Main API Application (main.py)
```python
from fastapi import FastAPI, File, UploadFile, HTTPException, BackgroundTasks
from fastapi.responses import FileResponse, StreamingResponse
from typing import List
import json
from pathlib import Path
from datetime import datetime
import asyncio

from models import (
    BatchJobResponse, BatchStatusResponse, BatchResultSummary,
    BatchResult, BatchStatus, ImagePrediction
)
from batch_processor import batch_predictor
from utils import (
    generate_batch_id, save_upload_file, extract_zip,
    validate_image_file, cleanup_batch_files
)
from config import settings

app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION,
    description="Batch prediction API for ML models"
)

# In-memory storage for batch status (use Redis in production)
batch_jobs = {}

async def update_batch_progress(batch_id: str, progress: float, processed: int):
    """Update batch job progress"""
    if batch_id in batch_jobs:
        batch_jobs[batch_id]['progress'] = progress
        batch_jobs[batch_id]['processed'] = processed

async def process_batch_job(batch_id: str, image_paths: List[Path]):
    """Background task to process batch"""
    try:
        batch_jobs[batch_id]['status'] = BatchStatus.PROCESSING
        batch_jobs[batch_id]['started_at'] = datetime.utcnow()

        # Run predictions
        predictions = await batch_predictor.predict_batch(
            image_paths,
            batch_id,
            update_callback=update_batch_progress
        )

        # Save results
        batch_predictor.save_results(batch_id, predictions, format='json')
        batch_predictor.save_results(batch_id, predictions, format='csv')

        # Update status
        successful = len([p for p in predictions if p.error is None])
        failed = len(predictions) - successful

        batch_jobs[batch_id].update({
            'status': BatchStatus.COMPLETED,
            'processed': len(predictions),
            'successful': successful,
            'failed': failed,
            'progress': 1.0,
            'completed_at': datetime.utcnow(),
            'predictions': predictions
        })

    except Exception as e:
        batch_jobs[batch_id]['status'] = BatchStatus.FAILED
        batch_jobs[batch_id]['error'] = str(e)

    finally:
        # Cleanup temp files
        cleanup_batch_files(batch_id)

@app.post("/predict/batch", response_model=BatchJobResponse)
async def predict_batch(
    background_tasks: BackgroundTasks,
    files: List[UploadFile] = File(...)
):
    """
    Upload multiple images for batch prediction

    - **files**: List of image files (JPEG, PNG)
    - Maximum 100 files per batch
    """
    # Validate number of files
    if len(files) > settings.MAX_BATCH_SIZE:
        raise HTTPException(
            status_code=400,
            detail=f"Maximum {settings.MAX_BATCH_SIZE} files allowed per batch"
        )

    # Validate file types
    for file in files:
        if not validate_image_file(file.filename):
            raise HTTPException(
                status_code=400,
                detail=f"Invalid file type: {file.filename}"
            )

    # Generate batch ID
    batch_id = generate_batch_id()
    batch_dir = settings.TEMP_DIR / batch_id
    batch_dir.mkdir(parents=True, exist_ok=True)

    # Save uploaded files
    image_paths = []
    for file in files:
        file_path = batch_dir / file.filename
        await save_upload_file(file, file_path)
        image_paths.append(file_path)

    # Initialize batch job
    batch_jobs[batch_id] = {
        'batch_id': batch_id,
        'status': BatchStatus.PENDING,
        'total': len(files),
        'processed': 0,
        'successful': 0,
        'failed': 0,
        'progress': 0.0,
        'created_at': datetime.utcnow()
    }

    # Start background processing
    background_tasks.add_task(process_batch_job, batch_id, image_paths)

    return BatchJobResponse(
        batch_id=batch_id,
        status=BatchStatus.PENDING,
        total_images=len(files),
        message="Batch job submitted successfully",
        estimated_time=len(files) * 2  # Rough estimate
    )

@app.post("/predict/batch-zip", response_model=BatchJobResponse)
async def predict_batch_zip(
    background_tasks: BackgroundTasks,
    file: UploadFile = File(...)
):
    """
    Upload ZIP file containing images for batch prediction

    - **file**: ZIP file containing images
    """
    if not file.filename.endswith('.zip'):
        raise HTTPException(
            status_code=400,
            detail="File must be a ZIP archive"
        )

    # Generate batch ID
    batch_id = generate_batch_id()
    batch_dir = settings.TEMP_DIR / batch_id
    batch_dir.mkdir(parents=True, exist_ok=True)

    # Save ZIP file
    zip_path = batch_dir / "upload.zip"
    await save_upload_file(file, zip_path)

    # Extract ZIP
    try:
        image_paths = extract_zip(zip_path, batch_dir)
    except Exception as e:
        raise HTTPException(
            status_code=400,
            detail=f"Error extracting ZIP: {str(e)}"
        )

    if len(image_paths) == 0:
        raise HTTPException(
            status_code=400,
            detail="No valid images found in ZIP file"
        )

    if len(image_paths) > settings.MAX_BATCH_SIZE:
        raise HTTPException(
            status_code=400,
            detail=f"ZIP contains too many files. Maximum {settings.MAX_BATCH_SIZE} allowed"
        )

    # Initialize batch job
    batch_jobs[batch_id] = {
        'batch_id': batch_id,
        'status': BatchStatus.PENDING,
        'total': len(image_paths),
        'processed': 0,
        'successful': 0,
        'failed': 0,
        'progress': 0.0,
        'created_at': datetime.utcnow()
    }

    # Start background processing
    background_tasks.add_task(process_batch_job, batch_id, image_paths)

    return BatchJobResponse(
        batch_id=batch_id,
        status=BatchStatus.PENDING,
        total_images=len(image_paths),
        message="ZIP file processed, batch job started",
        estimated_time=len(image_paths) * 2
    )

@app.get("/batch/{batch_id}/status", response_model=BatchStatusResponse)
async def get_batch_status(batch_id: str):
    """Get status of a batch job"""
    if batch_id not in batch_jobs:
        raise HTTPException(
            status_code=404,
            detail=f"Batch job {batch_id} not found"
        )

    job = batch_jobs[batch_id]

    return BatchStatusResponse(
        batch_id=batch_id,
        status=job['status'],
        total_images=job['total'],
        processed_images=job.get('processed', 0),
        successful=job.get('successful', 0),
        failed=job.get('failed', 0),
        progress=job.get('progress', 0.0),
        started_at=job.get('started_at'),
        completed_at=job.get('completed_at')
    )

@app.get("/batch/{batch_id}/results")
async def download_results(
    batch_id: str,
    format: str = "json"
):
    """Download batch results"""
    if batch_id not in batch_jobs:
        raise HTTPException(
            status_code=404,
            detail=f"Batch job {batch_id} not found"
        )

    job = batch_jobs[batch_id]

    if job['status'] != BatchStatus.COMPLETED:
        raise HTTPException(
            status_code=400,
            detail="Batch job not completed yet"
        )

    result_file = settings.RESULTS_DIR / f"{batch_id}.{format}"

    if not result_file.exists():
        raise HTTPException(
            status_code=404,
            detail="Result file not found"
        )

    return FileResponse(
        path=result_file,
        filename=f"batch_results_{batch_id}.{format}",
        media_type="application/octet-stream"
    )

@app.get("/batch/{batch_id}/summary", response_model=BatchResultSummary)
async def get_batch_summary(batch_id: str):
    """Get summary statistics for batch"""
    if batch_id not in batch_jobs:
        raise HTTPException(
            status_code=404,
            detail=f"Batch job {batch_id} not found"
        )

    job = batch_jobs[batch_id]

    if job['status'] != BatchStatus.COMPLETED:
        raise HTTPException(
            status_code=400,
            detail="Batch job not completed yet"
        )

    predictions = job.get('predictions', [])

    # Calculate statistics
    successful_preds = [p for p in predictions if p.error is None]
    avg_confidence = sum(p.confidence for p in successful_preds) / len(successful_preds) if successful_preds else 0
    avg_time = sum(p.processing_time for p in successful_preds) / len(successful_preds) if successful_preds else 0

    # Top predictions count
    from collections import Counter
    top_classes = Counter(p.top_prediction for p in successful_preds)

    return BatchResultSummary(
        batch_id=batch_id,
        total_images=job['total'],
        successful=job['successful'],
        failed=job['failed'],
        avg_confidence=avg_confidence,
        avg_processing_time=avg_time,
        top_predictions=dict(top_classes.most_common(10)),
        created_at=job['created_at']
    )

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "model_loaded": batch_predictor.model is not None,
        "active_batches": len([j for j in batch_jobs.values() if j['status'] == BatchStatus.PROCESSING])
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 8: Test the API
```bash
# Start the server
uvicorn main:app --reload --port 8000

# Test with multiple images
curl -X POST "http://localhost:8000/predict/batch" \
  -F "files=@image1.jpg" \
  -F "files=@image2.jpg" \
  -F "files=@image3.jpg"

# Test with ZIP file
curl -X POST "http://localhost:8000/predict/batch-zip" \
  -F "file=@images.zip"

# Check status
curl "http://localhost:8000/batch/{batch_id}/status"

# Download results
curl "http://localhost:8000/batch/{batch_id}/results?format=csv" -O
```

## Expected Outputs

### 1. Batch Job Response
```json
{
  "batch_id": "abc123-def456",
  "status": "pending",
  "total_images": 50,
  "message": "Batch job submitted successfully",
  "estimated_time": 100
}
```

### 2. Status Response
```json
{
  "batch_id": "abc123-def456",
  "status": "processing",
  "total_images": 50,
  "processed_images": 25,
  "successful": 24,
  "failed": 1,
  "progress": 0.5,
  "started_at": "2025-11-14T10:00:00",
  "completed_at": null
}
```

### 3. Results CSV
```csv
filename,top_prediction,confidence,processing_time,error
image1.jpg,golden_retriever,0.87,0.023,
image2.jpg,cat,0.92,0.021,
image3.jpg,car,0.78,0.024,
```

## Bonus Challenges

- [ ] Add S3 integration for file storage
- [ ] Implement streaming results (Server-Sent Events)
- [ ] Add image preprocessing options (resize, crop)
- [ ] Support video file processing (frame extraction)
- [ ] Create Excel reports with charts
- [ ] Add email notifications on completion
- [ ] Implement job scheduling (process at specific time)
- [ ] Add result caching and deduplication
- [ ] Create ZIP download with annotated images
- [ ] Add GPU usage monitoring

## Resources

- [FastAPI File Uploads](https://fastapi.tiangolo.com/tutorial/request-files/)
- [Python zipfile Documentation](https://docs.python.org/3/library/zipfile.html)
- [aiofiles for Async I/O](https://github.com/Tinche/aiofiles)
- [Pandas CSV Export](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_csv.html)
- [PyTorch Batch Processing](https://pytorch.org/tutorials/beginner/basics/data_tutorial.html)

## Success Criteria

- [ ] Can upload multiple files simultaneously
- [ ] ZIP file extraction works correctly
- [ ] Batch processing is efficient (uses GPU batching)
- [ ] Progress tracking updates in real-time
- [ ] Results downloadable in multiple formats
- [ ] Failed images don't stop entire batch
- [ ] Temporary files cleaned up after processing
- [ ] All tests pass
- [ ] API handles edge cases gracefully
