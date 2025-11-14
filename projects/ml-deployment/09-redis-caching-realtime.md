# Project 09: Deploy Real-Time Prediction Service with Redis Caching

## Overview

Build a high-performance real-time ML prediction service using Redis for caching predictions, feature storage, and rate limiting. This project demonstrates how to optimize latency and throughput for production ML systems serving millions of requests.

## Learning Objectives

- Implement intelligent caching strategies for ML predictions
- Use Redis for feature store and prediction cache
- Optimize cache hit rates and TTL strategies
- Implement rate limiting and request throttling
- Handle cache invalidation correctly
- Monitor cache performance metrics
- Implement request deduplication
- Build distributed caching with Redis Cluster
- Handle cache warming and precomputation
- Implement circuit breakers for cache failures

## Difficulty Level

**Advanced** - Requires understanding of caching strategies, distributed systems, and performance optimization.

## Technical Stack

- **ML Framework**: XGBoost, LightGBM
- **Web Framework**: FastAPI
- **Cache**: Redis, Redis Cluster
- **Monitoring**: Prometheus, Grafana
- **Load Testing**: Locust
- **Container**: Docker, Docker Compose
- **Message Queue**: Redis Streams (optional)

## Requirements

### Model Requirements
- Fast inference (< 50ms)
- Deterministic predictions for caching
- Support for bulk predictions
- Feature preprocessing

### Caching Requirements
- Redis for prediction cache
- TTL-based expiration
- Cache key generation
- Cache hit/miss monitoring
- Cache warming strategies

### API Requirements
- POST /predict - Cached predictions
- POST /predict/batch - Batch predictions
- GET /cache/stats - Cache statistics
- DELETE /cache/clear - Clear cache
- GET /health - Health and cache status

## Step-by-Step Implementation

### Step 1: Train Model

Create `train_model.py`:

```python
import xgboost as xgb
import lightgbm as lgb
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import joblib
import json
from datetime import datetime
import os

def train_models():
    """Train fast models for real-time inference"""

    # Create dataset
    X, y = make_classification(
        n_samples=10000,
        n_features=50,
        n_informative=30,
        n_classes=2,
        random_state=42
    )

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    # Train XGBoost (optimized for speed)
    print("Training XGBoost...")
    xgb_model = xgb.XGBClassifier(
        n_estimators=50,
        max_depth=4,
        learning_rate=0.1,
        tree_method='hist',  # Faster training
        random_state=42
    )
    xgb_model.fit(X_train, y_train)
    xgb_acc = xgb_model.score(X_test, y_test)
    print(f"XGBoost Accuracy: {xgb_acc:.4f}")

    # Train LightGBM (very fast)
    print("Training LightGBM...")
    lgb_model = lgb.LGBMClassifier(
        n_estimators=50,
        max_depth=4,
        learning_rate=0.1,
        random_state=42
    )
    lgb_model.fit(X_train, y_train)
    lgb_acc = lgb_model.score(X_test, y_test)
    print(f"LightGBM Accuracy: {lgb_acc:.4f}")

    # Save models
    os.makedirs('models', exist_ok=True)
    joblib.dump(xgb_model, 'models/xgb_model.joblib')
    joblib.dump(lgb_model, 'models/lgb_model.joblib')

    # Save metadata
    metadata = {
        'n_features': 50,
        'models': {
            'xgboost': {'accuracy': float(xgb_acc)},
            'lightgbm': {'accuracy': float(lgb_acc)}
        },
        'created_at': datetime.now().isoformat()
    }

    with open('models/metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    print("Models saved successfully")

if __name__ == "__main__":
    train_models()
```

### Step 2: Create Cache Manager

Create `cache_manager.py`:

```python
import redis
import hashlib
import json
import time
from typing import Optional, Dict, Any, List
import logging

logger = logging.getLogger(__name__)

class CacheManager:
    """Manage Redis caching for ML predictions"""

    def __init__(self, redis_url: str = "redis://localhost:6379/0"):
        self.redis = redis.from_url(redis_url, decode_responses=True)
        self.redis_binary = redis.from_url(redis_url, decode_responses=False)

        # Cache configuration
        self.default_ttl = 3600  # 1 hour
        self.prediction_prefix = "pred:"
        self.feature_prefix = "feat:"
        self.stats_key = "cache:stats"

        # Initialize stats
        self._init_stats()

    def _init_stats(self):
        """Initialize cache statistics"""
        if not self.redis.exists(self.stats_key):
            self.redis.hset(self.stats_key, mapping={
                'hits': 0,
                'misses': 0,
                'total_requests': 0
            })

    def generate_key(self, features: List[float], model_id: str = "default") -> str:
        """Generate cache key from features"""
        # Create deterministic hash from features
        feature_str = ','.join(f"{f:.6f}" for f in features)
        hash_obj = hashlib.sha256(f"{model_id}:{feature_str}".encode())
        return f"{self.prediction_prefix}{hash_obj.hexdigest()}"

    def get_prediction(self, cache_key: str) -> Optional[Dict]:
        """Get prediction from cache"""
        try:
            cached = self.redis.get(cache_key)

            # Update stats
            pipe = self.redis.pipeline()
            pipe.hincrby(self.stats_key, 'total_requests', 1)

            if cached:
                pipe.hincrby(self.stats_key, 'hits', 1)
                pipe.execute()
                return json.loads(cached)
            else:
                pipe.hincrby(self.stats_key, 'misses', 1)
                pipe.execute()
                return None

        except Exception as e:
            logger.error(f"Cache get error: {e}")
            return None

    def set_prediction(
        self,
        cache_key: str,
        prediction: Dict,
        ttl: Optional[int] = None
    ):
        """Store prediction in cache"""
        try:
            ttl = ttl or self.default_ttl
            self.redis.setex(
                cache_key,
                ttl,
                json.dumps(prediction)
            )
        except Exception as e:
            logger.error(f"Cache set error: {e}")

    def get_batch(self, cache_keys: List[str]) -> Dict[str, Optional[Dict]]:
        """Get multiple predictions at once"""
        try:
            pipeline = self.redis.pipeline()
            for key in cache_keys:
                pipeline.get(key)

            results = pipeline.execute()

            # Update stats
            hits = sum(1 for r in results if r is not None)
            misses = len(results) - hits

            self.redis.hincrby(self.stats_key, 'total_requests', len(results))
            self.redis.hincrby(self.stats_key, 'hits', hits)
            self.redis.hincrby(self.stats_key, 'misses', misses)

            return {
                key: json.loads(result) if result else None
                for key, result in zip(cache_keys, results)
            }

        except Exception as e:
            logger.error(f"Batch cache get error: {e}")
            return {key: None for key in cache_keys}

    def set_batch(self, predictions: Dict[str, Dict], ttl: Optional[int] = None):
        """Store multiple predictions"""
        try:
            ttl = ttl or self.default_ttl
            pipeline = self.redis.pipeline()

            for key, prediction in predictions.items():
                pipeline.setex(key, ttl, json.dumps(prediction))

            pipeline.execute()

        except Exception as e:
            logger.error(f"Batch cache set error: {e}")

    def invalidate_pattern(self, pattern: str):
        """Invalidate cache keys matching pattern"""
        try:
            cursor = 0
            while True:
                cursor, keys = self.redis.scan(cursor, match=pattern, count=100)
                if keys:
                    self.redis.delete(*keys)
                if cursor == 0:
                    break
        except Exception as e:
            logger.error(f"Cache invalidate error: {e}")

    def clear_all(self):
        """Clear all prediction cache"""
        self.invalidate_pattern(f"{self.prediction_prefix}*")

    def get_stats(self) -> Dict:
        """Get cache statistics"""
        try:
            stats = self.redis.hgetall(self.stats_key)
            stats = {k: int(v) for k, v in stats.items()}

            total = stats.get('total_requests', 0)
            hits = stats.get('hits', 0)

            hit_rate = (hits / total * 100) if total > 0 else 0

            return {
                **stats,
                'hit_rate': round(hit_rate, 2),
                'cache_size': self.get_cache_size()
            }

        except Exception as e:
            logger.error(f"Stats error: {e}")
            return {}

    def get_cache_size(self) -> int:
        """Get number of cached predictions"""
        try:
            cursor = 0
            count = 0
            while True:
                cursor, keys = self.redis.scan(
                    cursor,
                    match=f"{self.prediction_prefix}*",
                    count=1000
                )
                count += len(keys)
                if cursor == 0:
                    break
            return count
        except:
            return 0

    def check_rate_limit(
        self,
        user_id: str,
        max_requests: int = 100,
        window: int = 60
    ) -> bool:
        """Check if user is within rate limit"""
        key = f"rate_limit:{user_id}"

        try:
            current = self.redis.get(key)

            if current is None:
                # First request in window
                self.redis.setex(key, window, 1)
                return True

            if int(current) >= max_requests:
                return False

            # Increment counter
            self.redis.incr(key)
            return True

        except Exception as e:
            logger.error(f"Rate limit error: {e}")
            return True  # Allow on error

class FeatureStore:
    """Store and retrieve features in Redis"""

    def __init__(self, redis_url: str = "redis://localhost:6379/0"):
        self.redis = redis.from_url(redis_url, decode_responses=True)
        self.prefix = "features:"
        self.ttl = 86400  # 24 hours

    def set_features(self, entity_id: str, features: Dict):
        """Store features for entity"""
        key = f"{self.prefix}{entity_id}"
        self.redis.setex(key, self.ttl, json.dumps(features))

    def get_features(self, entity_id: str) -> Optional[Dict]:
        """Retrieve features for entity"""
        key = f"{self.prefix}{entity_id}"
        data = self.redis.get(key)
        return json.loads(data) if data else None

    def batch_get_features(self, entity_ids: List[str]) -> Dict[str, Optional[Dict]]:
        """Get features for multiple entities"""
        pipeline = self.redis.pipeline()
        keys = [f"{self.prefix}{eid}" for eid in entity_ids]

        for key in keys:
            pipeline.get(key)

        results = pipeline.execute()

        return {
            eid: json.loads(result) if result else None
            for eid, result in zip(entity_ids, results)
        }
```

### Step 3: Create API with Caching

Create `app.py`:

```python
from fastapi import FastAPI, HTTPException, Header
from pydantic import BaseModel, Field
from typing import List, Optional
import joblib
import numpy as np
import json
import logging
import time
from datetime import datetime
from cache_manager import CacheManager, FeatureStore
import os

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Real-Time ML API with Redis Caching",
    description="High-performance ML API with intelligent caching",
    version="1.0.0"
)

# Initialize cache
REDIS_URL = os.environ.get('REDIS_URL', 'redis://localhost:6379/0')
cache = CacheManager(REDIS_URL)
feature_store = FeatureStore(REDIS_URL)

# Load models
model = None
metadata = None

class PredictionInput(BaseModel):
    features: List[float] = Field(..., min_items=50, max_items=50)
    user_id: Optional[str] = None
    use_cache: bool = True

class BatchPredictionInput(BaseModel):
    instances: List[List[float]]
    use_cache: bool = True

class PredictionResponse(BaseModel):
    prediction: int
    probability: float
    cached: bool
    latency_ms: float
    timestamp: str

@app.on_event("startup")
async def load_model():
    """Load model on startup"""
    global model, metadata

    try:
        model = joblib.load('models/lgb_model.joblib')

        with open('models/metadata.json', 'r') as f:
            metadata = json.load(f)

        logger.info("Model loaded successfully")

    except Exception as e:
        logger.error(f"Failed to load model: {e}")
        raise

@app.get("/")
def root():
    """Root endpoint"""
    return {
        "service": "Real-Time ML API with Caching",
        "cache": "Redis",
        "model": "LightGBM"
    }

@app.post("/predict", response_model=PredictionResponse)
async def predict(
    input_data: PredictionInput,
    x_api_key: Optional[str] = Header(None)
):
    """Make prediction with caching"""

    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    # Rate limiting
    if input_data.user_id and not cache.check_rate_limit(input_data.user_id):
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    start_time = time.time()
    cached = False

    try:
        # Check cache if enabled
        if input_data.use_cache:
            cache_key = cache.generate_key(input_data.features)
            cached_result = cache.get_prediction(cache_key)

            if cached_result:
                cached = True
                latency_ms = (time.time() - start_time) * 1000

                return PredictionResponse(
                    prediction=cached_result['prediction'],
                    probability=cached_result['probability'],
                    cached=True,
                    latency_ms=round(latency_ms, 2),
                    timestamp=datetime.now().isoformat()
                )

        # Make prediction
        features = np.array(input_data.features).reshape(1, -1)
        prediction = int(model.predict(features)[0])
        probability = float(model.predict_proba(features)[0][1])

        latency_ms = (time.time() - start_time) * 1000

        # Store in cache
        if input_data.use_cache:
            prediction_data = {
                'prediction': prediction,
                'probability': probability
            }
            cache.set_prediction(cache_key, prediction_data)

        return PredictionResponse(
            prediction=prediction,
            probability=probability,
            cached=False,
            latency_ms=round(latency_ms, 2),
            timestamp=datetime.now().isoformat()
        )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/batch")
async def batch_predict(input_data: BatchPredictionInput):
    """Batch prediction with caching"""

    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    start_time = time.time()

    try:
        results = []
        cache_keys = []

        # Generate cache keys
        if input_data.use_cache:
            cache_keys = [
                cache.generate_key(features)
                for features in input_data.instances
            ]

            # Check cache
            cached_results = cache.get_batch(cache_keys)
        else:
            cached_results = {}

        # Predict uncached instances
        uncached_indices = []
        uncached_features = []

        for idx, (instance, cache_key) in enumerate(zip(input_data.instances, cache_keys or [None] * len(input_data.instances))):
            if input_data.use_cache and cache_key and cached_results.get(cache_key):
                # Use cached result
                cached_pred = cached_results[cache_key]
                results.append({
                    'index': idx,
                    'prediction': cached_pred['prediction'],
                    'probability': cached_pred['probability'],
                    'cached': True
                })
            else:
                uncached_indices.append(idx)
                uncached_features.append(instance)

        # Batch predict uncached
        if uncached_features:
            features_array = np.array(uncached_features)
            predictions = model.predict(features_array)
            probabilities = model.predict_proba(features_array)

            # Store predictions
            new_cache_entries = {}

            for i, (idx, pred, prob) in enumerate(zip(uncached_indices, predictions, probabilities)):
                prediction_data = {
                    'prediction': int(pred),
                    'probability': float(prob[1])
                }

                results.append({
                    'index': idx,
                    **prediction_data,
                    'cached': False
                })

                # Prepare for batch cache set
                if input_data.use_cache and cache_keys:
                    new_cache_entries[cache_keys[idx]] = prediction_data

            # Batch set cache
            if new_cache_entries:
                cache.set_batch(new_cache_entries)

        # Sort by index
        results.sort(key=lambda x: x['index'])

        latency_ms = (time.time() - start_time) * 1000
        cached_count = sum(1 for r in results if r['cached'])

        return {
            'predictions': results,
            'total': len(results),
            'cached': cached_count,
            'computed': len(results) - cached_count,
            'latency_ms': round(latency_ms, 2),
            'timestamp': datetime.now().isoformat()
        }

    except Exception as e:
        logger.error(f"Batch prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/cache/stats")
def get_cache_stats():
    """Get cache statistics"""
    try:
        stats = cache.get_stats()
        return {
            'cache_stats': stats,
            'timestamp': datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/cache/clear")
def clear_cache():
    """Clear all cached predictions"""
    try:
        cache.clear_all()
        return {
            'message': 'Cache cleared successfully',
            'timestamp': datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    """Health check including cache status"""
    try:
        # Test Redis connection
        cache.redis.ping()
        redis_healthy = True
    except:
        redis_healthy = False

    return {
        'status': 'healthy' if model and redis_healthy else 'unhealthy',
        'model_loaded': model is not None,
        'redis_connected': redis_healthy,
        'cache_stats': cache.get_stats() if redis_healthy else {},
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
  redis:
    image: redis:alpine
    container_name: ml-redis
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  api:
    build: .
    container_name: ml-api-cached
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
    volumes:
      - ./models:/app/models
    restart: unless-stopped

  # Optional: Redis Commander for monitoring
  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: redis-commander
    ports:
      - "8081:8081"
    environment:
      - REDIS_HOSTS=local:redis:6379
    depends_on:
      - redis
```

### Step 5: Create Load Testing

Create `load_test.py`:

```python
from locust import HttpUser, task, between
import random
import numpy as np

class MLAPIUser(HttpUser):
    wait_time = between(0.1, 0.5)

    def on_start(self):
        """Initialize user"""
        self.user_id = f"user_{random.randint(1, 1000)}"

    @task(10)
    def predict_cached(self):
        """Test prediction with caching (popular features)"""
        # Use one of 100 feature sets (high cache hit rate)
        feature_id = random.randint(1, 100)
        features = [float(feature_id + i * 0.1) for i in range(50)]

        self.client.post("/predict", json={
            "features": features,
            "user_id": self.user_id,
            "use_cache": True
        })

    @task(3)
    def predict_uncached(self):
        """Test prediction without cache (unique features)"""
        features = np.random.rand(50).tolist()

        self.client.post("/predict", json={
            "features": features,
            "user_id": self.user_id,
            "use_cache": True
        })

    @task(2)
    def batch_predict(self):
        """Test batch prediction"""
        instances = [
            [float(random.randint(1, 100) + i * 0.1) for i in range(50)]
            for _ in range(10)
        ]

        self.client.post("/predict/batch", json={
            "instances": instances,
            "use_cache": True
        })

    @task(1)
    def get_cache_stats(self):
        """Get cache statistics"""
        self.client.get("/cache/stats")
```

### Step 6: Create Benchmark Script

Create `benchmark.py`:

```python
import requests
import numpy as np
import time
from concurrent.futures import ThreadPoolExecutor

BASE_URL = "http://localhost:8000"

def benchmark_cache_performance():
    """Benchmark cache vs no-cache performance"""

    print("Benchmarking Cache Performance\n")

    # Test 1: Same features (cache hits)
    print("1. Testing cache hits (same features)...")
    features = np.random.rand(50).tolist()

    times_with_cache = []
    for i in range(100):
        start = time.time()
        response = requests.post(
            f"{BASE_URL}/predict",
            json={'features': features, 'use_cache': True}
        )
        times_with_cache.append((time.time() - start) * 1000)

    cached_count = sum(1 for i in range(1, 100) if times_with_cache[i] < times_with_cache[0] / 2)

    print(f"  First request: {times_with_cache[0]:.2f}ms")
    print(f"  Average (2-100): {np.mean(times_with_cache[1:]):.2f}ms")
    print(f"  Cache hits: {cached_count}/99")
    print(f"  Speedup: {times_with_cache[0] / np.mean(times_with_cache[1:]):.1f}x")

    # Test 2: Different features (cache misses)
    print("\n2. Testing cache misses (different features)...")
    times_no_hits = []
    for i in range(100):
        features = np.random.rand(50).tolist()
        start = time.time()
        response = requests.post(
            f"{BASE_URL}/predict",
            json={'features': features, 'use_cache': True}
        )
        times_no_hits.append((time.time() - start) * 1000)

    print(f"  Average latency: {np.mean(times_no_hits):.2f}ms")

    # Test 3: Batch predictions
    print("\n3. Testing batch predictions...")
    instances = [np.random.rand(50).tolist() for _ in range(100)]

    start = time.time()
    response = requests.post(
        f"{BASE_URL}/predict/batch",
        json={'instances': instances, 'use_cache': False}
    )
    batch_time_no_cache = (time.time() - start) * 1000

    # Second request (with cache)
    start = time.time()
    response = requests.post(
        f"{BASE_URL}/predict/batch",
        json={'instances': instances, 'use_cache': True}
    )
    batch_time_cached = (time.time() - start) * 1000
    result = response.json()

    print(f"  First batch (no cache): {batch_time_no_cache:.2f}ms")
    print(f"  Second batch (cached): {batch_time_cached:.2f}ms")
    print(f"  Cached: {result['cached']}/{result['total']}")
    print(f"  Speedup: {batch_time_no_cache / batch_time_cached:.1f}x")

    # Get cache stats
    print("\n4. Cache Statistics...")
    response = requests.get(f"{BASE_URL}/cache/stats")
    stats = response.json()['cache_stats']

    print(f"  Total requests: {stats['total_requests']}")
    print(f"  Cache hits: {stats['hits']}")
    print(f"  Cache misses: {stats['misses']}")
    print(f"  Hit rate: {stats['hit_rate']:.2f}%")
    print(f"  Cache size: {stats['cache_size']} entries")

if __name__ == "__main__":
    benchmark_cache_performance()
```

### Step 7: Create Requirements and Dockerfile

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
xgboost==2.0.3
lightgbm==4.1.0
numpy==1.24.3
joblib==1.3.2
redis==5.0.1
locust==2.20.0
```

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY cache_manager.py .
COPY models/ models/

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Expected Outputs

### Cached Prediction (Hit)
```json
{
  "prediction": 1,
  "probability": 0.7823,
  "cached": true,
  "latency_ms": 2.5,
  "timestamp": "2024-11-14T10:30:00"
}
```

### Cache Statistics
```json
{
  "cache_stats": {
    "hits": 8542,
    "misses": 1458,
    "total_requests": 10000,
    "hit_rate": 85.42,
    "cache_size": 1234
  }
}
```

### Performance Improvement
```
Without cache: 45ms average
With cache (hit): 2ms average
Speedup: 22.5x
Hit rate: 85%+
```

## Bonus Challenges

1. **Probabilistic Caching**: Use Bloom filters for cache presence
2. **Predictive Preloading**: Predict and cache likely requests
3. **Tiered Caching**: L1 (memory) + L2 (Redis)
4. **Adaptive TTL**: Adjust TTL based on access patterns
5. **Cache Compression**: Compress cached values
6. **Distributed Cache**: Redis Cluster for scaling
7. **Smart Invalidation**: Invalidate based on model updates
8. **Cache Metrics**: Detailed Prometheus metrics
9. **Request Coalescing**: Merge concurrent identical requests
10. **Feature Hashing**: Reduce cache key size

## Resources

- [Redis Documentation](https://redis.io/documentation)
- [Caching Strategies](https://redis.io/docs/manual/patterns/)
- [Redis Best Practices](https://redis.io/docs/manual/patterns/best-practices/)

## Success Criteria

- [ ] Model loads and serves predictions
- [ ] Redis caching implemented
- [ ] Cache hits are 10-20x faster than misses
- [ ] Cache hit rate > 70% for repeated requests
- [ ] Batch predictions utilize cache
- [ ] Rate limiting works correctly
- [ ] Cache stats accurate
- [ ] Cache clear works
- [ ] No cache on error doesn't break service
- [ ] TTL expires correctly
- [ ] Health check includes cache status
- [ ] Load test shows performance improvement

## Project Structure

```
redis-caching-realtime/
├── models/
│   ├── xgb_model.joblib
│   ├── lgb_model.joblib
│   └── metadata.json
├── app.py
├── cache_manager.py
├── train_model.py
├── benchmark.py
├── load_test.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── README.md
```
