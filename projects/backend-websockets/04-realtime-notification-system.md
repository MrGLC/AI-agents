# Project 4: Build Real-time Notification System

## Overview
Create a production-ready real-time notification system that delivers instant updates to users across multiple devices and channels. This project demonstrates building a scalable notification infrastructure with WebSocket, push notifications, email integration, and user preferences management.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Build multi-channel notification delivery (WebSocket, Push, Email, SMS)
- Implement notification routing and prioritization
- Handle user preferences and subscriptions
- Create notification templates and personalization
- Build notification queue and retry mechanisms
- Implement read/unread tracking
- Scale notifications with Redis pub/sub
- Handle offline users and notification persistence
- Create notification analytics

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Database**: PostgreSQL (notifications, preferences)
- **Cache/Queue**: Redis (real-time delivery, queuing)
- **Message Queue**: Celery (background tasks)
- **Push Notifications**: Firebase Cloud Messaging (FCM)
- **Email**: SendGrid or AWS SES
- **Frontend**: React or vanilla JavaScript
- **Testing**: pytest-asyncio, MockRedis

## Project Requirements

### 1. Core Features
- Real-time WebSocket notifications
- Push notifications (mobile/web)
- Email notifications
- SMS notifications (optional)
- User notification preferences
- Notification templates
- Priority levels
- Read/unread tracking
- Notification history
- Batch notifications

### 2. WebSocket Endpoints
- `WS /ws/notifications` - Real-time notification stream
- `GET /api/notifications` - Get notification history
- `PATCH /api/notifications/{id}/read` - Mark as read
- `POST /api/notifications/send` - Send notification
- `GET /api/preferences` - Get user preferences
- `PUT /api/preferences` - Update preferences

### 3. Notification Types
- System notifications
- User mentions
- Activity updates
- Marketing messages
- Security alerts
- Transactional notifications
- Social interactions

### 4. Delivery Channels
- WebSocket (real-time)
- Push notifications
- Email
- SMS
- In-app banners

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir notification-system
cd notification-system

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis celery
pip install pydantic pydantic-settings
pip install python-jose[cryptography]
pip install sendgrid firebase-admin
pip install pytest pytest-asyncio

cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
aiosqlite==0.19.0
redis==5.0.1
aioredis==2.0.1
celery==5.3.4
python-jose[cryptography]==3.3.0
pydantic==2.5.0
pydantic-settings==2.1.0
sendgrid==6.10.0
firebase-admin==6.2.0
pytest==7.4.3
pytest-asyncio==0.21.1
jinja2==3.1.2
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket,templates}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,notification,user,preference}.py

# Schemas
touch app/schemas/{__init__,notification,websocket}.py

# Services
touch app/services/{__init__,notification_service,email_service,push_service}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,handlers}.py

# Core
touch app/core/{__init__,redis,security,celery}.py

# Templates
touch app/templates/{email_notification.html,welcome_email.html}

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# API
touch app/api/{__init__,notifications,preferences}.py

# Frontend
mkdir static
touch static/notifications.html

# Tests
mkdir tests
touch tests/{test_notifications,test_delivery,test_preferences}.py

# Worker
touch worker.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    APP_NAME: str = "Real-time Notification System"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./notifications.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Celery
    CELERY_BROKER_URL: str = "redis://localhost:6379/1"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/2"

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440

    # Email (SendGrid)
    SENDGRID_API_KEY: Optional[str] = None
    FROM_EMAIL: str = "notifications@example.com"
    FROM_NAME: str = "Notification System"

    # Firebase (Push Notifications)
    FIREBASE_CREDENTIALS_PATH: Optional[str] = None

    # SMS (Twilio)
    TWILIO_ACCOUNT_SID: Optional[str] = None
    TWILIO_AUTH_TOKEN: Optional[str] = None
    TWILIO_PHONE_NUMBER: Optional[str] = None

    # Notification Settings
    MAX_NOTIFICATIONS_PER_USER: int = 1000
    NOTIFICATION_RETENTION_DAYS: int = 30
    BATCH_SIZE: int = 100
    MAX_RETRY_ATTEMPTS: int = 3

    # Rate Limiting
    NOTIFICATION_RATE_LIMIT: int = 100  # per minute per user

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/notification.py**:
```python
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Boolean, JSON
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Notification(Base):
    __tablename__ = "notifications"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False, index=True)
    title = Column(String(255), nullable=False)
    message = Column(Text, nullable=False)
    notification_type = Column(String(50), nullable=False)  # system, mention, activity, etc.
    priority = Column(String(20), default="normal")  # low, normal, high, urgent
    category = Column(String(50), index=True)

    # Delivery
    channels = Column(JSON)  # ["websocket", "push", "email"]
    delivered_via = Column(JSON, default=[])
    delivery_status = Column(String(20), default="pending")  # pending, sent, failed

    # Tracking
    is_read = Column(Boolean, default=False, index=True)
    read_at = Column(DateTime, nullable=True)

    # Metadata
    data = Column(JSON)  # Additional data for the notification
    action_url = Column(String(500))  # URL for notification action

    # Timing
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    scheduled_for = Column(DateTime, nullable=True)
    expires_at = Column(DateTime, nullable=True)

    # Retry
    retry_count = Column(Integer, default=0)
    last_retry_at = Column(DateTime, nullable=True)

    # Relationships
    user = relationship("User", back_populates="notifications")

class NotificationTemplate(Base):
    __tablename__ = "notification_templates"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, index=True)
    notification_type = Column(String(50), nullable=False)
    title_template = Column(String(255), nullable=False)
    message_template = Column(Text, nullable=False)
    email_template = Column(Text)
    channels = Column(JSON)  # Default channels for this template
    priority = Column(String(20), default="normal")
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

class UserPreference(Base):
    __tablename__ = "user_preferences"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)

    # Channel preferences
    websocket_enabled = Column(Boolean, default=True)
    push_enabled = Column(Boolean, default=True)
    email_enabled = Column(Boolean, default=True)
    sms_enabled = Column(Boolean, default=False)

    # Type preferences (JSON: {type: enabled})
    type_preferences = Column(JSON, default={})

    # Quiet hours
    quiet_hours_enabled = Column(Boolean, default=False)
    quiet_hours_start = Column(String(5))  # "22:00"
    quiet_hours_end = Column(String(5))  # "08:00"

    # Push tokens
    push_tokens = Column(JSON, default=[])  # FCM tokens

    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    user = relationship("User", back_populates="notification_preferences")
```

### Step 5: Pydantic Schemas

**app/schemas/notification.py**:
```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime

class NotificationBase(BaseModel):
    title: str = Field(..., max_length=255)
    message: str
    notification_type: str
    priority: str = "normal"
    category: Optional[str] = None
    channels: List[str] = ["websocket"]
    data: Optional[Dict[str, Any]] = None
    action_url: Optional[str] = None
    scheduled_for: Optional[datetime] = None
    expires_at: Optional[datetime] = None

class NotificationCreate(NotificationBase):
    user_ids: List[int]  # Can send to multiple users

class NotificationResponse(NotificationBase):
    id: int
    user_id: int
    is_read: bool
    read_at: Optional[datetime]
    delivery_status: str
    delivered_via: List[str]
    created_at: datetime

    class Config:
        from_attributes = True

class NotificationUpdate(BaseModel):
    is_read: Optional[bool] = None

class NotificationList(BaseModel):
    notifications: List[NotificationResponse]
    total: int
    unread_count: int

class UserPreferenceUpdate(BaseModel):
    websocket_enabled: Optional[bool] = None
    push_enabled: Optional[bool] = None
    email_enabled: Optional[bool] = None
    sms_enabled: Optional[bool] = None
    type_preferences: Optional[Dict[str, bool]] = None
    quiet_hours_enabled: Optional[bool] = None
    quiet_hours_start: Optional[str] = None
    quiet_hours_end: Optional[str] = None
    push_tokens: Optional[List[str]] = None
```

**app/schemas/websocket.py**:
```python
from pydantic import BaseModel
from typing import Optional, Dict, Any, Literal
from datetime import datetime

class WSNotification(BaseModel):
    """WebSocket notification message"""
    type: Literal["notification"] = "notification"
    id: int
    title: str
    message: str
    notification_type: str
    priority: str
    category: Optional[str] = None
    data: Optional[Dict[str, Any]] = None
    action_url: Optional[str] = None
    created_at: datetime

class WSNotificationRead(BaseModel):
    """Mark notification as read"""
    type: Literal["mark_read"] = "mark_read"
    notification_id: int

class WSStatus(BaseModel):
    """Status message"""
    type: Literal["status"] = "status"
    message: str
    unread_count: int
```

### Step 6: Notification Service (app/services/notification_service.py)
```python
from typing import List, Optional, Dict, Any
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_, or_, func
from datetime import datetime, timedelta
import json

from app.models.notification import Notification, NotificationTemplate, UserPreference
from app.schemas.notification import NotificationCreate
from app.core.redis import redis_client
from app.services.email_service import send_email_notification
from app.services.push_service import send_push_notification

class NotificationService:
    """Service for managing notifications"""

    @staticmethod
    async def create_notification(
        notification_data: NotificationCreate,
        db: AsyncSession
    ) -> List[Notification]:
        """Create and send notifications to multiple users"""
        notifications = []

        for user_id in notification_data.user_ids:
            # Get user preferences
            prefs = await NotificationService._get_user_preferences(user_id, db)

            # Filter channels based on preferences
            channels = NotificationService._filter_channels(
                notification_data.channels,
                prefs,
                notification_data.notification_type
            )

            if not channels:
                continue  # Skip if all channels disabled

            # Create notification
            notification = Notification(
                user_id=user_id,
                title=notification_data.title,
                message=notification_data.message,
                notification_type=notification_data.notification_type,
                priority=notification_data.priority,
                category=notification_data.category,
                channels=channels,
                data=notification_data.data,
                action_url=notification_data.action_url,
                scheduled_for=notification_data.scheduled_for,
                expires_at=notification_data.expires_at
            )

            db.add(notification)
            await db.flush()
            await db.refresh(notification)

            notifications.append(notification)

            # Deliver immediately if not scheduled
            if not notification_data.scheduled_for:
                await NotificationService._deliver_notification(
                    notification,
                    prefs,
                    db
                )

        await db.commit()
        return notifications

    @staticmethod
    async def _deliver_notification(
        notification: Notification,
        preferences: Optional[UserPreference],
        db: AsyncSession
    ):
        """Deliver notification via configured channels"""
        delivered_via = []

        # WebSocket delivery
        if "websocket" in notification.channels:
            success = await NotificationService._deliver_via_websocket(notification)
            if success:
                delivered_via.append("websocket")

        # Push notification
        if "push" in notification.channels and preferences and preferences.push_enabled:
            if preferences.push_tokens:
                # Send via Celery task (async)
                from app.core.celery import send_push_task
                send_push_task.delay(
                    notification.id,
                    notification.user_id,
                    notification.title,
                    notification.message,
                    preferences.push_tokens
                )
                delivered_via.append("push")

        # Email
        if "email" in notification.channels and preferences and preferences.email_enabled:
            # Send via Celery task
            from app.core.celery import send_email_task
            send_email_task.delay(
                notification.id,
                notification.user_id,
                notification.title,
                notification.message
            )
            delivered_via.append("email")

        # Update notification
        notification.delivered_via = delivered_via
        notification.delivery_status = "sent" if delivered_via else "failed"
        await db.commit()

    @staticmethod
    async def _deliver_via_websocket(notification: Notification) -> bool:
        """Deliver notification via WebSocket"""
        from app.websocket.connection_manager import notification_manager

        message = {
            "type": "notification",
            "id": notification.id,
            "title": notification.title,
            "message": notification.message,
            "notification_type": notification.notification_type,
            "priority": notification.priority,
            "category": notification.category,
            "data": notification.data,
            "action_url": notification.action_url,
            "created_at": notification.created_at.isoformat()
        }

        return await notification_manager.send_to_user(
            notification.user_id,
            message
        )

    @staticmethod
    async def _get_user_preferences(
        user_id: int,
        db: AsyncSession
    ) -> Optional[UserPreference]:
        """Get user notification preferences"""
        result = await db.execute(
            select(UserPreference).where(UserPreference.user_id == user_id)
        )
        return result.scalar_one_or_none()

    @staticmethod
    def _filter_channels(
        channels: List[str],
        preferences: Optional[UserPreference],
        notification_type: str
    ) -> List[str]:
        """Filter channels based on user preferences"""
        if not preferences:
            return channels

        filtered = []

        for channel in channels:
            # Check channel enabled
            if channel == "websocket" and not preferences.websocket_enabled:
                continue
            if channel == "push" and not preferences.push_enabled:
                continue
            if channel == "email" and not preferences.email_enabled:
                continue
            if channel == "sms" and not preferences.sms_enabled:
                continue

            # Check type preferences
            type_prefs = preferences.type_preferences or {}
            if notification_type in type_prefs and not type_prefs[notification_type]:
                continue

            # Check quiet hours
            if preferences.quiet_hours_enabled:
                if NotificationService._is_quiet_hours(
                    preferences.quiet_hours_start,
                    preferences.quiet_hours_end
                ):
                    if channel in ["push", "email", "sms"]:
                        continue

            filtered.append(channel)

        return filtered

    @staticmethod
    def _is_quiet_hours(start: str, end: str) -> bool:
        """Check if current time is within quiet hours"""
        from datetime import time

        now = datetime.now().time()
        start_time = datetime.strptime(start, "%H:%M").time()
        end_time = datetime.strptime(end, "%H:%M").time()

        if start_time < end_time:
            return start_time <= now <= end_time
        else:  # Crosses midnight
            return now >= start_time or now <= end_time

    @staticmethod
    async def mark_as_read(
        notification_id: int,
        user_id: int,
        db: AsyncSession
    ) -> Optional[Notification]:
        """Mark notification as read"""
        result = await db.execute(
            select(Notification).where(
                and_(
                    Notification.id == notification_id,
                    Notification.user_id == user_id
                )
            )
        )
        notification = result.scalar_one_or_none()

        if notification and not notification.is_read:
            notification.is_read = True
            notification.read_at = datetime.utcnow()
            await db.commit()
            await db.refresh(notification)

        return notification

    @staticmethod
    async def get_user_notifications(
        user_id: int,
        db: AsyncSession,
        skip: int = 0,
        limit: int = 50,
        unread_only: bool = False
    ) -> Dict:
        """Get user notifications with pagination"""
        query = select(Notification).where(Notification.user_id == user_id)

        if unread_only:
            query = query.where(Notification.is_read == False)

        # Get total count
        count_query = select(func.count()).select_from(
            query.subquery()
        )
        total_result = await db.execute(count_query)
        total = total_result.scalar()

        # Get unread count
        unread_query = select(func.count()).select_from(
            select(Notification)
            .where(
                and_(
                    Notification.user_id == user_id,
                    Notification.is_read == False
                )
            )
            .subquery()
        )
        unread_result = await db.execute(unread_query)
        unread_count = unread_result.scalar()

        # Get notifications
        query = query.order_by(Notification.created_at.desc())
        query = query.offset(skip).limit(limit)
        result = await db.execute(query)
        notifications = result.scalars().all()

        return {
            "notifications": notifications,
            "total": total,
            "unread_count": unread_count
        }
```

### Step 7: WebSocket Connection Manager (app/websocket/connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Optional
import json

class NotificationConnectionManager:
    """Manages WebSocket connections for notifications"""

    def __init__(self):
        # {user_id: WebSocket}
        self.connections: Dict[int, WebSocket] = {}

    async def connect(self, user_id: int, websocket: WebSocket):
        """Connect user"""
        await websocket.accept()
        self.connections[user_id] = websocket
        print(f"User {user_id} connected for notifications")

    def disconnect(self, user_id: int):
        """Disconnect user"""
        if user_id in self.connections:
            del self.connections[user_id]
            print(f"User {user_id} disconnected from notifications")

    async def send_to_user(self, user_id: int, message: Dict) -> bool:
        """Send notification to specific user"""
        websocket = self.connections.get(user_id)
        if websocket:
            try:
                await websocket.send_json(message)
                return True
            except Exception as e:
                print(f"Error sending to user {user_id}: {e}")
                self.disconnect(user_id)
                return False
        return False

    async def broadcast(self, user_ids: list, message: Dict):
        """Broadcast notification to multiple users"""
        for user_id in user_ids:
            await self.send_to_user(user_id, message)

    def is_user_connected(self, user_id: int) -> bool:
        """Check if user is connected"""
        return user_id in self.connections

    def get_stats(self) -> Dict:
        """Get connection statistics"""
        return {
            "connected_users": len(self.connections),
            "user_ids": list(self.connections.keys())
        }

# Global manager
notification_manager = NotificationConnectionManager()
```

### Step 8: WebSocket Handler (app/websocket/handlers.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json

from app.websocket.connection_manager import notification_manager
from app.core.security import decode_access_token
from app.models.user import User
from app.db.session import get_db
from app.services.notification_service import NotificationService

async def handle_notification_stream(
    websocket: WebSocket,
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
):
    """Handle real-time notification WebSocket"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    user_id = int(payload.get("sub"))

    # Connect
    await notification_manager.connect(user_id, websocket)

    # Get unread count and send
    notification_data = await NotificationService.get_user_notifications(
        user_id, db, limit=0
    )

    await websocket.send_json({
        "type": "status",
        "message": "Connected to notification stream",
        "unread_count": notification_data["unread_count"]
    })

    try:
        while True:
            # Receive messages from client
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "mark_read":
                # Mark notification as read
                notification_id = message.get("notification_id")
                await NotificationService.mark_as_read(
                    notification_id,
                    user_id,
                    db
                )

                # Send updated unread count
                notification_data = await NotificationService.get_user_notifications(
                    user_id, db, limit=0
                )

                await websocket.send_json({
                    "type": "status",
                    "message": "Notification marked as read",
                    "unread_count": notification_data["unread_count"]
                })

            elif msg_type == "get_unread_count":
                # Get current unread count
                notification_data = await NotificationService.get_user_notifications(
                    user_id, db, limit=0
                )

                await websocket.send_json({
                    "type": "status",
                    "message": "Unread count",
                    "unread_count": notification_data["unread_count"]
                })

    except WebSocketDisconnect:
        notification_manager.disconnect(user_id)
    except Exception as e:
        print(f"Error in notification WebSocket: {e}")
        notification_manager.disconnect(user_id)
```

### Step 9: API Endpoints (app/api/notifications.py)
```python
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List

from app.schemas.notification import (
    NotificationCreate,
    NotificationResponse,
    NotificationList,
    NotificationUpdate
)
from app.services.notification_service import NotificationService
from app.db.session import get_db
from app.core.security import get_current_user
from app.models.user import User

router = APIRouter()

@router.post("/send", response_model=List[NotificationResponse])
async def send_notification(
    notification: NotificationCreate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Send notification to users"""
    notifications = await NotificationService.create_notification(
        notification,
        db
    )
    return notifications

@router.get("/", response_model=NotificationList)
async def get_notifications(
    skip: int = Query(0, ge=0),
    limit: int = Query(50, ge=1, le=100),
    unread_only: bool = False,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Get user notifications"""
    data = await NotificationService.get_user_notifications(
        current_user.id,
        db,
        skip=skip,
        limit=limit,
        unread_only=unread_only
    )
    return NotificationList(**data)

@router.patch("/{notification_id}/read", response_model=NotificationResponse)
async def mark_notification_read(
    notification_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    """Mark notification as read"""
    notification = await NotificationService.mark_as_read(
        notification_id,
        current_user.id,
        db
    )

    if not notification:
        raise HTTPException(status_code=404, detail="Notification not found")

    return notification
```

### Step 10: Run the Application
```bash
# Start Redis
redis-server

# Start Celery worker
celery -A app.core.celery worker --loglevel=info

# Start the application
uvicorn app.main:app --reload --port 8000

# Test WebSocket
# ws://localhost:8000/ws/notifications?token=YOUR_TOKEN
```

## Expected Outputs

### 1. WebSocket Notification
```json
{
  "type": "notification",
  "id": 123,
  "title": "New Message",
  "message": "You have a new message from John",
  "notification_type": "message",
  "priority": "normal",
  "category": "social",
  "data": {
    "sender_id": 456,
    "message_id": 789
  },
  "action_url": "/messages/789",
  "created_at": "2024-01-15T10:30:00"
}
```

### 2. Status Update
```json
{
  "type": "status",
  "message": "Connected to notification stream",
  "unread_count": 5
}
```

### 3. Notification List
```json
{
  "notifications": [...],
  "total": 150,
  "unread_count": 12
}
```

## Bonus Challenges

- [ ] Add notification grouping/threading
- [ ] Implement digest emails (daily/weekly summary)
- [ ] Create notification sound customization
- [ ] Add notification snooze feature
- [ ] Implement A/B testing for notifications
- [ ] Create notification analytics dashboard
- [ ] Add user notification muting
- [ ] Implement smart notification batching
- [ ] Create notification templates editor
- [ ] Add multi-language support
- [ ] Implement notification actions (quick reply)
- [ ] Create notification scheduling calendar
- [ ] Add notification priority inbox
- [ ] Implement notification categories
- [ ] Create notification webhook integrations

## Resources

- [WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [SendGrid API](https://docs.sendgrid.com/)
- [Celery Documentation](https://docs.celeryproject.org/)
- [Redis Pub/Sub](https://redis.io/topics/pubsub)
- [Web Push Notifications](https://web.dev/push-notifications-overview/)

## Success Criteria

- [ ] Real-time notifications delivered via WebSocket
- [ ] Push notifications work on mobile/web
- [ ] Email notifications sent successfully
- [ ] User preferences respected
- [ ] Notification history persisted
- [ ] Read/unread tracking accurate
- [ ] Supports 1000+ concurrent WebSocket connections
- [ ] Notification delivery < 1 second
- [ ] Failed deliveries retry automatically
- [ ] All notification types supported
- [ ] Analytics track delivery rates
- [ ] System handles offline users gracefully
- [ ] Quiet hours work correctly
- [ ] Rate limiting prevents spam
