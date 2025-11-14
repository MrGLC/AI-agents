# Project 9: Build API with Request Validation and Error Handling

## Overview
Create a robust ML API with comprehensive request validation, custom error handling, structured logging, and detailed error responses. Learn to handle edge cases gracefully, validate complex inputs, and provide helpful error messages that guide users to fix their requests.

## Difficulty Level
Intermediate

## Learning Objectives
- Implement advanced Pydantic validation
- Create custom validators and error messages
- Build centralized error handling
- Implement structured logging
- Handle different error types gracefully
- Create helpful error responses
- Validate file uploads and formats
- Implement request/response logging
- Add error monitoring and alerting

## Technical Stack
- **Framework**: FastAPI with Pydantic V2
- **Validation**: Pydantic validators and custom validators
- **Logging**: Python logging, structlog
- **Error Tracking**: Sentry (optional)
- **Testing**: pytest with error scenarios
- **ML Framework**: PyTorch

## Project Requirements

### 1. Validation Features
- Input data validation (types, ranges, formats)
- File validation (size, format, content)
- Custom business logic validation
- Nested object validation
- Array validation with constraints
- Conditional validation
- Cross-field validation

### 2. Error Types to Handle
- Validation errors (422)
- Not found errors (404)
- Authentication errors (401, 403)
- Rate limit errors (429)
- Server errors (500)
- Model errors (custom)
- File processing errors

### 3. Error Response Format
- Consistent error structure
- Error codes and categories
- Helpful error messages
- Field-specific errors
- Suggested fixes
- Request ID for tracking

### 4. Logging
- Request/response logging
- Error logging with context
- Performance logging
- Structured JSON logs
- Log levels (DEBUG, INFO, WARNING, ERROR)

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn pydantic[email]
pip install python-multipart pillow torch torchvision
pip install structlog python-json-logger
pip install pytest httpx faker
pip install sentry-sdk  # Optional
```

### Step 2: Project Structure
```bash
mkdir validation_api
cd validation_api

touch main.py validators.py error_handlers.py
touch models.py exceptions.py logging_config.py
touch utils.py test_validation.py
```

### Step 3: Custom Exceptions (exceptions.py)
```python
from typing import Optional, Any, Dict

class APIException(Exception):
    """Base API exception"""
    def __init__(
        self,
        message: str,
        error_code: str,
        status_code: int = 500,
        details: Optional[Dict[str, Any]] = None
    ):
        self.message = message
        self.error_code = error_code
        self.status_code = status_code
        self.details = details or {}
        super().__init__(self.message)

class ValidationException(APIException):
    """Validation error"""
    def __init__(self, message: str, details: Optional[Dict] = None):
        super().__init__(
            message=message,
            error_code="VALIDATION_ERROR",
            status_code=422,
            details=details
        )

class ModelNotFoundException(APIException):
    """Model not found"""
    def __init__(self, model_id: str):
        super().__init__(
            message=f"Model '{model_id}' not found",
            error_code="MODEL_NOT_FOUND",
            status_code=404,
            details={"model_id": model_id}
        )

class ModelInferenceException(APIException):
    """Model inference error"""
    def __init__(self, message: str, details: Optional[Dict] = None):
        super().__init__(
            message=message,
            error_code="MODEL_INFERENCE_ERROR",
            status_code=500,
            details=details
        )

class FileProcessingException(APIException):
    """File processing error"""
    def __init__(self, message: str, details: Optional[Dict] = None):
        super().__init__(
            message=message,
            error_code="FILE_PROCESSING_ERROR",
            status_code=400,
            details=details
        )

class InvalidImageException(FileProcessingException):
    """Invalid image file"""
    def __init__(self, reason: str):
        super().__init__(
            message=f"Invalid image: {reason}",
            details={"reason": reason}
        )
```

### Step 4: Pydantic Models with Validation (models.py)
```python
from pydantic import (
    BaseModel, Field, validator, root_validator,
    EmailStr, HttpUrl, constr, conint, confloat
)
from typing import List, Optional, Dict, Any, Literal
from datetime import datetime
from enum import Enum
import base64
import re

class ModelType(str, Enum):
    CLASSIFICATION = "classification"
    DETECTION = "detection"
    SEGMENTATION = "segmentation"

class ImageFormat(str, Enum):
    JPEG = "jpeg"
    PNG = "png"
    WEBP = "webp"

class PredictionRequest(BaseModel):
    """Prediction request with comprehensive validation"""

    # Image data (base64 or URL)
    image: Optional[str] = Field(
        None,
        description="Base64 encoded image",
        min_length=100,
        max_length=10_000_000  # ~7.5MB base64
    )
    image_url: Optional[HttpUrl] = Field(
        None,
        description="URL to image"
    )

    # Model selection
    model_id: str = Field(
        default="resnet50",
        description="Model ID to use for prediction",
        min_length=1,
        max_length=50,
        regex=r'^[a-zA-Z0-9\-_]+$'
    )

    # Prediction parameters
    top_k: conint(ge=1, le=20) = Field(
        default=5,
        description="Number of top predictions to return"
    )

    confidence_threshold: confloat(ge=0.0, le=1.0) = Field(
        default=0.0,
        description="Minimum confidence threshold"
    )

    # Optional metadata
    metadata: Optional[Dict[str, Any]] = Field(
        default=None,
        description="Optional metadata"
    )

    @validator('image')
    def validate_base64_image(cls, v):
        """Validate base64 image format"""
        if v is None:
            return v

        # Check if valid base64
        try:
            # Remove data URL prefix if present
            if ',' in v:
                v = v.split(',', 1)[1]

            # Decode base64
            decoded = base64.b64decode(v, validate=True)

            # Check minimum size
            if len(decoded) < 100:
                raise ValueError("Image too small")

            # Check maximum size (10MB)
            if len(decoded) > 10 * 1024 * 1024:
                raise ValueError("Image too large (max 10MB)")

            return v

        except Exception as e:
            raise ValueError(f"Invalid base64 image: {str(e)}")

    @root_validator
    def validate_image_source(cls, values):
        """Ensure either image or image_url is provided"""
        image = values.get('image')
        image_url = values.get('image_url')

        if not image and not image_url:
            raise ValueError(
                "Either 'image' (base64) or 'image_url' must be provided"
            )

        if image and image_url:
            raise ValueError(
                "Provide only one: 'image' or 'image_url', not both"
            )

        return values

    class Config:
        schema_extra = {
            "example": {
                "image": "base64_encoded_string_here",
                "model_id": "resnet50",
                "top_k": 5,
                "confidence_threshold": 0.1
            }
        }

class BatchPredictionRequest(BaseModel):
    """Batch prediction request"""
    images: List[str] = Field(
        ...,
        min_items=1,
        max_items=100,
        description="List of base64 encoded images"
    )
    model_id: str = "resnet50"
    top_k: conint(ge=1, le=10) = 5

    @validator('images')
    def validate_images(cls, v):
        """Validate all images"""
        for idx, img in enumerate(v):
            try:
                if ',' in img:
                    img = img.split(',', 1)[1]
                base64.b64decode(img, validate=True)
            except Exception:
                raise ValueError(f"Invalid image at index {idx}")
        return v

class Prediction(BaseModel):
    """Single prediction result"""
    class_name: str
    confidence: float = Field(ge=0.0, le=1.0)
    rank: int = Field(ge=1)

class PredictionResponse(BaseModel):
    """Prediction response"""
    request_id: str
    model_id: str
    predictions: List[Prediction]
    top_prediction: str
    confidence: float
    processing_time: float
    timestamp: datetime
    metadata: Optional[Dict] = None

class ErrorDetail(BaseModel):
    """Error detail"""
    field: Optional[str] = None
    message: str
    error_code: str
    suggestion: Optional[str] = None

class ErrorResponse(BaseModel):
    """Standardized error response"""
    request_id: str
    error_code: str
    message: str
    details: List[ErrorDetail] = []
    timestamp: datetime
    path: Optional[str] = None

    class Config:
        schema_extra = {
            "example": {
                "request_id": "req_abc123",
                "error_code": "VALIDATION_ERROR",
                "message": "Request validation failed",
                "details": [
                    {
                        "field": "image",
                        "message": "Invalid base64 encoding",
                        "error_code": "INVALID_BASE64",
                        "suggestion": "Ensure the image is properly base64 encoded"
                    }
                ],
                "timestamp": "2025-11-14T10:00:00",
                "path": "/predict"
            }
        }
```

### Step 5: Custom Validators (validators.py)
```python
from PIL import Image
import io
import base64
from typing import Tuple

from exceptions import InvalidImageException

class ImageValidator:
    """Image validation utilities"""

    @staticmethod
    def validate_image_content(image_data: bytes) -> Tuple[str, Tuple[int, int]]:
        """
        Validate image content and return format and dimensions

        Returns: (format, (width, height))
        Raises: InvalidImageException
        """
        try:
            image = Image.open(io.BytesIO(image_data))
            image.verify()

            # Re-open for format check (verify() closes the file)
            image = Image.open(io.BytesIO(image_data))

            format = image.format.lower() if image.format else 'unknown'
            dimensions = image.size

            # Validate format
            if format not in ['jpeg', 'png', 'webp']:
                raise InvalidImageException(
                    f"Unsupported format: {format}. "
                    "Supported formats: JPEG, PNG, WEBP"
                )

            # Validate dimensions
            width, height = dimensions
            if width < 10 or height < 10:
                raise InvalidImageException(
                    f"Image too small: {width}x{height}. "
                    "Minimum size: 10x10"
                )

            if width > 5000 or height > 5000:
                raise InvalidImageException(
                    f"Image too large: {width}x{height}. "
                    "Maximum size: 5000x5000"
                )

            # Validate aspect ratio
            aspect_ratio = width / height
            if aspect_ratio > 10 or aspect_ratio < 0.1:
                raise InvalidImageException(
                    f"Invalid aspect ratio: {aspect_ratio:.2f}. "
                    "Aspect ratio must be between 0.1 and 10"
                )

            return format, dimensions

        except InvalidImageException:
            raise
        except Exception as e:
            raise InvalidImageException(f"Cannot process image: {str(e)}")

    @staticmethod
    def decode_base64_image(base64_string: str) -> bytes:
        """Decode base64 image with validation"""
        try:
            # Remove data URL prefix if present
            if ',' in base64_string:
                base64_string = base64_string.split(',', 1)[1]

            # Decode
            image_data = base64.b64decode(base64_string, validate=True)

            return image_data

        except Exception as e:
            raise InvalidImageException(f"Invalid base64 encoding: {str(e)}")
```

### Step 6: Error Handlers (error_handlers.py)
```python
from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError
import traceback
import uuid
from datetime import datetime

from exceptions import APIException
from models import ErrorResponse, ErrorDetail
from logging_config import get_logger

logger = get_logger(__name__)

async def api_exception_handler(request: Request, exc: APIException):
    """Handle custom API exceptions"""
    request_id = str(uuid.uuid4())

    # Log error
    logger.error(
        "API exception",
        extra={
            "request_id": request_id,
            "error_code": exc.error_code,
            "message": exc.message,
            "details": exc.details,
            "path": request.url.path
        }
    )

    error_response = ErrorResponse(
        request_id=request_id,
        error_code=exc.error_code,
        message=exc.message,
        details=[
            ErrorDetail(
                message=exc.message,
                error_code=exc.error_code
            )
        ],
        timestamp=datetime.utcnow(),
        path=request.url.path
    )

    return JSONResponse(
        status_code=exc.status_code,
        content=error_response.dict()
    )

async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """Handle Pydantic validation errors"""
    request_id = str(uuid.uuid4())

    # Convert Pydantic errors to our format
    details = []
    for error in exc.errors():
        field = ".".join(str(loc) for loc in error['loc'])
        message = error['msg']
        error_type = error['type']

        # Add helpful suggestions
        suggestion = get_validation_suggestion(error_type, field)

        details.append(
            ErrorDetail(
                field=field,
                message=message,
                error_code=error_type.upper().replace('.', '_'),
                suggestion=suggestion
            )
        )

    # Log validation error
    logger.warning(
        "Validation error",
        extra={
            "request_id": request_id,
            "errors": [d.dict() for d in details],
            "path": request.url.path
        }
    )

    error_response = ErrorResponse(
        request_id=request_id,
        error_code="VALIDATION_ERROR",
        message="Request validation failed",
        details=details,
        timestamp=datetime.utcnow(),
        path=request.url.path
    )

    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content=error_response.dict()
    )

async def general_exception_handler(request: Request, exc: Exception):
    """Handle unexpected exceptions"""
    request_id = str(uuid.uuid4())

    # Log with full traceback
    logger.error(
        "Unexpected error",
        extra={
            "request_id": request_id,
            "error": str(exc),
            "traceback": traceback.format_exc(),
            "path": request.url.path
        }
    )

    error_response = ErrorResponse(
        request_id=request_id,
        error_code="INTERNAL_SERVER_ERROR",
        message="An unexpected error occurred",
        details=[
            ErrorDetail(
                message="Internal server error. Please contact support with request ID.",
                error_code="INTERNAL_ERROR",
                suggestion=f"Contact support with request ID: {request_id}"
            )
        ],
        timestamp=datetime.utcnow(),
        path=request.url.path
    )

    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content=error_response.dict()
    )

def get_validation_suggestion(error_type: str, field: str) -> Optional[str]:
    """Get helpful suggestion based on validation error"""
    suggestions = {
        'value_error.missing': f"Field '{field}' is required. Please provide a value.",
        'type_error.integer': f"Field '{field}' must be an integer.",
        'type_error.float': f"Field '{field}' must be a number.",
        'value_error.url.scheme': "URL must start with http:// or https://",
        'value_error.email': "Please provide a valid email address.",
        'value_error.const': "Value must match the allowed constant.",
    }

    return suggestions.get(error_type, None)

from typing import Optional
```

### Step 7: Logging Configuration (logging_config.py)
```python
import logging
import sys
from pythonjsonlogger import jsonlogger

def setup_logging():
    """Setup structured JSON logging"""
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)

    # JSON formatter
    formatter = jsonlogger.JsonFormatter(
        '%(asctime)s %(name)s %(levelname)s %(message)s',
        timestamp=True
    )

    # Console handler
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger

def get_logger(name: str) -> logging.Logger:
    """Get logger instance"""
    return logging.getLogger(name)

# Setup logging on import
setup_logging()
```

### Step 8: Main Application (main.py)
```python
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from contextlib import asynccontextmanager
import time
import uuid

from models import PredictionRequest, PredictionResponse, Prediction
from validators import ImageValidator
from error_handlers import (
    api_exception_handler,
    validation_exception_handler,
    general_exception_handler
)
from exceptions import APIException, ModelInferenceException
from logging_config import get_logger

logger = get_logger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifespan events"""
    logger.info("Application starting up")
    yield
    logger.info("Application shutting down")

app = FastAPI(
    title="Validated ML API",
    version="1.0.0",
    description="ML API with comprehensive validation and error handling",
    lifespan=lifespan
)

# Register exception handlers
app.add_exception_handler(APIException, api_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(Exception, general_exception_handler)

# Request logging middleware
@app.middleware("http")
async def log_requests(request: Request, call_next):
    """Log all requests"""
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id

    start_time = time.time()

    # Log request
    logger.info(
        "Request started",
        extra={
            "request_id": request_id,
            "method": request.method,
            "path": request.url.path,
            "client": request.client.host if request.client else None
        }
    )

    # Process request
    response = await call_next(request)

    # Log response
    process_time = time.time() - start_time
    logger.info(
        "Request completed",
        extra={
            "request_id": request_id,
            "status_code": response.status_code,
            "process_time": round(process_time, 3)
        }
    )

    # Add request ID to response headers
    response.headers["X-Request-ID"] = request_id

    return response

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: Request, pred_request: PredictionRequest):
    """
    Make prediction with comprehensive validation

    This endpoint demonstrates:
    - Input validation with Pydantic
    - Custom validation logic
    - Structured error responses
    - Request/response logging
    """
    request_id = request.state.request_id

    try:
        # Decode and validate image
        if pred_request.image:
            image_data = ImageValidator.decode_base64_image(pred_request.image)
            format, dimensions = ImageValidator.validate_image_content(image_data)

            logger.info(
                "Image validated",
                extra={
                    "request_id": request_id,
                    "format": format,
                    "dimensions": dimensions
                }
            )

        # Simulate prediction
        start_time = time.time()

        # Dummy predictions
        predictions = [
            Prediction(class_name="golden_retriever", confidence=0.87, rank=1),
            Prediction(class_name="labrador", confidence=0.09, rank=2),
            Prediction(class_name="poodle", confidence=0.02, rank=3),
        ]

        # Filter by confidence threshold
        predictions = [
            p for p in predictions
            if p.confidence >= pred_request.confidence_threshold
        ]

        # Limit to top_k
        predictions = predictions[:pred_request.top_k]

        processing_time = time.time() - start_time

        response = PredictionResponse(
            request_id=request_id,
            model_id=pred_request.model_id,
            predictions=predictions,
            top_prediction=predictions[0].class_name if predictions else "none",
            confidence=predictions[0].confidence if predictions else 0.0,
            processing_time=processing_time,
            timestamp=datetime.utcnow(),
            metadata=pred_request.metadata
        )

        logger.info(
            "Prediction successful",
            extra={
                "request_id": request_id,
                "model_id": pred_request.model_id,
                "top_prediction": response.top_prediction,
                "confidence": response.confidence
            }
        )

        return response

    except Exception as e:
        logger.error(
            "Prediction failed",
            extra={
                "request_id": request_id,
                "error": str(e)
            }
        )
        raise ModelInferenceException(
            message="Failed to generate prediction",
            details={"error": str(e)}
        )

@app.get("/health")
async def health():
    """Health check"""
    return {
        "status": "healthy",
        "version": "1.0.0"
    }

from datetime import datetime

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 9: Test Error Scenarios
```bash
# Test validation error
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{}'

# Test invalid base64
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"image": "invalid-base64"}'

# Test invalid top_k
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"image": "valid-base64", "top_k": 100}'
```

## Expected Outputs

### 1. Validation Error Response
```json
{
  "request_id": "req_abc123",
  "error_code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": [
    {
      "field": "body.image",
      "message": "Either 'image' (base64) or 'image_url' must be provided",
      "error_code": "VALUE_ERROR",
      "suggestion": "Field 'image' is required. Please provide a value."
    }
  ],
  "timestamp": "2025-11-14T10:00:00",
  "path": "/predict"
}
```

### 2. Successful Response
```json
{
  "request_id": "req_xyz789",
  "model_id": "resnet50",
  "predictions": [
    {
      "class_name": "golden_retriever",
      "confidence": 0.87,
      "rank": 1
    }
  ],
  "top_prediction": "golden_retriever",
  "confidence": 0.87,
  "processing_time": 0.045,
  "timestamp": "2025-11-14T10:00:00"
}
```

## Bonus Challenges

- [ ] Add input sanitization for XSS prevention
- [ ] Implement request schema versioning
- [ ] Add custom error pages
- [ ] Create error recovery mechanisms
- [ ] Implement error aggregation and reporting
- [ ] Add error rate monitoring and alerting
- [ ] Create detailed error documentation
- [ ] Add error reproduction tools
- [ ] Implement graceful degradation
- [ ] Add circuit breaker for failing dependencies
- [ ] Create error analytics dashboard
- [ ] Add correlation ID across services

## Resources

- [Pydantic Validation](https://docs.pydantic.dev/latest/usage/validators/)
- [FastAPI Error Handling](https://fastapi.tiangolo.com/tutorial/handling-errors/)
- [Python Logging Best Practices](https://docs.python.org/3/howto/logging.html)
- [Structured Logging](https://www.structlog.org/)
- [HTTP Status Codes](https://httpstatuses.com/)

## Success Criteria

- [ ] Input validation catches all invalid requests
- [ ] Error responses are consistent and helpful
- [ ] All errors are properly logged
- [ ] Request IDs enable tracing
- [ ] Custom validators work correctly
- [ ] Error suggestions are helpful
- [ ] Tests cover all error scenarios
- [ ] Logs are structured (JSON)
- [ ] Performance impact is minimal
- [ ] Documentation is clear
