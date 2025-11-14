# Project 10: Build IoT Device Monitoring System

## Overview
Create a comprehensive IoT device monitoring and management platform that streams real-time telemetry data, handles device commands, monitors device health, and provides analytics dashboards. This project demonstrates building scalable IoT infrastructure with WebSocket communication, MQTT integration, time-series data handling, and edge computing patterns.

## Difficulty Level
Advanced

## Learning Objectives
- Stream real-time IoT telemetry data
- Implement bidirectional device communication
- Handle high-volume sensor data
- Build device fleet management
- Create anomaly detection systems
- Implement device provisioning
- Handle offline/online state
- Build time-series data storage
- Create real-time alerts and rules
- Implement device shadowing

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **IoT Protocol**: MQTT (Mosquitto) + WebSocket
- **Database**: PostgreSQL + TimescaleDB (time-series)
- **Cache**: Redis (device state, pub/sub)
- **Message Queue**: Celery, RabbitMQ
- **Analytics**: Pandas, NumPy
- **Monitoring**: Grafana, Prometheus
- **Testing**: pytest-asyncio, device simulation

## Project Requirements

### 1. Core Features
- Real-time telemetry streaming
- Device registration and provisioning
- Command and control (C2)
- Device status monitoring
- Alert and notification system
- Historical data queries
- Device grouping and tagging
- Firmware update management
- Device shadows (desired/reported state)

### 2. Device Types
- Temperature/humidity sensors
- Motion detectors
- Smart cameras
- Industrial equipment
- Environmental monitors
- Actuators and controllers
- GPS trackers
- Energy meters

### 3. WebSocket Endpoints
- `WS /ws/device/{device_id}` - Single device stream
- `WS /ws/fleet` - Fleet-wide monitoring
- `WS /ws/telemetry/{group_id}` - Group telemetry
- `POST /api/devices/register` - Device registration
- `POST /api/devices/{id}/command` - Send command
- `GET /api/devices/{id}/telemetry` - Historical data

### 4. Telemetry Data
- Sensor readings (temperature, pressure, etc.)
- Device health metrics
- Battery levels
- Network connectivity
- GPS coordinates
- Error logs
- Performance metrics

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir iot-monitoring-system
cd iot-monitoring-system

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis
pip install celery
pip install paho-mqtt  # MQTT client
pip install pandas numpy
pip install pydantic pydantic-settings
pip install python-jose[cryptography]
pip install pytest pytest-asyncio

cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
redis==5.0.1
celery==5.3.4
paho-mqtt==1.6.1
pandas==2.1.3
numpy==1.26.0
pydantic==2.5.0
python-jose[cryptography]==3.3.0
pytest==7.4.3
pytest-asyncio==0.21.1
python-dateutil==2.8.2
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket,mqtt,tasks}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,device,telemetry,alert}.py

# Schemas
touch app/schemas/{__init__,device,telemetry,websocket}.py

# Services
touch app/services/{__init__,device_service,telemetry_service,alert_service,analytics}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,device_handler}.py

# MQTT
touch app/mqtt/{__init__,client,message_handler}.py

# Tasks
touch app/tasks/{__init__,aggregation,monitoring}.py

# Core
touch app/core/{__init__,redis,security,celery}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Device simulator
mkdir simulator
touch simulator/{__init__,device_sim,sensors}.py

# Frontend
mkdir static
touch static/{dashboard.html,device-detail.html,fleet-map.html}

# Tests
mkdir tests
touch tests/{test_telemetry,test_commands,test_alerts}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "IoT Monitoring System"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@localhost/iot"
    # For SQLite: "sqlite+aiosqlite:///./iot.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Celery
    CELERY_BROKER_URL: str = "redis://localhost:6379/1"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/2"

    # MQTT
    MQTT_BROKER: str = "localhost"
    MQTT_PORT: int = 1883
    MQTT_USERNAME: str = ""
    MQTT_PASSWORD: str = ""
    MQTT_TOPIC_PREFIX: str = "iot"

    # Device Settings
    MAX_DEVICES_PER_USER: int = 1000
    DEVICE_OFFLINE_THRESHOLD: int = 300  # 5 minutes
    TELEMETRY_RETENTION_DAYS: int = 90

    # Telemetry
    TELEMETRY_BATCH_SIZE: int = 100
    TELEMETRY_BATCH_INTERVAL: float = 1.0  # 1 second
    MAX_TELEMETRY_RATE: int = 100  # messages per second per device

    # Alerts
    ALERT_CHECK_INTERVAL: int = 10  # seconds
    MAX_ALERTS_PER_DEVICE: int = 50

    # Aggregation
    AGGREGATION_INTERVALS: list = [60, 300, 3600, 86400]  # 1m, 5m, 1h, 1d

    # WebSocket
    PING_INTERVAL: int = 30
    MAX_CONNECTIONS: int = 10000

    # Security
    DEVICE_API_KEY_LENGTH: int = 32
    JWT_SECRET_KEY: str = "change-this-secret-key"
    JWT_ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/device.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Boolean, JSON, Float
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base
import secrets

class Device(Base):
    __tablename__ = "devices"

    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(String(100), unique=True, index=True, nullable=False)
    name = Column(String(255), nullable=False)
    device_type = Column(String(50), nullable=False)  # sensor, actuator, gateway
    manufacturer = Column(String(100))
    model = Column(String(100))
    firmware_version = Column(String(50))

    # Owner
    user_id = Column(Integer, ForeignKey("users.id"))

    # Authentication
    api_key = Column(String(100), unique=True, index=True)

    # Status
    status = Column(String(20), default="offline")  # online, offline, error
    is_active = Column(Boolean, default=True)
    last_seen = Column(DateTime, nullable=True)

    # Location
    latitude = Column(Float, nullable=True)
    longitude = Column(Float, nullable=True)
    location_name = Column(String(255))

    # Metadata
    tags = Column(JSON, default=[])
    metadata = Column(JSON, default={})

    # Shadow state
    desired_state = Column(JSON, default={})
    reported_state = Column(JSON, default={})

    # Timestamps
    provisioned_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    user = relationship("User", back_populates="devices")
    telemetry = relationship("TelemetryData", back_populates="device")
    alerts = relationship("DeviceAlert", back_populates="device")
    commands = relationship("DeviceCommand", back_populates="device")

    def generate_api_key(self):
        """Generate API key for device"""
        self.api_key = secrets.token_urlsafe(32)
        return self.api_key

class DeviceGroup(Base):
    __tablename__ = "device_groups"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    description = Column(String(500))
    user_id = Column(Integer, ForeignKey("users.id"))
    device_ids = Column(JSON, default=[])
    created_at = Column(DateTime, default=datetime.utcnow)
```

**app/models/telemetry.py**:
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, JSON, Index
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class TelemetryData(Base):
    __tablename__ = "telemetry_data"
    __table_args__ = (
        Index('idx_device_timestamp', 'device_id', 'timestamp'),
        Index('idx_metric_timestamp', 'metric_name', 'timestamp'),
    )

    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(String(100), ForeignKey("devices.device_id"), nullable=False, index=True)
    metric_name = Column(String(100), nullable=False, index=True)
    value = Column(Float, nullable=False)
    unit = Column(String(20))
    timestamp = Column(DateTime, nullable=False, index=True)

    # Additional data
    tags = Column(JSON, default={})

    device = relationship("Device", back_populates="telemetry")

class AggregatedTelemetry(Base):
    __tablename__ = "aggregated_telemetry"
    __table_args__ = (
        Index('idx_agg_device_metric', 'device_id', 'metric_name', 'interval_seconds', 'timestamp'),
    )

    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(String(100), nullable=False, index=True)
    metric_name = Column(String(100), nullable=False)
    interval_seconds = Column(Integer, nullable=False)  # 60, 300, 3600, 86400
    timestamp = Column(DateTime, nullable=False, index=True)

    avg_value = Column(Float)
    min_value = Column(Float)
    max_value = Column(Float)
    sum_value = Column(Float)
    count = Column(Integer)

class DeviceCommand(Base):
    __tablename__ = "device_commands"

    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(String(100), ForeignKey("devices.device_id"), index=True)
    command = Column(String(100), nullable=False)
    parameters = Column(JSON, default={})
    status = Column(String(20), default="pending")  # pending, sent, executed, failed
    result = Column(JSON)
    sent_at = Column(DateTime, default=datetime.utcnow)
    executed_at = Column(DateTime, nullable=True)

    device = relationship("Device", back_populates="commands")
```

**app/models/alert.py**:
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Boolean, JSON
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class AlertRule(Base):
    __tablename__ = "alert_rules"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    device_id = Column(String(100), ForeignKey("devices.device_id"))
    device_group_id = Column(Integer, ForeignKey("device_groups.id"), nullable=True)

    # Rule definition
    metric_name = Column(String(100), nullable=False)
    condition = Column(String(20), nullable=False)  # gt, lt, eq, gte, lte
    threshold = Column(Float, nullable=False)
    duration_seconds = Column(Integer, default=0)  # Alert if condition met for duration

    # Actions
    severity = Column(String(20), default="warning")  # info, warning, error, critical
    notification_channels = Column(JSON, default=["websocket"])

    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)

class DeviceAlert(Base):
    __tablename__ = "device_alerts"

    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(String(100), ForeignKey("devices.device_id"), index=True)
    rule_id = Column(Integer, ForeignKey("alert_rules.id"))
    severity = Column(String(20), nullable=False)
    message = Column(String(500), nullable=False)
    metric_value = Column(Float)
    is_active = Column(Boolean, default=True)
    triggered_at = Column(DateTime, default=datetime.utcnow, index=True)
    resolved_at = Column(DateTime, nullable=True)

    device = relationship("Device", back_populates="alerts")
    rule = relationship("AlertRule")
```

### Step 5: Telemetry Service (app/services/telemetry_service.py)
```python
from typing import List, Dict, Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_, func
from datetime import datetime, timedelta
import json

from app.models.telemetry import TelemetryData, AggregatedTelemetry
from app.core.redis import redis_client

class TelemetryService:
    """Service for managing telemetry data"""

    @staticmethod
    async def record_telemetry(
        device_id: str,
        metric_name: str,
        value: float,
        unit: Optional[str],
        timestamp: datetime,
        tags: Optional[Dict],
        db: AsyncSession
    ):
        """Record telemetry data point"""
        # Store in database
        telemetry = TelemetryData(
            device_id=device_id,
            metric_name=metric_name,
            value=value,
            unit=unit,
            timestamp=timestamp,
            tags=tags or {}
        )
        db.add(telemetry)
        await db.commit()

        # Cache latest value in Redis
        await TelemetryService._cache_latest(device_id, metric_name, value, timestamp)

        # Add to time-series in Redis
        await TelemetryService._add_to_timeseries(device_id, metric_name, value, timestamp)

    @staticmethod
    async def _cache_latest(
        device_id: str,
        metric_name: str,
        value: float,
        timestamp: datetime
    ):
        """Cache latest telemetry value"""
        key = f"telemetry:latest:{device_id}:{metric_name}"
        data = {
            "value": value,
            "timestamp": timestamp.isoformat()
        }
        await redis_client.redis.setex(
            key,
            3600,  # 1 hour
            json.dumps(data)
        )

    @staticmethod
    async def _add_to_timeseries(
        device_id: str,
        metric_name: str,
        value: float,
        timestamp: datetime
    ):
        """Add to Redis time-series"""
        key = f"telemetry:ts:{device_id}:{metric_name}"
        await redis_client.redis.zadd(
            key,
            {json.dumps({"value": value}): timestamp.timestamp()}
        )

        # Keep only last 24 hours
        cutoff = (datetime.utcnow() - timedelta(hours=24)).timestamp()
        await redis_client.redis.zremrangebyscore(key, 0, cutoff)

    @staticmethod
    async def get_latest_telemetry(device_id: str) -> Dict:
        """Get all latest telemetry for device"""
        pattern = f"telemetry:latest:{device_id}:*"
        keys = await redis_client.redis.keys(pattern)

        telemetry = {}
        for key in keys:
            metric_name = key.split(":")[-1]
            data = await redis_client.redis.get(key)
            if data:
                telemetry[metric_name] = json.loads(data)

        return telemetry

    @staticmethod
    async def get_telemetry_history(
        device_id: str,
        metric_name: str,
        start_time: datetime,
        end_time: datetime,
        db: AsyncSession
    ) -> List[Dict]:
        """Get historical telemetry data"""
        result = await db.execute(
            select(TelemetryData)
            .where(
                and_(
                    TelemetryData.device_id == device_id,
                    TelemetryData.metric_name == metric_name,
                    TelemetryData.timestamp >= start_time,
                    TelemetryData.timestamp <= end_time
                )
            )
            .order_by(TelemetryData.timestamp)
        )
        data = result.scalars().all()

        return [
            {
                "timestamp": point.timestamp.isoformat(),
                "value": point.value,
                "unit": point.unit
            }
            for point in data
        ]

    @staticmethod
    async def get_aggregated_telemetry(
        device_id: str,
        metric_name: str,
        interval_seconds: int,
        start_time: datetime,
        end_time: datetime,
        db: AsyncSession
    ) -> List[Dict]:
        """Get aggregated telemetry data"""
        result = await db.execute(
            select(AggregatedTelemetry)
            .where(
                and_(
                    AggregatedTelemetry.device_id == device_id,
                    AggregatedTelemetry.metric_name == metric_name,
                    AggregatedTelemetry.interval_seconds == interval_seconds,
                    AggregatedTelemetry.timestamp >= start_time,
                    AggregatedTelemetry.timestamp <= end_time
                )
            )
            .order_by(AggregatedTelemetry.timestamp)
        )
        data = result.scalars().all()

        return [
            {
                "timestamp": point.timestamp.isoformat(),
                "avg": point.avg_value,
                "min": point.min_value,
                "max": point.max_value,
                "count": point.count
            }
            for point in data
        ]
```

### Step 6: MQTT Client (app/mqtt/client.py)
```python
import paho.mqtt.client as mqtt
import json
import asyncio
from typing import Callable

from app.config import settings

class MQTTClient:
    """MQTT client for device communication"""

    def __init__(self):
        self.client = None
        self.is_connected = False
        self.message_callbacks = []

    def connect(self):
        """Connect to MQTT broker"""
        self.client = mqtt.Client()

        # Set callbacks
        self.client.on_connect = self._on_connect
        self.client.on_message = self._on_message
        self.client.on_disconnect = self._on_disconnect

        # Set credentials if provided
        if settings.MQTT_USERNAME and settings.MQTT_PASSWORD:
            self.client.username_pw_set(
                settings.MQTT_USERNAME,
                settings.MQTT_PASSWORD
            )

        # Connect
        self.client.connect(settings.MQTT_BROKER, settings.MQTT_PORT, 60)
        self.client.loop_start()

    def _on_connect(self, client, userdata, flags, rc):
        """Callback when connected"""
        if rc == 0:
            print("Connected to MQTT broker")
            self.is_connected = True

            # Subscribe to all device topics
            topic = f"{settings.MQTT_TOPIC_PREFIX}/+/telemetry"
            self.client.subscribe(topic)
            print(f"Subscribed to {topic}")

        else:
            print(f"Failed to connect, return code {rc}")

    def _on_message(self, client, userdata, msg):
        """Callback when message received"""
        try:
            topic = msg.topic
            payload = json.loads(msg.payload.decode())

            # Extract device ID from topic
            parts = topic.split("/")
            device_id = parts[1] if len(parts) > 1 else None

            # Call registered callbacks
            for callback in self.message_callbacks:
                try:
                    callback(device_id, payload)
                except Exception as e:
                    print(f"Error in message callback: {e}")

        except Exception as e:
            print(f"Error processing MQTT message: {e}")

    def _on_disconnect(self, client, userdata, rc):
        """Callback when disconnected"""
        print("Disconnected from MQTT broker")
        self.is_connected = False

    def register_callback(self, callback: Callable):
        """Register message callback"""
        self.message_callbacks.append(callback)

    def publish_command(self, device_id: str, command: str, parameters: dict):
        """Publish command to device"""
        topic = f"{settings.MQTT_TOPIC_PREFIX}/{device_id}/commands"
        payload = {
            "command": command,
            "parameters": parameters
        }
        self.client.publish(topic, json.dumps(payload))

    def disconnect(self):
        """Disconnect from broker"""
        if self.client:
            self.client.loop_stop()
            self.client.disconnect()

# Global MQTT client
mqtt_client = MQTTClient()
```

### Step 7: WebSocket Handler (app/websocket/device_handler.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json
import asyncio

from app.websocket.connection_manager import iot_manager
from app.services.telemetry_service import TelemetryService
from app.services.device_service import DeviceService
from app.models.device import Device
from app.core.security import decode_access_token
from app.db.session import get_db

async def handle_device_stream(
    websocket: WebSocket,
    device_id: str,
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
):
    """Handle device telemetry stream"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    user_id = int(payload.get("sub"))

    # Get device
    result = await db.execute(
        select(Device).where(Device.device_id == device_id)
    )
    device = result.scalar_one_or_none()

    if not device:
        await websocket.close(code=1008, reason="Device not found")
        return

    # Check permission
    if device.user_id != user_id:
        await websocket.close(code=1008, reason="Unauthorized")
        return

    await websocket.accept()

    # Connect to manager
    await iot_manager.connect(device_id, user_id, websocket)

    # Send initial device state
    telemetry = await TelemetryService.get_latest_telemetry(device_id)

    await websocket.send_json({
        "type": "device_state",
        "device_id": device_id,
        "status": device.status,
        "telemetry": telemetry,
        "desired_state": device.desired_state,
        "reported_state": device.reported_state
    })

    # Start telemetry streaming
    stream_task = asyncio.create_task(
        stream_telemetry(websocket, device_id)
    )

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "send_command":
                # Send command to device
                command = message.get("command")
                parameters = message.get("parameters", {})

                await DeviceService.send_command(
                    device_id,
                    command,
                    parameters,
                    db
                )

                await websocket.send_json({
                    "type": "command_sent",
                    "command": command
                })

            elif msg_type == "update_desired_state":
                # Update desired state
                desired_state = message.get("state")
                await DeviceService.update_desired_state(
                    device_id,
                    desired_state,
                    db
                )

    except WebSocketDisconnect:
        iot_manager.disconnect(device_id, user_id)
        stream_task.cancel()

async def stream_telemetry(websocket: WebSocket, device_id: str):
    """Stream telemetry updates"""
    try:
        while True:
            # Get latest telemetry
            telemetry = await TelemetryService.get_latest_telemetry(device_id)

            if telemetry:
                await websocket.send_json({
                    "type": "telemetry_update",
                    "data": telemetry
                })

            await asyncio.sleep(1)

    except asyncio.CancelledError:
        pass
```

### Step 8: Device Simulator (simulator/device_sim.py)
```python
import paho.mqtt.client as mqtt
import json
import time
import random
from datetime import datetime

class DeviceSimulator:
    """Simulates IoT device sending telemetry"""

    def __init__(self, device_id: str, device_type: str):
        self.device_id = device_id
        self.device_type = device_type
        self.client = mqtt.Client()
        self.is_running = False

    def connect(self, broker: str = "localhost", port: int = 1883):
        """Connect to MQTT broker"""
        self.client.connect(broker, port, 60)
        self.client.loop_start()
        print(f"Device {self.device_id} connected")

    def start_telemetry(self, interval: float = 5.0):
        """Start sending telemetry"""
        self.is_running = True

        while self.is_running:
            telemetry = self.generate_telemetry()
            self.send_telemetry(telemetry)
            time.sleep(interval)

    def generate_telemetry(self) -> dict:
        """Generate simulated telemetry data"""
        if self.device_type == "temperature_sensor":
            return {
                "temperature": round(random.uniform(18.0, 28.0), 2),
                "humidity": round(random.uniform(30.0, 70.0), 2),
                "timestamp": datetime.utcnow().isoformat()
            }
        elif self.device_type == "motion_sensor":
            return {
                "motion_detected": random.choice([True, False]),
                "timestamp": datetime.utcnow().isoformat()
            }
        else:
            return {
                "value": random.uniform(0, 100),
                "timestamp": datetime.utcnow().isoformat()
            }

    def send_telemetry(self, data: dict):
        """Send telemetry to broker"""
        topic = f"iot/{self.device_id}/telemetry"
        self.client.publish(topic, json.dumps(data))
        print(f"Sent telemetry: {data}")

    def stop(self):
        """Stop simulator"""
        self.is_running = False
        self.client.loop_stop()
        self.client.disconnect()

if __name__ == "__main__":
    # Simulate temperature sensor
    device = DeviceSimulator("temp-sensor-001", "temperature_sensor")
    device.connect()

    try:
        device.start_telemetry(interval=2.0)
    except KeyboardInterrupt:
        device.stop()
        print("Simulator stopped")
```

### Step 9: Run Application
```bash
# Install and start Mosquitto MQTT broker
# On Ubuntu: sudo apt-get install mosquitto
# On Mac: brew install mosquitto
mosquitto

# Start Redis
redis-server

# Start Celery worker
celery -A app.core.celery worker --loglevel=info

# Start application
uvicorn app.main:app --reload --port 8000

# Run device simulator (in another terminal)
python simulator/device_sim.py

# Open dashboard
# http://localhost:8000/static/dashboard.html
```

## Expected Outputs

### 1. Telemetry Update
```json
{
  "type": "telemetry_update",
  "data": {
    "temperature": {
      "value": 23.5,
      "timestamp": "2024-01-15T14:30:00"
    },
    "humidity": {
      "value": 45.2,
      "timestamp": "2024-01-15T14:30:00"
    }
  }
}
```

### 2. Device State
```json
{
  "type": "device_state",
  "device_id": "temp-sensor-001",
  "status": "online",
  "telemetry": {...},
  "desired_state": {"interval": 5},
  "reported_state": {"interval": 5, "battery": 85}
}
```

## Bonus Challenges

- [ ] Add edge computing capabilities
- [ ] Implement device twins
- [ ] Create predictive maintenance
- [ ] Add machine learning anomaly detection
- [ ] Implement OTA firmware updates
- [ ] Create geofencing alerts
- [ ] Add device certificate authentication
- [ ] Implement data retention policies
- [ ] Create custom dashboards
- [ ] Add webhook integrations
- [ ] Implement data export
- [ ] Create mobile app
- [ ] Add voice control integration
- [ ] Implement blockchain logging
- [ ] Create digital twins

## Resources

- [MQTT Protocol](https://mqtt.org/)
- [Paho MQTT Client](https://www.eclipse.org/paho/)
- [TimescaleDB](https://docs.timescale.com/)
- [AWS IoT Core](https://aws.amazon.com/iot-core/)
- [Azure IoT Hub](https://azure.microsoft.com/en-us/services/iot-hub/)
- [Grafana for IoT](https://grafana.com/grafana/dashboards/)

## Success Criteria

- [ ] Devices connect and authenticate
- [ ] Telemetry streams in real-time
- [ ] Commands sent to devices successfully
- [ ] Alert rules trigger correctly
- [ ] Dashboard displays live data
- [ ] Historical queries work
- [ ] Device status tracked accurately
- [ ] Handles 1000+ concurrent devices
- [ ] Data aggregation accurate
- [ ] System handles device offline/online
- [ ] MQTT and WebSocket integration works
- [ ] Time-series queries performant
- [ ] Alert latency < 5 seconds
- [ ] Firmware updates successful
