# Project 10: Build API with Automatic OpenAPI Documentation

## Overview
Create a production-ready ML API with comprehensive, auto-generated OpenAPI (Swagger) documentation. Learn to document endpoints effectively, add code examples, customize the documentation UI, and generate client SDKs automatically. This project focuses on making your API easy to understand and use.

## Difficulty Level
Intermediate

## Learning Objectives
- Generate automatic OpenAPI documentation
- Customize Swagger UI and ReDoc
- Document request/response schemas
- Add code examples for multiple languages
- Create interactive API documentation
- Generate client SDKs automatically
- Version API documentation
- Add authentication to docs
- Include API changelog and migration guides

## Technical Stack
- **Framework**: FastAPI (built-in OpenAPI support)
- **Documentation**: Swagger UI, ReDoc
- **Schema**: OpenAPI 3.0
- **SDK Generation**: openapi-generator, swagger-codegen
- **Versioning**: API versioning with FastAPI
- **ML Framework**: PyTorch

## Project Requirements

### 1. Documentation Features
- Auto-generated OpenAPI schema
- Interactive API explorer (Swagger UI)
- Alternative documentation (ReDoc)
- Request/response examples
- Error response documentation
- Authentication documentation
- Rate limit documentation

### 2. Customizations
- Custom logo and branding
- Custom color scheme
- Additional metadata (contact, license)
- Tags and grouping
- Deprecation warnings
- External documentation links

### 3. API Versioning
- Version in URL path (/v1, /v2)
- Version in headers
- Backward compatibility
- Migration guides

### 4. SDK Generation
- Python client SDK
- JavaScript/TypeScript SDK
- cURL examples
- HTTP examples

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn pydantic
pip install torch torchvision pillow
pip install markdown2  # For rendering markdown in docs
pip install pytest httpx

# For SDK generation (optional)
npm install -g @openapitools/openapi-generator-cli
```

### Step 2: Project Structure
```bash
mkdir documented_api
cd documented_api

touch main.py models.py config.py
touch api_v1.py api_v2.py
mkdir -p static docs
touch static/logo.png
touch docs/changelog.md docs/migration_guide.md
```

### Step 3: Configuration (config.py)
```python
from pydantic import BaseSettings

class Settings(BaseSettings):
    # API Metadata
    API_TITLE: str = "ML Prediction API"
    API_VERSION: str = "2.0.0"
    API_DESCRIPTION: str = """
## Machine Learning Prediction API

A production-ready API for image classification using deep learning models.

### Features

* 🚀 **Fast Predictions** - Optimized inference with batching
* 🔐 **Secure** - JWT authentication and rate limiting
* 📊 **Multiple Models** - Support for various ML models
* 📱 **Easy to Use** - Comprehensive documentation and SDKs

### Getting Started

1. **Authentication**: Obtain an API key from [your-domain.com/signup](https://your-domain.com/signup)
2. **Make a Request**: Use the interactive docs below to test endpoints
3. **Integrate**: Download client SDKs or use direct HTTP requests

### Support

- 📧 Email: support@your-domain.com
- 💬 Discord: [discord.gg/your-server](https://discord.gg/your-server)
- 📖 Docs: [docs.your-domain.com](https://docs.your-domain.com)
    """

    # Contact Information
    CONTACT_NAME: str = "API Support Team"
    CONTACT_EMAIL: str = "api-support@example.com"
    CONTACT_URL: str = "https://example.com/support"

    # License
    LICENSE_NAME: str = "MIT"
    LICENSE_URL: str = "https://opensource.org/licenses/MIT"

    # Terms of Service
    TERMS_OF_SERVICE: str = "https://example.com/terms"

    # External Docs
    EXTERNAL_DOCS_URL: str = "https://docs.example.com"
    EXTERNAL_DOCS_DESCRIPTION: str = "Full Documentation"

    # Server URLs
    SERVERS: list = [
        {
            "url": "https://api.example.com/v1",
            "description": "Production server"
        },
        {
            "url": "https://staging-api.example.com/v1",
            "description": "Staging server"
        },
        {
            "url": "http://localhost:8000/v1",
            "description": "Development server"
        }
    ]

settings = Settings()
```

### Step 4: Pydantic Models (models.py)
```python
from pydantic import BaseModel, Field, HttpUrl
from typing import List, Optional, Dict, Any
from datetime import datetime
from enum import Enum

class ModelType(str, Enum):
    """Supported model types"""
    CLASSIFICATION = "classification"
    DETECTION = "detection"
    SEGMENTATION = "segmentation"

class PredictionRequest(BaseModel):
    """
    Request schema for making predictions

    Provide either a base64 encoded image or a URL to an image.
    """
    image: Optional[str] = Field(
        None,
        description="Base64 encoded image data",
        example="iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg=="
    )

    image_url: Optional[HttpUrl] = Field(
        None,
        description="URL to image file",
        example="https://example.com/images/dog.jpg"
    )

    model_id: str = Field(
        default="resnet50",
        description="Model ID to use for prediction",
        example="resnet50"
    )

    top_k: int = Field(
        default=5,
        ge=1,
        le=20,
        description="Number of top predictions to return",
        example=5
    )

    confidence_threshold: float = Field(
        default=0.0,
        ge=0.0,
        le=1.0,
        description="Minimum confidence threshold for predictions",
        example=0.1
    )

    class Config:
        schema_extra = {
            "examples": [
                {
                    "image": "base64_encoded_image_data_here",
                    "model_id": "resnet50",
                    "top_k": 5,
                    "confidence_threshold": 0.1
                },
                {
                    "image_url": "https://example.com/images/cat.jpg",
                    "model_id": "mobilenet_v2",
                    "top_k": 3,
                    "confidence_threshold": 0.2
                }
            ]
        }

class Prediction(BaseModel):
    """Individual prediction result"""
    class_name: str = Field(
        ...,
        description="Predicted class name",
        example="golden_retriever"
    )
    confidence: float = Field(
        ...,
        description="Confidence score (0-1)",
        example=0.87,
        ge=0.0,
        le=1.0
    )
    rank: int = Field(
        ...,
        description="Rank of this prediction",
        example=1,
        ge=1
    )

class PredictionResponse(BaseModel):
    """
    Prediction response with results

    Returns the top predictions along with metadata about the request.
    """
    request_id: str = Field(
        ...,
        description="Unique request identifier",
        example="req_abc123xyz"
    )

    model_id: str = Field(
        ...,
        description="Model used for prediction",
        example="resnet50"
    )

    predictions: List[Prediction] = Field(
        ...,
        description="List of predictions ordered by confidence"
    )

    top_prediction: str = Field(
        ...,
        description="Most likely class",
        example="golden_retriever"
    )

    confidence: float = Field(
        ...,
        description="Confidence of top prediction",
        example=0.87
    )

    processing_time: float = Field(
        ...,
        description="Processing time in seconds",
        example=0.045
    )

    timestamp: datetime = Field(
        ...,
        description="Timestamp of prediction",
        example="2025-11-14T10:30:00Z"
    )

    class Config:
        schema_extra = {
            "example": {
                "request_id": "req_abc123xyz",
                "model_id": "resnet50",
                "predictions": [
                    {
                        "class_name": "golden_retriever",
                        "confidence": 0.87,
                        "rank": 1
                    },
                    {
                        "class_name": "labrador",
                        "confidence": 0.09,
                        "rank": 2
                    }
                ],
                "top_prediction": "golden_retriever",
                "confidence": 0.87,
                "processing_time": 0.045,
                "timestamp": "2025-11-14T10:30:00Z"
            }
        }

class ModelInfo(BaseModel):
    """Model information"""
    id: str = Field(..., description="Unique model identifier")
    name: str = Field(..., description="Human-readable model name")
    type: ModelType = Field(..., description="Type of model")
    version: str = Field(..., description="Model version")
    description: Optional[str] = Field(None, description="Model description")
    input_size: List[int] = Field(..., description="Expected input size [channels, height, width]")
    num_classes: int = Field(..., description="Number of output classes")
    accuracy: Optional[float] = Field(None, description="Model accuracy on test set")
    created_at: datetime = Field(..., description="When the model was added")

class ErrorResponse(BaseModel):
    """Error response schema"""
    error_code: str = Field(..., description="Machine-readable error code")
    message: str = Field(..., description="Human-readable error message")
    details: Optional[Dict[str, Any]] = Field(None, description="Additional error details")
    request_id: Optional[str] = Field(None, description="Request identifier for support")

    class Config:
        schema_extra = {
            "example": {
                "error_code": "VALIDATION_ERROR",
                "message": "Invalid input parameters",
                "details": {
                    "field": "image",
                    "reason": "Must provide either image or image_url"
                },
                "request_id": "req_xyz789"
            }
        }
```

### Step 5: API v1 (api_v1.py)
```python
from fastapi import APIRouter, HTTPException, status
from typing import List
import uuid
from datetime import datetime

from models import (
    PredictionRequest,
    PredictionResponse,
    Prediction,
    ModelInfo,
    ModelType,
    ErrorResponse
)

router = APIRouter(
    prefix="/v1",
    tags=["v1"]
)

@router.post(
    "/predict",
    response_model=PredictionResponse,
    status_code=status.HTTP_200_OK,
    summary="Make a prediction",
    description="""
    Make a prediction on an image using a machine learning model.

    ## Request

    Provide either:
    - **image**: Base64 encoded image data, or
    - **image_url**: URL to an image file

    ## Response

    Returns predictions ordered by confidence score.

    ## Example

    ```python
    import requests
    import base64

    with open('image.jpg', 'rb') as f:
        image_data = base64.b64encode(f.read()).decode()

    response = requests.post(
        'https://api.example.com/v1/predict',
        json={
            'image': image_data,
            'model_id': 'resnet50',
            'top_k': 5
        },
        headers={'Authorization': 'Bearer YOUR_API_KEY'}
    )

    print(response.json())
    ```
    """,
    responses={
        200: {
            "description": "Successful prediction",
            "content": {
                "application/json": {
                    "example": {
                        "request_id": "req_abc123",
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
                        "timestamp": "2025-11-14T10:30:00Z"
                    }
                }
            }
        },
        400: {
            "description": "Invalid request",
            "model": ErrorResponse
        },
        422: {
            "description": "Validation error",
            "model": ErrorResponse
        },
        500: {
            "description": "Server error",
            "model": ErrorResponse
        }
    }
)
async def predict(request: PredictionRequest):
    """Make ML prediction"""

    # Dummy prediction
    predictions = [
        Prediction(class_name="golden_retriever", confidence=0.87, rank=1),
        Prediction(class_name="labrador", confidence=0.09, rank=2),
        Prediction(class_name="poodle", confidence=0.02, rank=3),
    ]

    return PredictionResponse(
        request_id=f"req_{uuid.uuid4().hex[:10]}",
        model_id=request.model_id,
        predictions=predictions[:request.top_k],
        top_prediction=predictions[0].class_name,
        confidence=predictions[0].confidence,
        processing_time=0.045,
        timestamp=datetime.utcnow()
    )

@router.get(
    "/models",
    response_model=List[ModelInfo],
    summary="List available models",
    description="Get a list of all available machine learning models",
    tags=["Models"]
)
async def list_models():
    """List all available models"""
    return [
        ModelInfo(
            id="resnet50",
            name="ResNet50",
            type=ModelType.CLASSIFICATION,
            version="1.0.0",
            description="Deep residual network with 50 layers",
            input_size=[3, 224, 224],
            num_classes=1000,
            accuracy=0.76,
            created_at=datetime.utcnow()
        ),
        ModelInfo(
            id="mobilenet_v2",
            name="MobileNetV2",
            type=ModelType.CLASSIFICATION,
            version="2.0.0",
            description="Efficient mobile-optimized model",
            input_size=[3, 224, 224],
            num_classes=1000,
            accuracy=0.72,
            created_at=datetime.utcnow()
        )
    ]

@router.get(
    "/models/{model_id}",
    response_model=ModelInfo,
    summary="Get model details",
    description="Get detailed information about a specific model",
    tags=["Models"],
    responses={
        404: {
            "description": "Model not found",
            "model": ErrorResponse
        }
    }
)
async def get_model(model_id: str):
    """Get specific model information"""
    if model_id == "resnet50":
        return ModelInfo(
            id="resnet50",
            name="ResNet50",
            type=ModelType.CLASSIFICATION,
            version="1.0.0",
            description="Deep residual network with 50 layers",
            input_size=[3, 224, 224],
            num_classes=1000,
            accuracy=0.76,
            created_at=datetime.utcnow()
        )
    else:
        raise HTTPException(
            status_code=404,
            detail=f"Model '{model_id}' not found"
        )
```

### Step 6: Main Application (main.py)
```python
from fastapi import FastAPI
from fastapi.openapi.docs import get_swagger_ui_html, get_redoc_html
from fastapi.openapi.utils import get_openapi
from fastapi.staticfiles import StaticFiles
from fastapi.responses import HTMLResponse, JSONResponse
import markdown2

from config import settings
from api_v1 import router as v1_router

# Custom OpenAPI schema
def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=settings.API_TITLE,
        version=settings.API_VERSION,
        description=settings.API_DESCRIPTION,
        routes=app.routes,
        servers=settings.SERVERS
    )

    # Add contact information
    openapi_schema["info"]["contact"] = {
        "name": settings.CONTACT_NAME,
        "email": settings.CONTACT_EMAIL,
        "url": settings.CONTACT_URL
    }

    # Add license
    openapi_schema["info"]["license"] = {
        "name": settings.LICENSE_NAME,
        "url": settings.LICENSE_URL
    }

    # Add terms of service
    openapi_schema["info"]["termsOfService"] = settings.TERMS_OF_SERVICE

    # Add external documentation
    openapi_schema["externalDocs"] = {
        "description": settings.EXTERNAL_DOCS_DESCRIPTION,
        "url": settings.EXTERNAL_DOCS_URL
    }

    # Add security schemes
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT"
        },
        "ApiKeyAuth": {
            "type": "apiKey",
            "in": "header",
            "name": "X-API-Key"
        }
    }

    # Add tags
    openapi_schema["tags"] = [
        {
            "name": "Predictions",
            "description": "Operations related to making predictions"
        },
        {
            "name": "Models",
            "description": "Operations related to ML models"
        },
        {
            "name": "v1",
            "description": "API version 1 (current)",
            "externalDocs": {
                "description": "V1 Documentation",
                "url": "https://docs.example.com/v1"
            }
        }
    ]

    app.openapi_schema = openapi_schema
    return app.openapi_schema

# Create FastAPI app
app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION,
    description=settings.API_DESCRIPTION,
    docs_url=None,  # We'll customize these
    redoc_url=None,
    openapi_url=None
)

# Include routers
app.include_router(v1_router, tags=["Predictions"])

# Custom OpenAPI endpoint
@app.get("/openapi.json", include_in_schema=False)
async def get_open_api_endpoint():
    return JSONResponse(custom_openapi())

# Custom Swagger UI
@app.get("/docs", include_in_schema=False)
async def custom_swagger_ui_html():
    return get_swagger_ui_html(
        openapi_url="/openapi.json",
        title=f"{settings.API_TITLE} - Swagger UI",
        swagger_favicon_url="https://fastapi.tiangolo.com/img/favicon.png",
        swagger_ui_parameters={
            "defaultModelsExpandDepth": 3,
            "displayRequestDuration": True,
            "filter": True,
            "showExtensions": True,
            "tryItOutEnabled": True
        }
    )

# Custom ReDoc
@app.get("/redoc", include_in_schema=False)
async def redoc_html():
    return get_redoc_html(
        openapi_url="/openapi.json",
        title=f"{settings.API_TITLE} - ReDoc",
        redoc_favicon_url="https://fastapi.tiangolo.com/img/favicon.png"
    )

# Changelog endpoint
@app.get("/changelog", response_class=HTMLResponse, include_in_schema=False)
async def get_changelog():
    """Display API changelog"""
    changelog_md = """
# API Changelog

## Version 2.0.0 (2025-11-14)

### Added
- New batch prediction endpoint
- Support for multiple model types
- Enhanced error handling

### Changed
- Improved prediction accuracy
- Updated response format

### Deprecated
- Legacy v0 endpoints (will be removed in v3.0.0)

## Version 1.0.0 (2025-01-01)

### Added
- Initial release
- Basic prediction endpoint
- Model listing
    """

    html_content = markdown2.markdown(changelog_md)

    return f"""
    <html>
        <head>
            <title>API Changelog</title>
            <style>
                body {{ font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }}
                h1 {{ color: #2c3e50; }}
                h2 {{ color: #3498db; }}
            </style>
        </head>
        <body>
            {html_content}
        </body>
    </html>
    """

@app.get("/", include_in_schema=False)
async def root():
    """API root with links to documentation"""
    return {
        "message": "ML Prediction API",
        "version": settings.API_VERSION,
        "documentation": {
            "swagger": "/docs",
            "redoc": "/redoc",
            "openapi_schema": "/openapi.json",
            "changelog": "/changelog"
        },
        "endpoints": {
            "predict": "/v1/predict",
            "models": "/v1/models"
        }
    }

@app.get("/health", tags=["System"])
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "version": settings.API_VERSION
    }

# Set custom OpenAPI
app.openapi = custom_openapi

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 7: Generate Client SDKs
```bash
# Export OpenAPI schema
curl http://localhost:8000/openapi.json > openapi.json

# Generate Python client
openapi-generator-cli generate \
  -i openapi.json \
  -g python \
  -o ./sdks/python \
  --additional-properties=packageName=ml_api_client

# Generate JavaScript/TypeScript client
openapi-generator-cli generate \
  -i openapi.json \
  -g typescript-axios \
  -o ./sdks/typescript

# Generate cURL examples
openapi-generator-cli generate \
  -i openapi.json \
  -g bash \
  -o ./sdks/bash
```

### Step 8: Example Client Usage
```python
# Using generated Python SDK
from ml_api_client import ApiClient, Configuration, PredictionsApi
from ml_api_client.models import PredictionRequest

# Configure client
config = Configuration()
config.host = "https://api.example.com"
config.api_key['ApiKeyAuth'] = 'your-api-key'

# Create API instance
client = ApiClient(config)
api = PredictionsApi(client)

# Make prediction
request = PredictionRequest(
    image="base64_encoded_image",
    model_id="resnet50",
    top_k=5
)

response = api.v1_predict_post(request)
print(f"Top prediction: {response.top_prediction}")
print(f"Confidence: {response.confidence}")
```

## Expected Outputs

### 1. Interactive Swagger UI
- Available at `http://localhost:8000/docs`
- Try out endpoints directly
- View request/response schemas
- See code examples

### 2. ReDoc Documentation
- Available at `http://localhost:8000/redoc`
- Beautiful, responsive documentation
- Printable format
- Download OpenAPI spec

### 3. OpenAPI Schema
```json
{
  "openapi": "3.0.2",
  "info": {
    "title": "ML Prediction API",
    "version": "2.0.0",
    "description": "Machine Learning Prediction API...",
    "contact": {
      "name": "API Support Team",
      "email": "api-support@example.com"
    }
  },
  "paths": {
    "/v1/predict": {
      "post": {
        "summary": "Make a prediction",
        "operationId": "predict_v1_predict_post",
        "requestBody": {...},
        "responses": {...}
      }
    }
  }
}
```

## Bonus Challenges

- [ ] Add Postman collection generation
- [ ] Create interactive API tutorials
- [ ] Add code snippets for multiple languages
- [ ] Implement API versioning with deprecation warnings
- [ ] Create API playground with examples
- [ ] Add response time metrics to docs
- [ ] Create downloadable SDK packages
- [ ] Add webhook documentation
- [ ] Create API comparison table (v1 vs v2)
- [ ] Add video tutorials embedded in docs
- [ ] Create API status page
- [ ] Add interactive schema explorer

## Resources

- [FastAPI OpenAPI](https://fastapi.tiangolo.com/tutorial/metadata/)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger UI](https://swagger.io/tools/swagger-ui/)
- [ReDoc](https://github.com/Redocly/redoc)
- [OpenAPI Generator](https://openapi-generator.tech/)

## Success Criteria

- [ ] Swagger UI is accessible and functional
- [ ] ReDoc provides clear documentation
- [ ] All endpoints are documented
- [ ] Request/response schemas are accurate
- [ ] Code examples are included
- [ ] Error responses are documented
- [ ] Authentication is explained
- [ ] SDKs can be generated successfully
- [ ] External documentation links work
- [ ] Changelog is accessible
- [ ] API versioning is clear
