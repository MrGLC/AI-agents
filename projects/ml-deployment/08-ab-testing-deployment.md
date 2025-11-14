# Project 08: Deploy Model with A/B Testing Capability

## Overview

Implement A/B testing infrastructure for machine learning models, enabling controlled experiments to compare model performance in production. This project covers traffic splitting, experiment tracking, statistical analysis, and automated decision-making for model deployment.

## Learning Objectives

- Implement A/B testing for ML models
- Configure traffic splitting strategies
- Track experiment metrics and user assignments
- Perform statistical significance testing
- Implement multi-armed bandit algorithms
- Create experiment dashboards
- Automate winner selection
- Handle experiment lifecycle (start, pause, conclude)
- Implement user segmentation for experiments
- Monitor model performance in real-time

## Difficulty Level

**Advanced** - Requires understanding of experimental design, statistics, and production deployment strategies.

## Technical Stack

- **ML Framework**: scikit-learn, XGBoost
- **Web Framework**: FastAPI
- **Database**: PostgreSQL (experiment data), Redis (user assignments)
- **Traffic Splitting**: Custom middleware
- **Analytics**: Pandas, SciPy
- **Visualization**: Plotly (experiment dashboard)
- **Monitoring**: Prometheus, Grafana
- **Container**: Docker, Docker Compose

## Requirements

### Model Requirements
- Baseline model (Model A - control)
- Challenger model (Model B - variant)
- Version tracking and metadata
- Rollback capability

### A/B Testing Requirements
- Traffic splitting (50/50, 90/10, custom ratios)
- User assignment persistence
- Experiment configuration management
- Metrics collection and analysis
- Statistical significance calculation

### API Requirements
- POST /predict - Prediction with A/B assignment
- POST /feedback - Record feedback/outcomes
- GET /experiments - List active experiments
- POST /experiments - Create new experiment
- GET /experiments/{id}/results - View results
- POST /experiments/{id}/conclude - End experiment

## Step-by-Step Implementation

### Step 1: Train Models

Create `train_models.py`:

```python
import numpy as np
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
import xgboost as xgb
import joblib
import json
from datetime import datetime

def create_dataset():
    """Create classification dataset"""
    X, y = make_classification(
        n_samples=5000,
        n_features=20,
        n_informative=15,
        n_classes=2,
        weights=[0.7, 0.3],
        random_state=42
    )
    return train_test_split(X, y, test_size=0.2, random_state=42)

def train_baseline_model():
    """Train baseline model (Model A - Control)"""
    print("Training baseline model...")
    X_train, X_test, y_train, y_test = create_dataset()

    model = RandomForestClassifier(
        n_estimators=100,
        max_depth=10,
        random_state=42
    )
    model.fit(X_train, y_train)
    accuracy = model.score(X_test, y_test)

    print(f"Baseline Accuracy: {accuracy:.4f}")

    # Save
    joblib.dump(model, 'models/model_a_baseline.joblib')

    metadata = {
        'model_id': 'model_a',
        'name': 'Baseline Random Forest',
        'type': 'RandomForestClassifier',
        'accuracy': float(accuracy),
        'created_at': datetime.now().isoformat()
    }

    with open('models/model_a_metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    return model, accuracy

def train_challenger_model():
    """Train challenger model (Model B - Variant)"""
    print("\nTraining challenger model...")
    X_train, X_test, y_train, y_test = create_dataset()

    model = xgb.XGBClassifier(
        n_estimators=150,
        max_depth=6,
        learning_rate=0.05,
        random_state=42
    )
    model.fit(X_train, y_train)
    accuracy = model.score(X_test, y_test)

    print(f"Challenger Accuracy: {accuracy:.4f}")

    # Save
    joblib.dump(model, 'models/model_b_challenger.joblib')

    metadata = {
        'model_id': 'model_b',
        'name': 'Challenger XGBoost',
        'type': 'XGBClassifier',
        'accuracy': float(accuracy),
        'created_at': datetime.now().isoformat()
    }

    with open('models/model_b_metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    return model, accuracy

if __name__ == "__main__":
    import os
    os.makedirs('models', exist_ok=True)

    baseline_model, baseline_acc = train_baseline_model()
    challenger_model, challenger_acc = train_challenger_model()

    print(f"\n{'='*60}")
    print(f"Baseline (A): {baseline_acc:.4f}")
    print(f"Challenger (B): {challenger_acc:.4f}")
    print(f"Expected improvement: {((challenger_acc - baseline_acc) / baseline_acc * 100):.2f}%")
```

### Step 2: Create Experiment Manager

Create `experiment_manager.py`:

```python
import psycopg2
from psycopg2.extras import RealDictCursor
import redis
import json
import hashlib
from datetime import datetime
from typing import Optional, Dict, List
import numpy as np
from scipy import stats

class ExperimentManager:
    """Manage A/B testing experiments"""

    def __init__(self, db_config: Dict, redis_config: Dict):
        self.db = psycopg2.connect(**db_config)
        self.redis = redis.Redis(**redis_config, decode_responses=True)
        self._init_db()

    def _init_db(self):
        """Initialize database schema"""
        with self.db.cursor() as cur:
            # Experiments table
            cur.execute("""
                CREATE TABLE IF NOT EXISTS experiments (
                    id SERIAL PRIMARY KEY,
                    name VARCHAR(255) NOT NULL,
                    description TEXT,
                    model_a VARCHAR(100) NOT NULL,
                    model_b VARCHAR(100) NOT NULL,
                    traffic_split FLOAT DEFAULT 0.5,
                    status VARCHAR(50) DEFAULT 'active',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    concluded_at TIMESTAMP
                )
            """)

            # Predictions table
            cur.execute("""
                CREATE TABLE IF NOT EXISTS predictions (
                    id SERIAL PRIMARY KEY,
                    experiment_id INTEGER REFERENCES experiments(id),
                    user_id VARCHAR(255),
                    variant VARCHAR(10),
                    prediction INTEGER,
                    probability FLOAT,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            # Feedback table
            cur.execute("""
                CREATE TABLE IF NOT EXISTS feedback (
                    id SERIAL PRIMARY KEY,
                    prediction_id INTEGER REFERENCES predictions(id),
                    actual_outcome INTEGER,
                    latency_ms FLOAT,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)

            self.db.commit()

    def create_experiment(
        self,
        name: str,
        description: str,
        model_a: str,
        model_b: str,
        traffic_split: float = 0.5
    ) -> int:
        """Create new A/B experiment"""
        with self.db.cursor() as cur:
            cur.execute("""
                INSERT INTO experiments (name, description, model_a, model_b, traffic_split)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id
            """, (name, description, model_a, model_b, traffic_split))
            experiment_id = cur.fetchone()[0]
            self.db.commit()

        print(f"Created experiment {experiment_id}: {name}")
        return experiment_id

    def assign_variant(self, experiment_id: int, user_id: str) -> str:
        """Assign user to variant (A or B) with persistence"""

        # Check if user already assigned
        cache_key = f"experiment:{experiment_id}:user:{user_id}"
        cached_variant = self.redis.get(cache_key)

        if cached_variant:
            return cached_variant

        # Get experiment details
        with self.db.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute(
                "SELECT traffic_split FROM experiments WHERE id = %s",
                (experiment_id,)
            )
            exp = cur.fetchone()

        if not exp:
            return 'A'  # Default to control

        # Deterministic assignment based on user_id hash
        hash_value = int(hashlib.md5(f"{user_id}:{experiment_id}".encode()).hexdigest(), 16)
        variant = 'B' if (hash_value % 100) / 100 < exp['traffic_split'] else 'A'

        # Cache assignment
        self.redis.setex(cache_key, 86400, variant)  # 24 hour TTL

        return variant

    def log_prediction(
        self,
        experiment_id: int,
        user_id: str,
        variant: str,
        prediction: int,
        probability: float
    ) -> int:
        """Log prediction for experiment tracking"""
        with self.db.cursor() as cur:
            cur.execute("""
                INSERT INTO predictions (experiment_id, user_id, variant, prediction, probability)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id
            """, (experiment_id, user_id, variant, prediction, probability))
            prediction_id = cur.fetchone()[0]
            self.db.commit()

        return prediction_id

    def log_feedback(
        self,
        prediction_id: int,
        actual_outcome: int,
        latency_ms: float
    ):
        """Log feedback/outcome for prediction"""
        with self.db.cursor() as cur:
            cur.execute("""
                INSERT INTO feedback (prediction_id, actual_outcome, latency_ms)
                VALUES (%s, %s, %s)
            """, (prediction_id, actual_outcome, latency_ms))
            self.db.commit()

    def get_experiment_results(self, experiment_id: int) -> Dict:
        """Calculate experiment results with statistical analysis"""
        with self.db.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute("""
                SELECT
                    p.variant,
                    COUNT(*) as total_predictions,
                    COUNT(f.id) as total_feedback,
                    AVG(CASE WHEN p.prediction = f.actual_outcome THEN 1 ELSE 0 END) as accuracy,
                    AVG(f.latency_ms) as avg_latency
                FROM predictions p
                LEFT JOIN feedback f ON f.prediction_id = p.id
                WHERE p.experiment_id = %s
                GROUP BY p.variant
            """, (experiment_id,))
            results = cur.fetchall()

        if len(results) < 2:
            return {'error': 'Insufficient data for comparison'}

        # Extract metrics
        variant_a = next((r for r in results if r['variant'] == 'A'), None)
        variant_b = next((r for r in results if r['variant'] == 'B'), None)

        if not variant_a or not variant_b:
            return {'error': 'Missing variant data'}

        # Fetch individual outcomes for statistical test
        with self.db.cursor() as cur:
            cur.execute("""
                SELECT p.variant, f.actual_outcome, p.prediction
                FROM predictions p
                JOIN feedback f ON f.prediction_id = p.id
                WHERE p.experiment_id = %s
            """, (experiment_id,))
            outcomes = cur.fetchall()

        # Separate by variant
        a_correct = [1 if r[1] == r[2] else 0 for r in outcomes if r[0] == 'A']
        b_correct = [1 if r[1] == r[2] else 0 for r in outcomes if r[0] == 'B']

        # Perform statistical test
        if len(a_correct) > 30 and len(b_correct) > 30:
            t_stat, p_value = stats.ttest_ind(a_correct, b_correct)
            significant = p_value < 0.05
        else:
            t_stat, p_value, significant = None, None, False

        return {
            'experiment_id': experiment_id,
            'variant_a': {
                'total_predictions': variant_a['total_predictions'],
                'total_feedback': variant_a['total_feedback'],
                'accuracy': float(variant_a['accuracy'] or 0),
                'avg_latency_ms': float(variant_a['avg_latency'] or 0)
            },
            'variant_b': {
                'total_predictions': variant_b['total_predictions'],
                'total_feedback': variant_b['total_feedback'],
                'accuracy': float(variant_b['accuracy'] or 0),
                'avg_latency_ms': float(variant_b['avg_latency'] or 0)
            },
            'statistical_test': {
                't_statistic': float(t_stat) if t_stat else None,
                'p_value': float(p_value) if p_value else None,
                'significant': significant
            },
            'winner': self._determine_winner(variant_a, variant_b, significant)
        }

    def _determine_winner(self, variant_a, variant_b, significant: bool) -> str:
        """Determine winner based on metrics"""
        if not significant:
            return 'inconclusive'

        acc_a = variant_a['accuracy'] or 0
        acc_b = variant_b['accuracy'] or 0

        if acc_b > acc_a:
            return 'B'
        elif acc_a > acc_b:
            return 'A'
        else:
            return 'tie'

    def conclude_experiment(self, experiment_id: int, winner: str):
        """Conclude experiment and mark winner"""
        with self.db.cursor() as cur:
            cur.execute("""
                UPDATE experiments
                SET status = 'concluded', concluded_at = CURRENT_TIMESTAMP
                WHERE id = %s
            """, (experiment_id,))
            self.db.commit()

        print(f"Experiment {experiment_id} concluded. Winner: {winner}")

    def list_experiments(self) -> List[Dict]:
        """List all experiments"""
        with self.db.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute("""
                SELECT id, name, description, status, created_at
                FROM experiments
                ORDER BY created_at DESC
            """)
            return cur.fetchall()
```

### Step 3: Create API with A/B Testing

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
from experiment_manager import ExperimentManager
import os

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="A/B Testing ML API",
    description="ML API with built-in A/B testing",
    version="1.0.0"
)

# Database configuration
DB_CONFIG = {
    'host': os.environ.get('DB_HOST', 'localhost'),
    'port': int(os.environ.get('DB_PORT', 5432)),
    'database': os.environ.get('DB_NAME', 'experiments'),
    'user': os.environ.get('DB_USER', 'postgres'),
    'password': os.environ.get('DB_PASSWORD', 'postgres')
}

REDIS_CONFIG = {
    'host': os.environ.get('REDIS_HOST', 'localhost'),
    'port': int(os.environ.get('REDIS_PORT', 6379)),
    'db': 0
}

# Initialize experiment manager
exp_manager = ExperimentManager(DB_CONFIG, REDIS_CONFIG)

# Load models
models = {}
model_metadata = {}

class PredictionInput(BaseModel):
    features: List[float] = Field(..., min_items=20, max_items=20)
    user_id: str

class PredictionResponse(BaseModel):
    prediction: int
    probability: float
    variant: str
    model_id: str
    prediction_id: int
    experiment_id: int
    timestamp: str

class FeedbackInput(BaseModel):
    prediction_id: int
    actual_outcome: int

class ExperimentCreate(BaseModel):
    name: str
    description: str
    traffic_split: float = Field(default=0.5, ge=0.0, le=1.0)

@app.on_event("startup")
async def load_models():
    """Load all models"""
    global models, model_metadata

    try:
        # Load Model A (baseline)
        models['model_a'] = joblib.load('models/model_a_baseline.joblib')
        with open('models/model_a_metadata.json', 'r') as f:
            model_metadata['model_a'] = json.load(f)

        # Load Model B (challenger)
        models['model_b'] = joblib.load('models/model_b_challenger.joblib')
        with open('models/model_b_metadata.json', 'r') as f:
            model_metadata['model_b'] = json.load(f)

        logger.info("All models loaded successfully")

        # Create default experiment if none exists
        experiments = exp_manager.list_experiments()
        if not experiments:
            exp_manager.create_experiment(
                name="Baseline vs Challenger",
                description="Compare baseline Random Forest with challenger XGBoost",
                model_a="model_a",
                model_b="model_b",
                traffic_split=0.5
            )

    except Exception as e:
        logger.error(f"Failed to load models: {e}")
        raise

@app.get("/")
def root():
    """Root endpoint"""
    return {
        "service": "A/B Testing ML API",
        "models": list(models.keys()),
        "active_experiments": len([e for e in exp_manager.list_experiments() if e['status'] == 'active'])
    }

@app.post("/predict", response_model=PredictionResponse)
async def predict(
    input_data: PredictionInput,
    experiment_id: int = 1
):
    """Make prediction with A/B testing"""

    if not models:
        raise HTTPException(status_code=503, detail="Models not loaded")

    try:
        # Assign variant
        variant = exp_manager.assign_variant(experiment_id, input_data.user_id)

        # Select model based on variant
        model_id = f"model_{variant.lower()}"
        model = models.get(model_id)

        if not model:
            raise HTTPException(status_code=500, detail=f"Model {model_id} not found")

        # Make prediction
        features = np.array(input_data.features).reshape(1, -1)
        start_time = time.time()

        prediction = model.predict(features)[0]
        probability = model.predict_proba(features)[0][1]

        latency_ms = (time.time() - start_time) * 1000

        # Log prediction
        prediction_id = exp_manager.log_prediction(
            experiment_id=experiment_id,
            user_id=input_data.user_id,
            variant=variant,
            prediction=int(prediction),
            probability=float(probability)
        )

        logger.info(f"User {input_data.user_id}: Variant {variant}, "
                   f"Prediction: {prediction}, Latency: {latency_ms:.2f}ms")

        return PredictionResponse(
            prediction=int(prediction),
            probability=float(probability),
            variant=variant,
            model_id=model_id,
            prediction_id=prediction_id,
            experiment_id=experiment_id,
            timestamp=datetime.now().isoformat()
        )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/feedback")
async def submit_feedback(feedback: FeedbackInput):
    """Submit feedback for a prediction"""
    try:
        exp_manager.log_feedback(
            prediction_id=feedback.prediction_id,
            actual_outcome=feedback.actual_outcome,
            latency_ms=0.0  # Computed on prediction
        )

        return {
            'message': 'Feedback recorded',
            'prediction_id': feedback.prediction_id
        }

    except Exception as e:
        logger.error(f"Feedback error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/experiments")
def list_experiments():
    """List all experiments"""
    try:
        experiments = exp_manager.list_experiments()
        return {
            'experiments': experiments,
            'count': len(experiments)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/experiments")
def create_experiment(exp: ExperimentCreate):
    """Create new experiment"""
    try:
        exp_id = exp_manager.create_experiment(
            name=exp.name,
            description=exp.description,
            model_a="model_a",
            model_b="model_b",
            traffic_split=exp.traffic_split
        )

        return {
            'experiment_id': exp_id,
            'message': 'Experiment created successfully'
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/experiments/{experiment_id}/results")
def get_experiment_results(experiment_id: int):
    """Get experiment results with statistical analysis"""
    try:
        results = exp_manager.get_experiment_results(experiment_id)
        return results
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/experiments/{experiment_id}/conclude")
def conclude_experiment(experiment_id: int):
    """Conclude experiment"""
    try:
        results = exp_manager.get_experiment_results(experiment_id)
        winner = results.get('winner', 'inconclusive')

        exp_manager.conclude_experiment(experiment_id, winner)

        return {
            'experiment_id': experiment_id,
            'winner': winner,
            'results': results
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    """Health check"""
    return {
        'status': 'healthy',
        'models_loaded': len(models),
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
    container_name: ab-postgres
    environment:
      POSTGRES_DB: experiments
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: ab-redis
    ports:
      - "6379:6379"
    restart: unless-stopped

  api:
    build: .
    container_name: ab-api
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=experiments
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - postgres
      - redis
    volumes:
      - ./models:/app/models
    restart: unless-stopped

volumes:
  postgres_data:
```

### Step 5: Create Test Suite

Create `test_ab.py`:

```python
import requests
import random
import time
import numpy as np

BASE_URL = "http://localhost:8000"

def simulate_ab_test(n_users=100, n_predictions_per_user=5):
    """Simulate A/B test with multiple users"""

    print("Simulating A/B Test\n")

    experiment_id = 1
    prediction_ids = []

    # Make predictions
    print(f"Making predictions for {n_users} users...")
    for user_idx in range(n_users):
        user_id = f"user_{user_idx}"

        for _ in range(n_predictions_per_user):
            features = np.random.rand(20).tolist()

            response = requests.post(
                f"{BASE_URL}/predict",
                json={
                    'features': features,
                    'user_id': user_id
                },
                params={'experiment_id': experiment_id}
            )

            if response.status_code == 200:
                result = response.json()
                prediction_ids.append({
                    'prediction_id': result['prediction_id'],
                    'prediction': result['prediction'],
                    'variant': result['variant']
                })

        if (user_idx + 1) % 10 == 0:
            print(f"  Processed {user_idx + 1} users")

    # Submit feedback (simulate 80% accuracy for B, 75% for A)
    print(f"\nSubmitting feedback for {len(prediction_ids)} predictions...")
    for pred_info in prediction_ids:
        # Simulate different accuracies for variants
        if pred_info['variant'] == 'B':
            correct_prob = 0.80
        else:
            correct_prob = 0.75

        # Simulate outcome
        is_correct = random.random() < correct_prob
        actual_outcome = pred_info['prediction'] if is_correct else 1 - pred_info['prediction']

        requests.post(
            f"{BASE_URL}/feedback",
            json={
                'prediction_id': pred_info['prediction_id'],
                'actual_outcome': actual_outcome
            }
        )

    # Get results
    print("\nFetching experiment results...")
    response = requests.get(f"{BASE_URL}/experiments/{experiment_id}/results")
    results = response.json()

    print(f"\n{'='*60}")
    print("Experiment Results")
    print(f"{'='*60}")

    for variant in ['variant_a', 'variant_b']:
        data = results[variant]
        print(f"\n{variant.upper()}:")
        print(f"  Predictions: {data['total_predictions']}")
        print(f"  Feedback: {data['total_feedback']}")
        print(f"  Accuracy: {data['accuracy']:.4f}")
        print(f"  Avg Latency: {data['avg_latency_ms']:.2f}ms")

    stats = results['statistical_test']
    print(f"\nStatistical Test:")
    print(f"  P-value: {stats['p_value']:.4f}" if stats['p_value'] else "  Insufficient data")
    print(f"  Significant: {stats['significant']}")
    print(f"\nWinner: {results['winner'].upper()}")

if __name__ == "__main__":
    simulate_ab_test(n_users=100, n_predictions_per_user=5)
```

### Step 6: Create Requirements and Dockerfile

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
xgboost==2.0.3
numpy==1.24.3
joblib==1.3.2
psycopg2-binary==2.9.9
redis==5.0.1
scipy==1.11.4
pandas==2.1.4
```

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY experiment_manager.py .
COPY models/ models/

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Expected Outputs

### Prediction with A/B Assignment
```json
{
  "prediction": 1,
  "probability": 0.7823,
  "variant": "B",
  "model_id": "model_b",
  "prediction_id": 42,
  "experiment_id": 1,
  "timestamp": "2024-11-14T10:30:00"
}
```

### Experiment Results
```json
{
  "experiment_id": 1,
  "variant_a": {
    "total_predictions": 250,
    "total_feedback": 248,
    "accuracy": 0.7540,
    "avg_latency_ms": 12.5
  },
  "variant_b": {
    "total_predictions": 250,
    "total_feedback": 247,
    "accuracy": 0.8016,
    "avg_latency_ms": 15.2
  },
  "statistical_test": {
    "t_statistic": 2.45,
    "p_value": 0.0145,
    "significant": true
  },
  "winner": "B"
}
```

## Bonus Challenges

1. **Multi-Armed Bandit**: Implement Thompson Sampling or UCB
2. **Sequential Testing**: Implement early stopping rules
3. **Stratified Sampling**: Segment users by demographics
4. **Metric Guardrails**: Alert on degraded metrics
5. **Automated Rollout**: Gradual traffic increase for winner
6. **Multi-Variant**: Support A/B/C/D testing
7. **Bayesian Analysis**: Implement Bayesian hypothesis testing
8. **Cost-Benefit**: Factor in prediction costs
9. **Interaction Effects**: Test multiple features together
10. **Real-Time Dashboard**: Live experiment monitoring

## Resources

- [A/B Testing Guide](https://www.optimizely.com/optimization-glossary/ab-testing/)
- [Statistical Significance](https://www.statsmodels.org/stable/index.html)
- [Multi-Armed Bandits](https://arxiv.org/abs/1707.02038)

## Success Criteria

- [ ] Both models train successfully
- [ ] Traffic splits correctly (50/50 or custom)
- [ ] Users assigned consistently to same variant
- [ ] Predictions logged to database
- [ ] Feedback recorded correctly
- [ ] Statistical test calculates p-value
- [ ] Winner determination works
- [ ] Experiment lifecycle managed
- [ ] Results API returns metrics
- [ ] Performance differences detected
- [ ] User assignments persist in Redis
- [ ] Database schema supports experiments

## Project Structure

```
ab-testing-deployment/
├── models/
│   ├── model_a_baseline.joblib
│   ├── model_a_metadata.json
│   ├── model_b_challenger.joblib
│   └── model_b_metadata.json
├── app.py
├── experiment_manager.py
├── train_models.py
├── test_ab.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── README.md
```
