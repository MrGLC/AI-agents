# Project 8: Build Rate-Limited API with Redis

## Overview
Create an ML API with sophisticated rate limiting using Redis. Implement multiple rate limiting strategies (fixed window, sliding window, token bucket), per-user limits, IP-based limits, and real-time quota management. Essential for preventing abuse and managing API costs.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Implement rate limiting with Redis
- Use Redis for distributed caching
- Create multiple rate limiting algorithms
- Implement tiered rate limits per user
- Track API usage metrics in real-time
- Handle rate limit exceeded responses
- Create rate limit reset mechanisms
- Monitor Redis performance

## Technical Stack
- **Framework**: FastAPI
- **Cache/Rate Limit**: Redis
- **Redis Client**: redis-py, aioredis
- **ML Framework**: PyTorch
- **Testing**: pytest, pytest-redis
- **Monitoring**: Redis CLI, redis-py metrics

## Project Requirements

### 1. Rate Limiting Strategies
- Fixed window counter
- Sliding window log
- Token bucket algorithm
- Leaky bucket algorithm
- Per-user rate limits
- Per-IP rate limits
- Per-endpoint rate limits

### 2. API Features
- Automatic rate limit headers (X-RateLimit-*)
- Rate limit exceeded responses (429)
- Quota reset information
- Burst allowance
- Premium tier limits
- Rate limit bypass for admins

### 3. Redis Usage
- Request counting
- Result caching
- Session management
- Distributed locking
- Pub/Sub for notifications

### 4. Monitoring
- Real-time usage dashboards
- Rate limit alerts
- Usage analytics
- Performance metrics

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn redis aioredis
pip install torch torchvision pillow
pip install pytest pytest-asyncio pytest-redis
pip install python-dotenv

# Start Redis (using Docker)
docker run -d -p 6379:6379 redis:latest

# Or install Redis locally
# Mac: brew install redis && redis-server
# Linux: sudo apt-get install redis-server && redis-server
```

### Step 2: Project Structure
```bash
mkdir rate_limited_api
cd rate_limited_api

touch main.py rate_limiter.py redis_client.py
touch models.py config.py middleware.py
touch cache.py test_rate_limit.py
```

### Step 3: Configuration (config.py)
```python
from pydantic import BaseSettings
from typing import Dict

class Settings(BaseSettings):
    # API Settings
    API_TITLE = "Rate-Limited ML API"
    API_VERSION = "1.0.0"

    # Redis Settings
    REDIS_HOST: str = "localhost"
    REDIS_PORT: int = 6379
    REDIS_DB: int = 0
    REDIS_PASSWORD: str = ""
    REDIS_URL: str = f"redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}"

    # Rate Limit Settings (requests per minute)
    RATE_LIMIT_GUEST: int = 10
    RATE_LIMIT_BASIC: int = 60
    RATE_LIMIT_PREMIUM: int = 300
    RATE_LIMIT_ADMIN: int = -1  # Unlimited

    # Rate Limit Windows (in seconds)
    WINDOW_SIZE: int = 60  # 1 minute
    BURST_SIZE: int = 5  # Allow burst of 5 requests

    # Cache Settings
    CACHE_TTL: int = 300  # 5 minutes
    CACHE_ENABLED: bool = True

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Redis Client (redis_client.py)
```python
import redis.asyncio as redis
from typing import Optional
import json

from config import settings

class RedisClient:
    """Async Redis client wrapper"""

    def __init__(self):
        self.redis: Optional[redis.Redis] = None

    async def connect(self):
        """Connect to Redis"""
        self.redis = await redis.from_url(
            settings.REDIS_URL,
            encoding="utf-8",
            decode_responses=True
        )
        print("Connected to Redis")

    async def disconnect(self):
        """Disconnect from Redis"""
        if self.redis:
            await self.redis.close()

    async def ping(self) -> bool:
        """Test Redis connection"""
        try:
            return await self.redis.ping()
        except Exception as e:
            print(f"Redis ping failed: {e}")
            return False

    # Rate limiting operations

    async def increment_counter(
        self,
        key: str,
        expire_seconds: int = 60
    ) -> int:
        """Increment counter with expiration"""
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, expire_seconds)
        results = await pipe.execute()
        return results[0]

    async def get_counter(self, key: str) -> int:
        """Get counter value"""
        value = await self.redis.get(key)
        return int(value) if value else 0

    async def add_to_sorted_set(
        self,
        key: str,
        value: float,
        score: float,
        expire_seconds: int = 60
    ):
        """Add to sorted set (for sliding window)"""
        pipe = self.redis.pipeline()
        pipe.zadd(key, {str(value): score})
        pipe.expire(key, expire_seconds)
        await pipe.execute()

    async def remove_from_sorted_set_by_score(
        self,
        key: str,
        min_score: float,
        max_score: float
    ):
        """Remove items from sorted set by score range"""
        await self.redis.zremrangebyscore(key, min_score, max_score)

    async def count_sorted_set(
        self,
        key: str,
        min_score: float,
        max_score: float
    ) -> int:
        """Count items in sorted set within score range"""
        return await self.redis.zcount(key, min_score, max_score)

    # Caching operations

    async def set_cache(
        self,
        key: str,
        value: dict,
        ttl: int = 300
    ):
        """Set cache with TTL"""
        await self.redis.setex(
            key,
            ttl,
            json.dumps(value)
        )

    async def get_cache(self, key: str) -> Optional[dict]:
        """Get cached value"""
        value = await self.redis.get(key)
        return json.loads(value) if value else None

    async def delete_cache(self, key: str):
        """Delete cached value"""
        await self.redis.delete(key)

    # Token bucket operations

    async def consume_tokens(
        self,
        key: str,
        tokens: int,
        max_tokens: int,
        refill_rate: float
    ) -> tuple[bool, int]:
        """
        Token bucket algorithm

        Returns: (allowed, remaining_tokens)
        """
        now = await self.get_current_time()

        # Get current bucket state
        bucket_data = await self.redis.hgetall(key)

        if not bucket_data:
            # Initialize bucket
            current_tokens = max_tokens - tokens
            last_refill = now
        else:
            current_tokens = float(bucket_data.get('tokens', max_tokens))
            last_refill = float(bucket_data.get('last_refill', now))

            # Refill tokens based on time passed
            time_passed = now - last_refill
            tokens_to_add = time_passed * refill_rate
            current_tokens = min(max_tokens, current_tokens + tokens_to_add)

        # Try to consume tokens
        if current_tokens >= tokens:
            current_tokens -= tokens
            allowed = True
        else:
            allowed = False

        # Update bucket state
        await self.redis.hset(
            key,
            mapping={
                'tokens': current_tokens,
                'last_refill': now
            }
        )
        await self.redis.expire(key, 3600)  # 1 hour expiry

        return allowed, int(current_tokens)

    async def get_current_time(self) -> float:
        """Get current time from Redis server"""
        time_data = await self.redis.time()
        return float(time_data[0]) + float(time_data[1]) / 1000000

# Global Redis client
redis_client = RedisClient()
```

### Step 5: Rate Limiter (rate_limiter.py)
```python
from typing import Optional
import time
from fastapi import HTTPException, status

from redis_client import redis_client
from config import settings

class RateLimiter:
    """Rate limiting implementation"""

    def __init__(self, strategy: str = "fixed_window"):
        self.strategy = strategy
        self.window_size = settings.WINDOW_SIZE

    async def check_rate_limit(
        self,
        identifier: str,
        limit: int,
        window: Optional[int] = None
    ) -> tuple[bool, dict]:
        """
        Check if request is allowed

        Returns: (allowed, info_dict)
        """
        if limit == -1:  # Unlimited (admin)
            return True, {
                'limit': -1,
                'remaining': -1,
                'reset': 0
            }

        window = window or self.window_size

        if self.strategy == "fixed_window":
            return await self._fixed_window(identifier, limit, window)
        elif self.strategy == "sliding_window":
            return await self._sliding_window(identifier, limit, window)
        elif self.strategy == "token_bucket":
            return await self._token_bucket(identifier, limit, window)
        else:
            raise ValueError(f"Unknown strategy: {self.strategy}")

    async def _fixed_window(
        self,
        identifier: str,
        limit: int,
        window: int
    ) -> tuple[bool, dict]:
        """Fixed window counter algorithm"""
        key = f"rate_limit:fixed:{identifier}:{int(time.time()) // window}"

        current = await redis_client.increment_counter(key, window)

        allowed = current <= limit
        remaining = max(0, limit - current)
        reset = (int(time.time()) // window + 1) * window

        return allowed, {
            'limit': limit,
            'remaining': remaining,
            'reset': reset,
            'current': current
        }

    async def _sliding_window(
        self,
        identifier: str,
        limit: int,
        window: int
    ) -> tuple[bool, dict]:
        """Sliding window log algorithm"""
        key = f"rate_limit:sliding:{identifier}"
        now = time.time()
        window_start = now - window

        # Remove old entries
        await redis_client.remove_from_sorted_set_by_score(
            key,
            0,
            window_start
        )

        # Count requests in window
        current = await redis_client.count_sorted_set(
            key,
            window_start,
            now
        )

        allowed = current < limit

        if allowed:
            # Add current request
            await redis_client.add_to_sorted_set(
                key,
                now,
                now,
                window * 2
            )
            current += 1

        remaining = max(0, limit - current)
        reset = int(now + window)

        return allowed, {
            'limit': limit,
            'remaining': remaining,
            'reset': reset,
            'current': current
        }

    async def _token_bucket(
        self,
        identifier: str,
        limit: int,
        window: int
    ) -> tuple[bool, dict]:
        """Token bucket algorithm"""
        key = f"rate_limit:token:{identifier}"

        # Refill rate: limit tokens per window
        refill_rate = limit / window

        allowed, remaining_tokens = await redis_client.consume_tokens(
            key,
            tokens=1,
            max_tokens=limit,
            refill_rate=refill_rate
        )

        # Calculate reset time
        if not allowed:
            time_to_refill = (1 / refill_rate) if refill_rate > 0 else window
            reset = int(time.time() + time_to_refill)
        else:
            reset = int(time.time() + window)

        return allowed, {
            'limit': limit,
            'remaining': remaining_tokens,
            'reset': reset
        }

    async def get_rate_limit_info(
        self,
        identifier: str,
        limit: int
    ) -> dict:
        """Get current rate limit status without incrementing"""
        key = f"rate_limit:fixed:{identifier}:{int(time.time()) // self.window_size}"

        current = await redis_client.get_counter(key)
        remaining = max(0, limit - current)
        reset = (int(time.time()) // self.window_size + 1) * self.window_size

        return {
            'limit': limit,
            'remaining': remaining,
            'reset': reset,
            'current': current
        }

# Global rate limiter
rate_limiter = RateLimiter(strategy="sliding_window")
```

### Step 6: Cache Layer (cache.py)
```python
import hashlib
import json
from typing import Optional, Callable, Any
from functools import wraps

from redis_client import redis_client
from config import settings

def generate_cache_key(prefix: str, *args, **kwargs) -> str:
    """Generate cache key from function arguments"""
    key_data = json.dumps({
        'args': args,
        'kwargs': sorted(kwargs.items())
    }, sort_keys=True)

    hash_key = hashlib.md5(key_data.encode()).hexdigest()
    return f"{prefix}:{hash_key}"

def cached(prefix: str, ttl: int = None):
    """Decorator to cache function results"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            if not settings.CACHE_ENABLED:
                return await func(*args, **kwargs)

            # Generate cache key
            cache_key = generate_cache_key(prefix, *args, **kwargs)

            # Try to get from cache
            cached_result = await redis_client.get_cache(cache_key)
            if cached_result is not None:
                return cached_result

            # Execute function
            result = await func(*args, **kwargs)

            # Cache result
            cache_ttl = ttl or settings.CACHE_TTL
            await redis_client.set_cache(cache_key, result, cache_ttl)

            return result

        return wrapper
    return decorator

async def invalidate_cache(prefix: str, *args, **kwargs):
    """Invalidate specific cache entry"""
    cache_key = generate_cache_key(prefix, *args, **kwargs)
    await redis_client.delete_cache(cache_key)
```

### Step 7: Rate Limit Middleware (middleware.py)
```python
from fastapi import Request, HTTPException, status
from starlette.middleware.base import BaseHTTPMiddleware
from typing import Callable

from rate_limiter import rate_limiter
from config import settings

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Middleware to enforce rate limits"""

    async def dispatch(self, request: Request, call_next: Callable):
        # Skip rate limiting for health check
        if request.url.path in ["/health", "/docs", "/openapi.json"]:
            return await call_next(request)

        # Get user identifier (IP or user_id)
        client_ip = request.client.host
        user_id = getattr(request.state, 'user_id', None)
        identifier = f"user:{user_id}" if user_id else f"ip:{client_ip}"

        # Get rate limit for user (default to GUEST)
        limit = settings.RATE_LIMIT_GUEST

        # Check if user is authenticated and get their limit
        # (This would integrate with your auth system)
        user_role = getattr(request.state, 'user_role', 'guest')
        limit = {
            'guest': settings.RATE_LIMIT_GUEST,
            'basic': settings.RATE_LIMIT_BASIC,
            'premium': settings.RATE_LIMIT_PREMIUM,
            'admin': settings.RATE_LIMIT_ADMIN
        }.get(user_role, settings.RATE_LIMIT_GUEST)

        # Check rate limit
        allowed, info = await rate_limiter.check_rate_limit(
            identifier,
            limit,
            settings.WINDOW_SIZE
        )

        # Add rate limit headers
        response = None
        if allowed:
            response = await call_next(request)
        else:
            response = HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail={
                    "error": "Rate limit exceeded",
                    "limit": info['limit'],
                    "reset": info['reset'],
                    "retry_after": info['reset'] - int(time.time())
                }
            )
            response = JSONResponse(
                status_code=429,
                content=response.detail
            )

        # Add rate limit headers
        response.headers["X-RateLimit-Limit"] = str(info['limit'])
        response.headers["X-RateLimit-Remaining"] = str(info['remaining'])
        response.headers["X-RateLimit-Reset"] = str(info['reset'])

        return response

from fastapi.responses import JSONResponse
import time
```

### Step 8: Main Application (main.py)
```python
from fastapi import FastAPI, Request, HTTPException
from contextlib import asynccontextmanager
import time

from redis_client import redis_client
from rate_limiter import rate_limiter
from middleware import RateLimitMiddleware
from cache import cached
from config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifespan context manager"""
    # Startup
    await redis_client.connect()
    yield
    # Shutdown
    await redis_client.disconnect()

app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION,
    lifespan=lifespan
)

# Add rate limit middleware
app.add_middleware(RateLimitMiddleware)

@app.get("/")
async def root():
    """API info"""
    return {
        "message": "Rate-Limited ML API",
        "rate_limits": {
            "guest": f"{settings.RATE_LIMIT_GUEST}/min",
            "basic": f"{settings.RATE_LIMIT_BASIC}/min",
            "premium": f"{settings.RATE_LIMIT_PREMIUM}/min"
        }
    }

@app.get("/health")
async def health():
    """Health check"""
    redis_ok = await redis_client.ping()

    return {
        "status": "healthy" if redis_ok else "unhealthy",
        "redis": "connected" if redis_ok else "disconnected"
    }

@app.post("/predict")
@cached(prefix="prediction", ttl=300)
async def predict(request: Request, image: str):
    """
    ML Prediction endpoint (rate-limited and cached)

    Returns cached results for identical inputs
    """
    # Simulate ML prediction
    time.sleep(0.1)

    result = {
        "predictions": [
            {"golden_retriever": 0.87},
            {"labrador": 0.09}
        ],
        "top_prediction": "golden_retriever",
        "confidence": 0.87,
        "cached": False
    }

    return result

@app.get("/rate-limit/status")
async def get_rate_limit_status(request: Request):
    """Get current rate limit status"""
    client_ip = request.client.host
    identifier = f"ip:{client_ip}"

    info = await rate_limiter.get_rate_limit_info(
        identifier,
        settings.RATE_LIMIT_GUEST
    )

    return {
        "identifier": identifier,
        "limit": info['limit'],
        "remaining": info['remaining'],
        "reset": info['reset'],
        "reset_in_seconds": info['reset'] - int(time.time())
    }

@app.post("/cache/clear")
async def clear_cache():
    """Clear all cached predictions (admin only)"""
    # In production, add authentication
    # For now, this is a simple example
    return {"message": "Cache cleared (not implemented)"}

@app.get("/stats")
async def get_stats():
    """Get API statistics"""
    # Get Redis info
    redis_ok = await redis_client.ping()

    return {
        "status": "operational",
        "redis_connected": redis_ok,
        "cache_enabled": settings.CACHE_ENABLED,
        "rate_limit_window": f"{settings.WINDOW_SIZE}s"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 9: Test Rate Limiting
```python
# test_rate_limit.py
import pytest
import asyncio
from httpx import AsyncClient
from main import app

@pytest.mark.asyncio
async def test_rate_limit():
    """Test rate limiting"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Make requests up to limit
        responses = []
        for i in range(15):
            response = await client.post(
                "/predict",
                json={"image": "test"}
            )
            responses.append(response)
            await asyncio.sleep(0.1)

        # Check that some requests were rate limited
        success_count = sum(1 for r in responses if r.status_code == 200)
        limited_count = sum(1 for r in responses if r.status_code == 429)

        assert limited_count > 0, "No requests were rate limited"
        assert success_count <= 10, "Too many requests allowed"

@pytest.mark.asyncio
async def test_rate_limit_headers():
    """Test rate limit headers"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/predict", json={"image": "test"})

        assert "X-RateLimit-Limit" in response.headers
        assert "X-RateLimit-Remaining" in response.headers
        assert "X-RateLimit-Reset" in response.headers
```

### Step 10: Run and Test
```bash
# Start Redis
redis-server

# Start API
uvicorn main:app --reload

# Test rate limiting
for i in {1..20}; do
  curl -X POST http://localhost:8000/predict \
    -H "Content-Type: application/json" \
    -d '{"image": "base64data"}' \
    -w "\nStatus: %{http_code}\n"
  sleep 0.5
done

# Check rate limit status
curl http://localhost:8000/rate-limit/status
```

## Expected Outputs

### 1. Successful Request (200)
```json
{
  "predictions": [
    {"golden_retriever": 0.87},
    {"labrador": 0.09}
  ],
  "top_prediction": "golden_retriever",
  "confidence": 0.87,
  "cached": false
}
```

**Headers:**
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1699965600
```

### 2. Rate Limited Request (429)
```json
{
  "error": "Rate limit exceeded",
  "limit": 60,
  "reset": 1699965600,
  "retry_after": 23
}
```

### 3. Rate Limit Status
```json
{
  "identifier": "ip:127.0.0.1",
  "limit": 60,
  "remaining": 32,
  "reset": 1699965600,
  "reset_in_seconds": 45
}
```

## Bonus Challenges

- [ ] Add distributed rate limiting across multiple servers
- [ ] Implement rate limit exemptions for specific IPs
- [ ] Create rate limit analytics dashboard
- [ ] Add dynamic rate limit adjustment
- [ ] Implement cost-based rate limiting (expensive operations)
- [ ] Add rate limit warnings before hitting limit
- [ ] Create rate limit purchase/upgrade flow
- [ ] Implement geographic rate limiting
- [ ] Add DDoS protection with progressive challenges
- [ ] Create rate limit reporting API
- [ ] Add Redis Cluster support
- [ ] Implement circuit breaker pattern

## Resources

- [Redis Rate Limiting](https://redis.io/docs/manual/patterns/rate-limiter/)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
- [Sliding Window Rate Limiting](https://hechao.li/2018/06/25/Rate-Limiter-Part1/)
- [FastAPI Middleware](https://fastapi.tiangolo.com/tutorial/middleware/)
- [redis-py Documentation](https://redis-py.readthedocs.io/)

## Success Criteria

- [ ] Redis connection established successfully
- [ ] Rate limiting works for different user tiers
- [ ] Rate limit headers are returned
- [ ] 429 responses when limit exceeded
- [ ] Caching reduces redundant predictions
- [ ] Multiple rate limit strategies work
- [ ] Redis operations are efficient
- [ ] All tests pass
- [ ] Rate limits reset correctly
- [ ] System handles high load
