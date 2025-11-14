# Project 3: Build GraphQL API for ML Models

## Overview
Create a GraphQL API for machine learning models that allows flexible querying of predictions, model metadata, and performance metrics. GraphQL provides clients with the ability to request exactly the data they need in a single query.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Understand GraphQL schema design
- Implement GraphQL queries and mutations
- Use Strawberry or Graphene for Python GraphQL
- Handle file uploads in GraphQL
- Implement data loaders for efficient batching
- Create subscriptions for real-time updates
- Design type-safe ML API schemas

## Technical Stack
- **Framework**: FastAPI + Strawberry GraphQL
- **ML Framework**: PyTorch or TensorFlow
- **GraphQL**: Strawberry (or Graphene)
- **Database**: SQLite or PostgreSQL (for history)
- **Testing**: pytest with GraphQL test client
- **Frontend**: GraphiQL or Apollo Studio

## Project Requirements

### 1. GraphQL Schema
- **Queries**: Get predictions, model info, prediction history
- **Mutations**: Create predictions, upload models
- **Subscriptions**: Real-time prediction updates
- **Types**: Prediction, Model, Metrics, Image

### 2. Core Features
- Single and batch predictions
- Model metadata queries
- Prediction history with filtering
- Performance metrics aggregation
- Error handling with GraphQL errors

### 3. Advanced Features
- DataLoaders for N+1 query optimization
- Field-level permissions
- Query complexity analysis
- Pagination with cursor-based approach
- File upload support

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn strawberry-graphql
pip install pillow torch torchvision
pip install sqlalchemy databases aiosqlite
pip install pytest httpx
```

### Step 2: Project Structure
```bash
mkdir graphql_ml_api
cd graphql_ml_api

touch main.py schema.py types.py resolvers.py
touch models.py database.py config.py
touch test_graphql.py
```

### Step 3: Define GraphQL Types (types.py)
```python
import strawberry
from typing import List, Optional
from datetime import datetime
from enum import Enum

@strawberry.enum
class ModelType(Enum):
    IMAGE_CLASSIFICATION = "image_classification"
    OBJECT_DETECTION = "object_detection"
    SEGMENTATION = "segmentation"

@strawberry.enum
class PredictionStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"

@strawberry.type
class ClassPrediction:
    """Individual class prediction"""
    class_name: str
    confidence: float
    rank: int

@strawberry.type
class Prediction:
    """Prediction result"""
    id: str
    image_url: Optional[str]
    predictions: List[ClassPrediction]
    top_prediction: str
    confidence: float
    model_version: str
    created_at: datetime
    processing_time: float
    status: PredictionStatus

@strawberry.type
class Model:
    """ML Model metadata"""
    id: str
    name: str
    version: str
    type: ModelType
    input_shape: List[int]
    num_classes: int
    accuracy: Optional[float]
    created_at: datetime
    description: Optional[str]

@strawberry.type
class ModelMetrics:
    """Model performance metrics"""
    model_id: str
    total_predictions: int
    avg_confidence: float
    avg_processing_time: float
    top_classes: List[str]
    predictions_today: int

@strawberry.type
class PredictionHistory:
    """Paginated prediction history"""
    predictions: List[Prediction]
    total_count: int
    has_next_page: bool
    cursor: Optional[str]

@strawberry.input
class PredictionInput:
    """Input for creating a prediction"""
    image_base64: str
    model_id: Optional[str] = None
    top_k: Optional[int] = 5

@strawberry.input
class PredictionFilter:
    """Filter for querying predictions"""
    model_id: Optional[str] = None
    status: Optional[PredictionStatus] = None
    min_confidence: Optional[float] = None
    date_from: Optional[datetime] = None
    date_to: Optional[datetime] = None

@strawberry.type
class Query:
    """Root Query type - will be implemented in resolvers.py"""
    pass

@strawberry.type
class Mutation:
    """Root Mutation type - will be implemented in resolvers.py"""
    pass

@strawberry.type
class Subscription:
    """Root Subscription type - will be implemented in resolvers.py"""
    pass
```

### Step 4: Database Models (models.py)
```python
from sqlalchemy import Column, String, Float, Integer, DateTime, Enum
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
import enum

Base = declarative_base()

class PredictionStatusEnum(enum.Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"

class PredictionModel(Base):
    __tablename__ = "predictions"

    id = Column(String, primary_key=True)
    image_url = Column(String, nullable=True)
    top_prediction = Column(String)
    confidence = Column(Float)
    model_version = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    processing_time = Column(Float)
    status = Column(Enum(PredictionStatusEnum))
    predictions_json = Column(String)  # JSON string of all predictions

class MLModelDB(Base):
    __tablename__ = "models"

    id = Column(String, primary_key=True)
    name = Column(String)
    version = Column(String)
    type = Column(String)
    num_classes = Column(Integer)
    accuracy = Column(Float, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    description = Column(String, nullable=True)
```

### Step 5: Database Setup (database.py)
```python
from databases import Database
from sqlalchemy import create_engine
from models import Base
import os

DATABASE_URL = "sqlite:///./predictions.db"

database = Database(DATABASE_URL)
engine = create_engine(DATABASE_URL)

async def init_db():
    """Initialize database"""
    Base.metadata.create_all(bind=engine)

async def connect_db():
    """Connect to database"""
    await database.connect()

async def disconnect_db():
    """Disconnect from database"""
    await database.disconnect()
```

### Step 6: GraphQL Resolvers (resolvers.py)
```python
import strawberry
from typing import List, Optional, AsyncGenerator
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import io
import base64
import uuid
import json
import time
from datetime import datetime
import asyncio

from types import (
    Prediction, Model, ModelMetrics, ClassPrediction,
    PredictionHistory, PredictionInput, PredictionFilter,
    PredictionStatus, ModelType
)
from database import database

# Load ML model
ml_model = models.resnet50(pretrained=True)
ml_model.eval()

# Dummy ImageNet classes (first 10 for brevity)
IMAGENET_CLASSES = [
    "tench", "goldfish", "great_white_shark", "tiger_shark",
    "hammerhead", "electric_ray", "stingray", "cock", "hen", "ostrich"
    # ... add all 1000 classes
]

async def create_prediction_from_image(
    image_base64: str,
    model_id: Optional[str] = None,
    top_k: int = 5
) -> Prediction:
    """Create prediction from base64 image"""

    # Decode base64 image
    image_data = base64.b64decode(image_base64)
    image = Image.open(io.BytesIO(image_data)).convert('RGB')

    # Preprocess
    preprocess = transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(
            mean=[0.485, 0.456, 0.406],
            std=[0.229, 0.224, 0.225]
        )
    ])

    img_tensor = preprocess(image).unsqueeze(0)

    # Predict
    start_time = time.time()
    with torch.no_grad():
        outputs = ml_model(img_tensor)
        probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

    processing_time = time.time() - start_time

    # Get top K predictions
    top_probs, top_indices = torch.topk(probabilities, top_k)

    predictions_list = [
        ClassPrediction(
            class_name=IMAGENET_CLASSES[idx] if idx < len(IMAGENET_CLASSES) else f"class_{idx}",
            confidence=float(prob),
            rank=i + 1
        )
        for i, (prob, idx) in enumerate(zip(top_probs, top_indices))
    ]

    # Create prediction record
    prediction_id = str(uuid.uuid4())
    prediction = Prediction(
        id=prediction_id,
        image_url=None,
        predictions=predictions_list,
        top_prediction=predictions_list[0].class_name,
        confidence=predictions_list[0].confidence,
        model_version=model_id or "resnet50-v1",
        created_at=datetime.utcnow(),
        processing_time=processing_time,
        status=PredictionStatus.COMPLETED
    )

    # Save to database
    query = """
        INSERT INTO predictions
        (id, top_prediction, confidence, model_version, processing_time, status, predictions_json, created_at)
        VALUES (:id, :top_prediction, :confidence, :model_version, :processing_time, :status, :predictions_json, :created_at)
    """

    values = {
        "id": prediction_id,
        "top_prediction": prediction.top_prediction,
        "confidence": prediction.confidence,
        "model_version": prediction.model_version,
        "processing_time": processing_time,
        "status": "completed",
        "predictions_json": json.dumps([
            {"class": p.class_name, "confidence": p.confidence}
            for p in predictions_list
        ]),
        "created_at": prediction.created_at
    }

    await database.execute(query=query, values=values)

    return prediction

@strawberry.type
class Query:
    @strawberry.field
    async def prediction(self, id: str) -> Optional[Prediction]:
        """Get a single prediction by ID"""
        query = "SELECT * FROM predictions WHERE id = :id"
        result = await database.fetch_one(query=query, values={"id": id})

        if not result:
            return None

        predictions_data = json.loads(result["predictions_json"])
        predictions_list = [
            ClassPrediction(
                class_name=p["class"],
                confidence=p["confidence"],
                rank=i + 1
            )
            for i, p in enumerate(predictions_data)
        ]

        return Prediction(
            id=result["id"],
            image_url=result["image_url"],
            predictions=predictions_list,
            top_prediction=result["top_prediction"],
            confidence=result["confidence"],
            model_version=result["model_version"],
            created_at=result["created_at"],
            processing_time=result["processing_time"],
            status=PredictionStatus.COMPLETED
        )

    @strawberry.field
    async def predictions(
        self,
        filter: Optional[PredictionFilter] = None,
        limit: int = 10,
        offset: int = 0
    ) -> PredictionHistory:
        """Get prediction history with filtering"""
        query = "SELECT * FROM predictions ORDER BY created_at DESC LIMIT :limit OFFSET :offset"
        results = await database.fetch_all(
            query=query,
            values={"limit": limit, "offset": offset}
        )

        predictions_list = []
        for result in results:
            predictions_data = json.loads(result["predictions_json"])
            pred_objects = [
                ClassPrediction(
                    class_name=p["class"],
                    confidence=p["confidence"],
                    rank=i + 1
                )
                for i, p in enumerate(predictions_data)
            ]

            predictions_list.append(
                Prediction(
                    id=result["id"],
                    image_url=result["image_url"],
                    predictions=pred_objects,
                    top_prediction=result["top_prediction"],
                    confidence=result["confidence"],
                    model_version=result["model_version"],
                    created_at=result["created_at"],
                    processing_time=result["processing_time"],
                    status=PredictionStatus.COMPLETED
                )
            )

        # Get total count
        count_query = "SELECT COUNT(*) as count FROM predictions"
        count_result = await database.fetch_one(query=count_query)
        total_count = count_result["count"]

        return PredictionHistory(
            predictions=predictions_list,
            total_count=total_count,
            has_next_page=offset + limit < total_count,
            cursor=str(offset + limit) if offset + limit < total_count else None
        )

    @strawberry.field
    async def model(self, id: str) -> Optional[Model]:
        """Get model information"""
        # For demo, return hardcoded model
        return Model(
            id=id,
            name="ResNet50",
            version="1.0.0",
            type=ModelType.IMAGE_CLASSIFICATION,
            input_shape=[3, 224, 224],
            num_classes=1000,
            accuracy=0.76,
            created_at=datetime.utcnow(),
            description="Pre-trained ResNet50 on ImageNet"
        )

    @strawberry.field
    async def model_metrics(self, model_id: str) -> ModelMetrics:
        """Get model performance metrics"""
        query = """
            SELECT
                COUNT(*) as total,
                AVG(confidence) as avg_confidence,
                AVG(processing_time) as avg_time
            FROM predictions
            WHERE model_version = :model_id
        """
        result = await database.fetch_one(
            query=query,
            values={"model_id": model_id}
        )

        return ModelMetrics(
            model_id=model_id,
            total_predictions=result["total"] or 0,
            avg_confidence=result["avg_confidence"] or 0.0,
            avg_processing_time=result["avg_time"] or 0.0,
            top_classes=["golden_retriever", "labrador", "poodle"],
            predictions_today=0
        )

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_prediction(
        self,
        input: PredictionInput
    ) -> Prediction:
        """Create a new prediction"""
        return await create_prediction_from_image(
            input.image_base64,
            input.model_id,
            input.top_k or 5
        )

    @strawberry.mutation
    async def delete_prediction(self, id: str) -> bool:
        """Delete a prediction"""
        query = "DELETE FROM predictions WHERE id = :id"
        await database.execute(query=query, values={"id": id})
        return True

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def prediction_updates(
        self,
        model_id: Optional[str] = None
    ) -> AsyncGenerator[Prediction, None]:
        """Subscribe to prediction updates"""
        # Simulated real-time updates
        while True:
            await asyncio.sleep(5)
            # In real implementation, this would listen to a message queue
            # and yield new predictions as they come in
            yield Prediction(
                id=str(uuid.uuid4()),
                image_url=None,
                predictions=[],
                top_prediction="sample",
                confidence=0.95,
                model_version=model_id or "resnet50-v1",
                created_at=datetime.utcnow(),
                processing_time=0.5,
                status=PredictionStatus.COMPLETED
            )
```

### Step 7: Create Schema (schema.py)
```python
import strawberry
from resolvers import Query, Mutation, Subscription

schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    subscription=Subscription
)
```

### Step 8: Main Application (main.py)
```python
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
from contextlib import asynccontextmanager

from schema import schema
from database import init_db, connect_db, disconnect_db

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifespan context manager"""
    # Startup
    init_db()
    await connect_db()
    yield
    # Shutdown
    await disconnect_db()

app = FastAPI(lifespan=lifespan)

# Add GraphQL router
graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")

@app.get("/")
async def root():
    return {
        "message": "GraphQL ML API",
        "graphql_endpoint": "/graphql",
        "playground": "/graphql (open in browser)"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 9: Example GraphQL Queries

#### Query: Get Prediction
```graphql
query GetPrediction($id: String!) {
  prediction(id: $id) {
    id
    topPrediction
    confidence
    predictions {
      className
      confidence
      rank
    }
    processingTime
    createdAt
  }
}
```

#### Query: Get Predictions with Pagination
```graphql
query GetPredictions($limit: Int, $offset: Int) {
  predictions(limit: $limit, offset: $offset) {
    predictions {
      id
      topPrediction
      confidence
      createdAt
    }
    totalCount
    hasNextPage
    cursor
  }
}
```

#### Query: Get Model Metrics
```graphql
query GetModelMetrics($modelId: String!) {
  modelMetrics(modelId: $modelId) {
    modelId
    totalPredictions
    avgConfidence
    avgProcessingTime
    topClasses
  }
}
```

#### Mutation: Create Prediction
```graphql
mutation CreatePrediction($input: PredictionInput!) {
  createPrediction(input: $input) {
    id
    topPrediction
    confidence
    predictions {
      className
      confidence
      rank
    }
    status
  }
}
```

#### Subscription: Watch Predictions
```graphql
subscription WatchPredictions($modelId: String) {
  predictionUpdates(modelId: $modelId) {
    id
    topPrediction
    confidence
    createdAt
  }
}
```

### Step 10: Run and Test
```bash
# Start the application
uvicorn main:app --reload

# Open GraphQL Playground
# Navigate to http://localhost:8000/graphql
```

## Expected Outputs

### 1. GraphQL Playground
- Interactive GraphQL IDE at `/graphql`
- Auto-complete for queries
- Schema documentation
- Query validation

### 2. Sample Query Response
```json
{
  "data": {
    "prediction": {
      "id": "abc123",
      "topPrediction": "golden_retriever",
      "confidence": 0.87,
      "predictions": [
        {
          "className": "golden_retriever",
          "confidence": 0.87,
          "rank": 1
        },
        {
          "className": "labrador",
          "confidence": 0.09,
          "rank": 2
        }
      ],
      "processingTime": 0.234,
      "createdAt": "2025-11-14T10:30:00"
    }
  }
}
```

### 3. Mutation Response
```json
{
  "data": {
    "createPrediction": {
      "id": "def456",
      "topPrediction": "cat",
      "confidence": 0.92,
      "status": "COMPLETED"
    }
  }
}
```

## Bonus Challenges

- [ ] Implement DataLoader for batch loading
- [ ] Add authentication with JWT in GraphQL context
- [ ] Create query complexity analysis
- [ ] Implement field-level caching
- [ ] Add rate limiting per client
- [ ] Support file upload via GraphQL multipart
- [ ] Create GraphQL code generator for frontend
- [ ] Add Apollo Federation for microservices
- [ ] Implement persisted queries
- [ ] Add performance monitoring with Apollo Studio

## Resources

- [Strawberry GraphQL Documentation](https://strawberry.rocks/)
- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
- [GraphQL Schema Design](https://www.apollographql.com/docs/apollo-server/schema/schema/)
- [DataLoader Pattern](https://github.com/graphql/dataloader)
- [GraphQL Subscriptions](https://www.apollographql.com/docs/apollo-server/data/subscriptions/)

## Success Criteria

- [ ] GraphQL schema is well-designed and type-safe
- [ ] Queries return correct data
- [ ] Mutations create/update data successfully
- [ ] Subscriptions work in real-time
- [ ] GraphQL playground is accessible
- [ ] Schema documentation is auto-generated
- [ ] Error handling returns GraphQL errors
- [ ] All queries are tested
- [ ] Performance is optimized (no N+1 queries)
