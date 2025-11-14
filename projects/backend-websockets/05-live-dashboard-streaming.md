# Project 5: Build Live Dashboard with Streaming Metrics

## Overview
Create a real-time analytics dashboard that streams live metrics, KPIs, and system health data using WebSockets. This project demonstrates building high-performance data streaming infrastructure for monitoring applications, admin panels, and real-time business intelligence dashboards.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Stream real-time metrics via WebSocket
- Implement efficient data aggregation
- Build time-series data streaming
- Handle high-frequency updates
- Create data buffering and batching
- Implement multiple metric types
- Build alert systems
- Handle data retention and sampling
- Create responsive dashboard UIs

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Database**: PostgreSQL + TimescaleDB (time-series)
- **Cache**: Redis (real-time metrics)
- **Metrics**: Prometheus client, custom collectors
- **Frontend**: Chart.js or D3.js
- **Data Processing**: Pandas, NumPy
- **Testing**: pytest-asyncio, load testing

## Project Requirements

### 1. Core Features
- Real-time metric streaming
- Multiple dashboard views
- Historical data queries
- Alert thresholds
- Data aggregation (1s, 1m, 1h, 1d)
- Custom metric definitions
- Dashboard sharing
- Export capabilities
- Mobile-responsive design

### 2. Metric Types
- Counter metrics (requests, errors)
- Gauge metrics (CPU, memory, active users)
- Histogram metrics (response times)
- Rate metrics (requests/second)
- Business metrics (revenue, conversions)
- Custom calculated metrics

### 3. WebSocket Endpoints
- `WS /ws/dashboard/{dashboard_id}` - Stream dashboard data
- `WS /ws/metrics/live` - Live metrics stream
- `GET /api/dashboards` - List dashboards
- `POST /api/dashboards` - Create dashboard
- `GET /api/metrics/history` - Historical data

### 4. Dashboard Features
- Multiple widgets
- Auto-refresh
- Time range selection
- Metric comparison
- Alert visualization
- Drill-down capabilities

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir live-dashboard
cd live-dashboard

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis
pip install pandas numpy
pip install prometheus-client
pip install pydantic pydantic-settings
pip install pytest pytest-asyncio

cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
redis==5.0.1
aioredis==2.0.1
pandas==2.1.3
numpy==1.26.0
prometheus-client==0.19.0
pydantic==2.5.0
python-jose[cryptography]==3.3.0
pytest==7.4.3
pytest-asyncio==0.21.1
psutil==5.9.6
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket,collectors}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,dashboard,metric,alert}.py

# Schemas
touch app/schemas/{__init__,dashboard,metric,websocket}.py

# Services
touch app/services/{__init__,metric_service,aggregation,alert_service}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,handlers,streamer}.py

# Collectors
touch app/collectors/{__init__,system_metrics,app_metrics,custom_metrics}.py

# Core
touch app/core/{__init__,redis,security}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/{dashboard.html,admin.html}
mkdir static/{js,css}
touch static/js/dashboard.js static/css/dashboard.css

# Tests
mkdir tests
touch tests/{test_streaming,test_aggregation,test_alerts}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "Live Dashboard"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@localhost/dashboard"
    # For SQLite: "sqlite+aiosqlite:///./dashboard.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Metrics
    METRICS_RETENTION_SECONDS: int = 86400  # 24 hours
    METRICS_UPDATE_INTERVAL: float = 1.0  # 1 second
    METRICS_BATCH_SIZE: int = 100
    ENABLE_SYSTEM_METRICS: bool = True

    # Aggregation
    AGGREGATION_INTERVALS: list = [1, 60, 3600, 86400]  # 1s, 1m, 1h, 1d

    # Streaming
    STREAM_BUFFER_SIZE: int = 1000
    STREAM_BATCH_INTERVAL: float = 0.1  # 100ms
    MAX_CONNECTIONS_PER_DASHBOARD: int = 100

    # Alerts
    ALERT_CHECK_INTERVAL: int = 5  # seconds
    ALERT_COOLDOWN: int = 300  # 5 minutes

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/dashboard.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, JSON, Boolean, Text
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Dashboard(Base):
    __tablename__ = "dashboards"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(255), nullable=False)
    description = Column(Text)
    owner_id = Column(Integer, ForeignKey("users.id"))
    is_public = Column(Boolean, default=False)
    layout = Column(JSON)  # Widget positions and sizes
    refresh_interval = Column(Integer, default=5)  # seconds
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    owner = relationship("User", back_populates="dashboards")
    widgets = relationship("Widget", back_populates="dashboard", cascade="all, delete-orphan")

class Widget(Base):
    __tablename__ = "widgets"

    id = Column(Integer, primary_key=True, index=True)
    dashboard_id = Column(Integer, ForeignKey("dashboards.id"), nullable=False)
    widget_type = Column(String(50), nullable=False)  # line_chart, bar_chart, gauge, counter
    title = Column(String(255), nullable=False)
    metric_id = Column(Integer, ForeignKey("metric_definitions.id"))
    config = Column(JSON)  # Chart config, colors, thresholds
    position = Column(JSON)  # {x, y, width, height}
    created_at = Column(DateTime, default=datetime.utcnow)

    dashboard = relationship("Dashboard", back_populates="widgets")
    metric = relationship("MetricDefinition")
```

**app/models/metric.py**:
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, JSON, Index
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class MetricDefinition(Base):
    __tablename__ = "metric_definitions"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, index=True)
    metric_type = Column(String(20), nullable=False)  # counter, gauge, histogram, rate
    description = Column(String(500))
    unit = Column(String(20))  # %, ms, MB, count, etc.
    tags = Column(JSON)  # Additional metadata
    aggregation_method = Column(String(20), default="avg")  # avg, sum, min, max, count
    created_at = Column(DateTime, default=datetime.utcnow)

class MetricData(Base):
    __tablename__ = "metric_data"
    __table_args__ = (
        Index('idx_metric_timestamp', 'metric_id', 'timestamp'),
    )

    id = Column(Integer, primary_key=True, index=True)
    metric_id = Column(Integer, ForeignKey("metric_definitions.id"), nullable=False)
    value = Column(Float, nullable=False)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)
    tags = Column(JSON)  # Additional dimensions

    metric = relationship("MetricDefinition")

class AggregatedMetric(Base):
    __tablename__ = "aggregated_metrics"
    __table_args__ = (
        Index('idx_agg_metric_interval_time', 'metric_id', 'interval_seconds', 'timestamp'),
    )

    id = Column(Integer, primary_key=True, index=True)
    metric_id = Column(Integer, ForeignKey("metric_definitions.id"), nullable=False)
    interval_seconds = Column(Integer, nullable=False)  # 60, 3600, 86400
    timestamp = Column(DateTime, nullable=False, index=True)
    avg_value = Column(Float)
    min_value = Column(Float)
    max_value = Column(Float)
    sum_value = Column(Float)
    count = Column(Integer)

    metric = relationship("MetricDefinition")
```

**app/models/alert.py**:
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Boolean, JSON
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Alert(Base):
    __tablename__ = "alerts"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(255), nullable=False)
    metric_id = Column(Integer, ForeignKey("metric_definitions.id"))
    condition = Column(String(20), nullable=False)  # gt, lt, eq, gte, lte
    threshold = Column(Float, nullable=False)
    severity = Column(String(20), default="warning")  # info, warning, error, critical
    is_active = Column(Boolean, default=True)
    notification_channels = Column(JSON)  # ["email", "slack", "webhook"]
    created_at = Column(DateTime, default=datetime.utcnow)

    metric = relationship("MetricDefinition")
    triggers = relationship("AlertTrigger", back_populates="alert")

class AlertTrigger(Base):
    __tablename__ = "alert_triggers"

    id = Column(Integer, primary_key=True, index=True)
    alert_id = Column(Integer, ForeignKey("alerts.id"))
    metric_value = Column(Float, nullable=False)
    triggered_at = Column(DateTime, default=datetime.utcnow)
    resolved_at = Column(DateTime, nullable=True)
    is_active = Column(Boolean, default=True)

    alert = relationship("Alert", back_populates="triggers")
```

### Step 5: System Metrics Collector (app/collectors/system_metrics.py)
```python
import psutil
import asyncio
from typing import Dict
from datetime import datetime

class SystemMetricsCollector:
    """Collects system-level metrics"""

    def __init__(self):
        self.metrics = {}

    async def collect(self) -> Dict[str, float]:
        """Collect all system metrics"""
        metrics = {}

        # CPU metrics
        metrics["system_cpu_percent"] = psutil.cpu_percent(interval=0.1)
        metrics["system_cpu_count"] = psutil.cpu_count()

        # Memory metrics
        mem = psutil.virtual_memory()
        metrics["system_memory_percent"] = mem.percent
        metrics["system_memory_used_mb"] = mem.used / (1024 * 1024)
        metrics["system_memory_available_mb"] = mem.available / (1024 * 1024)

        # Disk metrics
        disk = psutil.disk_usage('/')
        metrics["system_disk_percent"] = disk.percent
        metrics["system_disk_used_gb"] = disk.used / (1024 * 1024 * 1024)

        # Network metrics
        net = psutil.net_io_counters()
        metrics["system_network_bytes_sent"] = net.bytes_sent
        metrics["system_network_bytes_recv"] = net.bytes_recv

        return metrics

    async def start_collection(self, interval: float = 1.0):
        """Start periodic collection"""
        while True:
            metrics = await self.collect()
            self.metrics = metrics
            await asyncio.sleep(interval)

# Global collector
system_collector = SystemMetricsCollector()
```

### Step 6: Metric Service (app/services/metric_service.py)
```python
from typing import List, Dict, Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_, func
from datetime import datetime, timedelta
import json

from app.models.metric import MetricDefinition, MetricData, AggregatedMetric
from app.core.redis import redis_client

class MetricService:
    """Service for managing metrics"""

    @staticmethod
    async def record_metric(
        metric_name: str,
        value: float,
        tags: Optional[Dict] = None,
        db: AsyncSession = None,
        redis_only: bool = False
    ):
        """Record a metric value"""
        timestamp = datetime.utcnow()

        # Store in Redis for real-time access
        redis_key = f"metric:{metric_name}:latest"
        metric_data = {
            "value": value,
            "timestamp": timestamp.isoformat(),
            "tags": tags or {}
        }
        await redis_client.redis.setex(
            redis_key,
            300,  # 5 minute expiry
            json.dumps(metric_data)
        )

        # Add to time-series in Redis
        ts_key = f"metric:{metric_name}:timeseries"
        await redis_client.redis.zadd(
            ts_key,
            {json.dumps(metric_data): timestamp.timestamp()}
        )

        # Trim old data (keep last hour)
        cutoff = (timestamp - timedelta(hours=1)).timestamp()
        await redis_client.redis.zremrangebyscore(ts_key, 0, cutoff)

        # Optionally store in database
        if db and not redis_only:
            # Get or create metric definition
            result = await db.execute(
                select(MetricDefinition).where(MetricDefinition.name == metric_name)
            )
            metric_def = result.scalar_one_or_none()

            if not metric_def:
                metric_def = MetricDefinition(
                    name=metric_name,
                    metric_type="gauge"  # Default type
                )
                db.add(metric_def)
                await db.flush()

            # Store data point
            metric_data_point = MetricData(
                metric_id=metric_def.id,
                value=value,
                timestamp=timestamp,
                tags=tags
            )
            db.add(metric_data_point)
            await db.commit()

    @staticmethod
    async def get_latest_metrics(metric_names: List[str]) -> Dict[str, Dict]:
        """Get latest values for multiple metrics"""
        metrics = {}

        for name in metric_names:
            redis_key = f"metric:{name}:latest"
            data = await redis_client.redis.get(redis_key)

            if data:
                metrics[name] = json.loads(data)
            else:
                metrics[name] = None

        return metrics

    @staticmethod
    async def get_metric_history(
        metric_name: str,
        start_time: datetime,
        end_time: datetime,
        interval_seconds: Optional[int] = None,
        db: AsyncSession = None
    ) -> List[Dict]:
        """Get historical metric data"""
        # Try Redis first for recent data
        if not interval_seconds and (datetime.utcnow() - start_time).total_seconds() < 3600:
            return await MetricService._get_history_from_redis(
                metric_name,
                start_time,
                end_time
            )

        # Fall back to database for older/aggregated data
        if db:
            return await MetricService._get_history_from_db(
                metric_name,
                start_time,
                end_time,
                interval_seconds,
                db
            )

        return []

    @staticmethod
    async def _get_history_from_redis(
        metric_name: str,
        start_time: datetime,
        end_time: datetime
    ) -> List[Dict]:
        """Get metric history from Redis"""
        ts_key = f"metric:{metric_name}:timeseries"

        # Get data in time range
        data = await redis_client.redis.zrangebyscore(
            ts_key,
            start_time.timestamp(),
            end_time.timestamp(),
            withscores=True
        )

        history = []
        for item, score in data:
            metric_data = json.loads(item)
            history.append(metric_data)

        return history

    @staticmethod
    async def _get_history_from_db(
        metric_name: str,
        start_time: datetime,
        end_time: datetime,
        interval_seconds: Optional[int],
        db: AsyncSession
    ) -> List[Dict]:
        """Get metric history from database"""
        # Get metric definition
        result = await db.execute(
            select(MetricDefinition).where(MetricDefinition.name == metric_name)
        )
        metric_def = result.scalar_one_or_none()

        if not metric_def:
            return []

        if interval_seconds:
            # Get aggregated data
            query = select(AggregatedMetric).where(
                and_(
                    AggregatedMetric.metric_id == metric_def.id,
                    AggregatedMetric.interval_seconds == interval_seconds,
                    AggregatedMetric.timestamp >= start_time,
                    AggregatedMetric.timestamp <= end_time
                )
            ).order_by(AggregatedMetric.timestamp)
        else:
            # Get raw data
            query = select(MetricData).where(
                and_(
                    MetricData.metric_id == metric_def.id,
                    MetricData.timestamp >= start_time,
                    MetricData.timestamp <= end_time
                )
            ).order_by(MetricData.timestamp)

        result = await db.execute(query)
        data = result.scalars().all()

        history = []
        for point in data:
            if isinstance(point, AggregatedMetric):
                history.append({
                    "timestamp": point.timestamp.isoformat(),
                    "value": point.avg_value,
                    "min": point.min_value,
                    "max": point.max_value,
                    "count": point.count
                })
            else:
                history.append({
                    "timestamp": point.timestamp.isoformat(),
                    "value": point.value,
                    "tags": point.tags
                })

        return history
```

### Step 7: WebSocket Streamer (app/websocket/streamer.py)
```python
import asyncio
from typing import Set, Dict
from datetime import datetime
import json

from app.services.metric_service import MetricService
from app.collectors.system_metrics import system_collector

class MetricStreamer:
    """Streams metrics to connected clients"""

    def __init__(self):
        self.active_streams: Dict[str, asyncio.Task] = {}
        self.metric_subscriptions: Dict[str, Set[str]] = {}  # {connection_id: {metric_names}}

    async def start_stream(
        self,
        connection_id: str,
        metric_names: List[str],
        callback
    ):
        """Start streaming metrics to a connection"""
        self.metric_subscriptions[connection_id] = set(metric_names)

        # Create stream task
        task = asyncio.create_task(
            self._stream_metrics(connection_id, callback)
        )
        self.active_streams[connection_id] = task

    async def stop_stream(self, connection_id: str):
        """Stop streaming to a connection"""
        if connection_id in self.active_streams:
            task = self.active_streams[connection_id]
            task.cancel()
            try:
                await task
            except asyncio.CancelledError:
                pass

            del self.active_streams[connection_id]

        if connection_id in self.metric_subscriptions:
            del self.metric_subscriptions[connection_id]

    async def _stream_metrics(self, connection_id: str, callback):
        """Stream metrics periodically"""
        from app.config import settings

        try:
            while True:
                metric_names = list(self.metric_subscriptions.get(connection_id, set()))

                if not metric_names:
                    await asyncio.sleep(settings.STREAM_BATCH_INTERVAL)
                    continue

                # Get latest metrics
                metrics_data = await MetricService.get_latest_metrics(metric_names)

                # Add system metrics
                system_metrics = system_collector.metrics
                metrics_data.update({
                    f"system.{k}": {"value": v, "timestamp": datetime.utcnow().isoformat()}
                    for k, v in system_metrics.items()
                })

                # Send to callback
                await callback({
                    "type": "metrics_update",
                    "metrics": metrics_data,
                    "timestamp": datetime.utcnow().isoformat()
                })

                await asyncio.sleep(settings.STREAM_BATCH_INTERVAL)

        except asyncio.CancelledError:
            pass
        except Exception as e:
            print(f"Error in metric stream: {e}")

# Global streamer
metric_streamer = MetricStreamer()
```

### Step 8: WebSocket Handler (app/websocket/handlers.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Query
import json
import uuid
from datetime import datetime, timedelta

from app.websocket.streamer import metric_streamer
from app.services.metric_service import MetricService
from app.core.security import decode_access_token

async def handle_dashboard_stream(
    websocket: WebSocket,
    dashboard_id: int,
    token: str = Query(...)
):
    """Handle dashboard WebSocket streaming"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    connection_id = str(uuid.uuid4())

    await websocket.accept()

    # Send initial message
    await websocket.send_json({
        "type": "connected",
        "connection_id": connection_id,
        "dashboard_id": dashboard_id
    })

    subscribed_metrics = []

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "subscribe_metrics":
                # Subscribe to metrics
                metric_names = message.get("metrics", [])
                subscribed_metrics = metric_names

                # Start streaming
                async def send_update(data):
                    try:
                        await websocket.send_json(data)
                    except:
                        pass

                await metric_streamer.start_stream(
                    connection_id,
                    metric_names,
                    send_update
                )

            elif msg_type == "get_history":
                # Get historical data
                metric_name = message.get("metric")
                hours = message.get("hours", 1)

                end_time = datetime.utcnow()
                start_time = end_time - timedelta(hours=hours)

                history = await MetricService.get_metric_history(
                    metric_name,
                    start_time,
                    end_time
                )

                await websocket.send_json({
                    "type": "history",
                    "metric": metric_name,
                    "data": history
                })

            elif msg_type == "unsubscribe":
                # Stop streaming
                await metric_streamer.stop_stream(connection_id)
                subscribed_metrics = []

    except WebSocketDisconnect:
        await metric_streamer.stop_stream(connection_id)
    except Exception as e:
        print(f"Error in dashboard WebSocket: {e}")
        await metric_streamer.stop_stream(connection_id)
```

### Step 9: Frontend Dashboard (static/dashboard.html)
```html
<!DOCTYPE html>
<html>
<head>
    <title>Live Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: Arial, sans-serif; background: #f5f5f5; padding: 20px; }
        .dashboard { display: grid; grid-template-columns: repeat(auto-fit, minmax(400px, 1fr)); gap: 20px; }
        .widget { background: white; border-radius: 8px; padding: 20px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .widget h3 { margin-bottom: 15px; color: #333; }
        .metric-value { font-size: 48px; font-weight: bold; color: #0084ff; }
        .metric-label { font-size: 14px; color: #666; margin-top: 10px; }
        canvas { max-height: 300px; }
        .status { padding: 10px; background: #e8f5e9; border-radius: 4px; margin-bottom: 20px; }
        .status.error { background: #ffebee; }
    </style>
</head>
<body>
    <div id="status" class="status">Connecting...</div>

    <div class="dashboard">
        <div class="widget">
            <h3>CPU Usage</h3>
            <canvas id="cpuChart"></canvas>
        </div>

        <div class="widget">
            <h3>Memory Usage</h3>
            <canvas id="memoryChart"></canvas>
        </div>

        <div class="widget">
            <h3>Active Users</h3>
            <div class="metric-value" id="activeUsers">--</div>
            <div class="metric-label">Current</div>
        </div>

        <div class="widget">
            <h3>Requests/Second</h3>
            <canvas id="requestsChart"></canvas>
        </div>
    </div>

    <script>
        let ws = null;
        let charts = {};
        const maxDataPoints = 60;

        // Initialize charts
        function initCharts() {
            const chartConfig = {
                type: 'line',
                options: {
                    responsive: true,
                    maintainAspectRatio: true,
                    animation: { duration: 0 },
                    scales: {
                        y: { beginAtZero: true, max: 100 }
                    }
                }
            };

            charts.cpu = new Chart(
                document.getElementById('cpuChart'),
                {
                    ...chartConfig,
                    data: {
                        labels: [],
                        datasets: [{
                            label: 'CPU %',
                            data: [],
                            borderColor: '#0084ff',
                            tension: 0.4
                        }]
                    }
                }
            );

            charts.memory = new Chart(
                document.getElementById('memoryChart'),
                {
                    ...chartConfig,
                    data: {
                        labels: [],
                        datasets: [{
                            label: 'Memory %',
                            data: [],
                            borderColor: '#00c853',
                            tension: 0.4
                        }]
                    }
                }
            );

            charts.requests = new Chart(
                document.getElementById('requestsChart'),
                {
                    ...chartConfig,
                    data: {
                        labels: [],
                        datasets: [{
                            label: 'Requests/s',
                            data: [],
                            borderColor: '#ff9100',
                            tension: 0.4
                        }]
                    },
                    options: {
                        ...chartConfig.options,
                        scales: { y: { beginAtZero: true } }
                    }
                }
            );
        }

        function updateChart(chart, value, label) {
            chart.data.labels.push(label);
            chart.data.datasets[0].data.push(value);

            // Keep only last N points
            if (chart.data.labels.length > maxDataPoints) {
                chart.data.labels.shift();
                chart.data.datasets[0].data.shift();
            }

            chart.update('none');
        }

        function connect() {
            const token = localStorage.getItem('auth_token') || 'demo-token';
            ws = new WebSocket(`ws://localhost:8000/ws/dashboard/1?token=${token}`);

            ws.onopen = () => {
                document.getElementById('status').textContent = 'Connected';
                document.getElementById('status').className = 'status';

                // Subscribe to metrics
                ws.send(JSON.stringify({
                    type: 'subscribe_metrics',
                    metrics: [
                        'system_cpu_percent',
                        'system_memory_percent',
                        'active_users',
                        'requests_per_second'
                    ]
                }));
            };

            ws.onmessage = (event) => {
                const data = JSON.parse(event.data);

                if (data.type === 'metrics_update') {
                    handleMetricsUpdate(data.metrics);
                }
            };

            ws.onerror = () => {
                document.getElementById('status').textContent = 'Error';
                document.getElementById('status').className = 'status error';
            };

            ws.onclose = () => {
                document.getElementById('status').textContent = 'Disconnected';
                document.getElementById('status').className = 'status error';
                setTimeout(connect, 3000);
            };
        }

        function handleMetricsUpdate(metrics) {
            const now = new Date().toLocaleTimeString();

            // Update CPU chart
            if (metrics['system.system_cpu_percent']) {
                updateChart(
                    charts.cpu,
                    metrics['system.system_cpu_percent'].value,
                    now
                );
            }

            // Update Memory chart
            if (metrics['system.system_memory_percent']) {
                updateChart(
                    charts.memory,
                    metrics['system.system_memory_percent'].value,
                    now
                );
            }

            // Update active users
            if (metrics['active_users']) {
                document.getElementById('activeUsers').textContent =
                    Math.round(metrics['active_users'].value);
            }

            // Update requests chart
            if (metrics['requests_per_second']) {
                updateChart(
                    charts.requests,
                    metrics['requests_per_second'].value,
                    now
                );
            }
        }

        // Initialize
        initCharts();
        connect();
    </script>
</body>
</html>
```

### Step 10: Run the Application
```bash
# Start Redis
redis-server

# Start metric collection
python -c "from app.collectors.system_metrics import system_collector; import asyncio; asyncio.run(system_collector.start_collection())" &

# Start the application
uvicorn app.main:app --reload --port 8000

# Open dashboard
# http://localhost:8000/static/dashboard.html
```

## Expected Outputs

### 1. Metrics Update
```json
{
  "type": "metrics_update",
  "metrics": {
    "system_cpu_percent": {
      "value": 45.2,
      "timestamp": "2024-01-15T10:30:00"
    },
    "system_memory_percent": {
      "value": 62.8,
      "timestamp": "2024-01-15T10:30:00"
    }
  },
  "timestamp": "2024-01-15T10:30:00"
}
```

### 2. Historical Data
```json
{
  "type": "history",
  "metric": "cpu_usage",
  "data": [
    {"timestamp": "2024-01-15T10:29:00", "value": 42.1},
    {"timestamp": "2024-01-15T10:30:00", "value": 45.2}
  ]
}
```

## Bonus Challenges

- [ ] Add custom metric formulas
- [ ] Implement dashboard templates
- [ ] Create anomaly detection
- [ ] Add forecasting/predictions
- [ ] Implement metric correlations
- [ ] Create mobile app
- [ ] Add PDF/image export
- [ ] Implement dashboard sharing
- [ ] Create custom visualizations
- [ ] Add multi-tenant support
- [ ] Implement data sampling
- [ ] Create alert rules engine
- [ ] Add dashboard versioning
- [ ] Implement metric tagging
- [ ] Create custom aggregations

## Resources

- [TimescaleDB Documentation](https://docs.timescale.com/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Chart.js Documentation](https://www.chartjs.org/)
- [D3.js for Advanced Viz](https://d3js.org/)
- [Grafana Inspiration](https://grafana.com/)
- [WebSocket Performance](https://ably.com/topic/websocket-performance)

## Success Criteria

- [ ] Real-time metrics stream smoothly
- [ ] Multiple widgets update in sync
- [ ] Historical data queries work
- [ ] Charts render without lag
- [ ] Alerts trigger correctly
- [ ] Dashboard loads in < 2 seconds
- [ ] Handles 1000+ metrics
- [ ] Supports 100+ concurrent viewers
- [ ] Data aggregation accurate
- [ ] No memory leaks
- [ ] Mobile responsive design
- [ ] Export features work
- [ ] System handles high metric volume
- [ ] Dashboard customization works
