# Project 1: Build Real-time Chat Application with FastAPI WebSockets

## Overview
Create a production-ready real-time chat application using FastAPI WebSockets with features like private messaging, chat rooms, user presence tracking, message history, and typing indicators. This project teaches the fundamentals of WebSocket communication, connection management, and building scalable real-time applications.

## Difficulty Level
Intermediate

## Learning Objectives
- Implement WebSocket connections in FastAPI
- Manage multiple concurrent WebSocket connections
- Build chat rooms and private messaging
- Handle user authentication with WebSockets
- Implement presence detection and typing indicators
- Store and retrieve message history
- Handle connection failures and reconnections
- Scale WebSocket applications with Redis pub/sub

## Technical Stack
- **Framework**: FastAPI 0.104+
- **WebSocket**: FastAPI WebSocket
- **Database**: PostgreSQL (message history) + Redis (real-time state)
- **Authentication**: JWT tokens
- **ORM**: SQLAlchemy
- **Async**: asyncio, aioredis
- **Frontend**: HTML5 + JavaScript (demo)
- **Testing**: pytest-asyncio, websockets

## Project Requirements

### 1. Core Features
- User authentication and authorization
- Public chat rooms
- Private one-on-one messaging
- Group chats
- Message history persistence
- Real-time user presence
- Typing indicators
- Read receipts
- File sharing support

### 2. WebSocket Endpoints
- `WS /ws/chat/{room_id}` - Join chat room
- `WS /ws/private/{user_id}` - Private messaging
- `WS /ws/global` - Global chat
- `GET /api/rooms` - List available rooms
- `GET /api/messages/{room_id}` - Get message history

### 3. Message Types
- Text messages
- System messages (join/leave)
- Typing indicators
- Read receipts
- User presence updates
- File attachments

### 4. Connection Management
- Handle multiple connections per user
- Graceful disconnect/reconnect
- Connection health monitoring
- Rate limiting per user
- Broadcast optimizations

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project directory
mkdir realtime-chat-websocket
cd realtime-chat-websocket

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis
pip install python-jose[cryptography] passlib[bcrypt]
pip install pydantic-settings python-multipart
pip install pytest pytest-asyncio websockets
pip install alembic

# Create requirements.txt
cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
aiosqlite==0.19.0
asyncpg==0.29.0
redis==5.0.1
aioredis==2.0.1
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
pydantic-settings==2.1.0
python-multipart==0.0.6
pytest==7.4.3
pytest-asyncio==0.21.1
websockets==12.0
alembic==1.13.0
EOF
```

### Step 2: Project Structure
```bash
# Create directory structure
mkdir -p app/{api,core,db,models,schemas,services,websocket}
touch app/__init__.py
touch app/main.py
touch app/config.py

# Database
touch app/db/__init__.py
touch app/db/session.py
touch app/db/base.py

# Models
touch app/models/__init__.py
touch app/models/user.py
touch app/models/message.py
touch app/models/room.py

# Schemas
touch app/schemas/__init__.py
touch app/schemas/user.py
touch app/schemas/message.py
touch app/schemas/websocket.py

# Services
touch app/services/__init__.py
touch app/services/auth.py
touch app/services/chat.py

# WebSocket
touch app/websocket/__init__.py
touch app/websocket/connection_manager.py
touch app/websocket/handlers.py

# API routes
touch app/api/__init__.py
touch app/api/auth.py
touch app/api/rooms.py
touch app/api/messages.py

# Core utilities
touch app/core/__init__.py
touch app/core/security.py
touch app/core/redis.py

# Static files
mkdir -p static
touch static/chat.html

# Tests
mkdir -p tests
touch tests/__init__.py
touch tests/test_websocket.py

# Alembic
alembic init alembic
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    # Application
    APP_NAME: str = "Real-time Chat"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = True

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./chat.db"
    # For PostgreSQL: postgresql+asyncpg://user:pass@localhost/chatdb

    # Redis
    REDIS_HOST: str = "localhost"
    REDIS_PORT: int = 6379
    REDIS_DB: int = 0
    REDIS_URL: str = f"redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}"

    # JWT
    SECRET_KEY: str = "your-secret-key-change-in-production"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24  # 24 hours

    # WebSocket
    MAX_CONNECTIONS_PER_USER: int = 5
    MESSAGE_RATE_LIMIT: int = 10  # messages per second
    PING_INTERVAL: int = 30  # seconds
    MAX_MESSAGE_LENGTH: int = 4096

    # Chat
    MAX_ROOM_SIZE: int = 100
    MESSAGE_HISTORY_LIMIT: int = 100
    TYPING_TIMEOUT: int = 3  # seconds

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/user.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True, nullable=False)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String)
    is_active = Column(Boolean, default=True)
    is_online = Column(Boolean, default=False)
    last_seen = Column(DateTime, default=datetime.utcnow)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    sent_messages = relationship("Message", back_populates="sender", foreign_keys="Message.sender_id")
    rooms = relationship("RoomMember", back_populates="user")
```

**app/models/room.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean, ForeignKey, Table
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Room(Base):
    __tablename__ = "rooms"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, unique=True, index=True, nullable=False)
    description = Column(String)
    is_private = Column(Boolean, default=False)
    created_by = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    members = relationship("RoomMember", back_populates="room")
    messages = relationship("Message", back_populates="room")

class RoomMember(Base):
    __tablename__ = "room_members"

    id = Column(Integer, primary_key=True, index=True)
    room_id = Column(Integer, ForeignKey("rooms.id"), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    joined_at = Column(DateTime, default=datetime.utcnow)
    is_admin = Column(Boolean, default=False)

    # Relationships
    room = relationship("Room", back_populates="members")
    user = relationship("User", back_populates="rooms")
```

**app/models/message.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Text, Boolean
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Message(Base):
    __tablename__ = "messages"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    sender_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    room_id = Column(Integer, ForeignKey("rooms.id"), nullable=True)
    recipient_id = Column(Integer, ForeignKey("users.id"), nullable=True)  # For private messages
    message_type = Column(String, default="text")  # text, system, file
    is_read = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)

    # Relationships
    sender = relationship("User", back_populates="sent_messages", foreign_keys=[sender_id])
    room = relationship("Room", back_populates="messages")
```

### Step 5: Database Session (app/db/session.py)
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from app.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
    future=True
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

**app/db/base.py**:
```python
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

# Import all models here for Alembic
from app.models.user import User
from app.models.room import Room, RoomMember
from app.models.message import Message
```

### Step 6: Pydantic Schemas

**app/schemas/websocket.py**:
```python
from pydantic import BaseModel, Field
from typing import Optional, List, Literal
from datetime import datetime

class WSMessage(BaseModel):
    """Base WebSocket message"""
    type: str
    timestamp: float = Field(default_factory=lambda: datetime.utcnow().timestamp())

class ChatMessage(WSMessage):
    """Chat message from client"""
    type: Literal["message"] = "message"
    content: str = Field(..., max_length=4096)
    room_id: Optional[int] = None
    recipient_id: Optional[int] = None

class ChatMessageResponse(WSMessage):
    """Chat message to client"""
    type: Literal["message"] = "message"
    id: int
    content: str
    sender_id: int
    sender_username: str
    room_id: Optional[int] = None
    created_at: datetime

class TypingIndicator(WSMessage):
    """Typing indicator"""
    type: Literal["typing"] = "typing"
    user_id: int
    username: str
    room_id: Optional[int] = None
    is_typing: bool

class PresenceUpdate(WSMessage):
    """User presence update"""
    type: Literal["presence"] = "presence"
    user_id: int
    username: str
    status: Literal["online", "offline", "away"]

class UserJoined(WSMessage):
    """User joined room"""
    type: Literal["user_joined"] = "user_joined"
    user_id: int
    username: str
    room_id: int

class UserLeft(WSMessage):
    """User left room"""
    type: Literal["user_left"] = "user_left"
    user_id: int
    username: str
    room_id: int

class SystemMessage(WSMessage):
    """System message"""
    type: Literal["system"] = "system"
    content: str
    level: Literal["info", "warning", "error"] = "info"

class ReadReceipt(WSMessage):
    """Message read receipt"""
    type: Literal["read_receipt"] = "read_receipt"
    message_id: int
    user_id: int

class ErrorMessage(WSMessage):
    """Error message"""
    type: Literal["error"] = "error"
    message: str
    code: Optional[str] = None
```

### Step 7: Redis Client (app/core/redis.py)
```python
import redis.asyncio as redis
from typing import Optional
from app.config import settings

class RedisClient:
    """Redis client for pub/sub and caching"""

    def __init__(self):
        self.redis: Optional[redis.Redis] = None
        self.pubsub: Optional[redis.client.PubSub] = None

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

    async def publish(self, channel: str, message: str):
        """Publish message to channel"""
        await self.redis.publish(channel, message)

    async def subscribe(self, *channels: str):
        """Subscribe to channels"""
        self.pubsub = self.redis.pubsub()
        await self.pubsub.subscribe(*channels)
        return self.pubsub

    async def set_user_online(self, user_id: int, username: str):
        """Mark user as online"""
        await self.redis.setex(
            f"user:online:{user_id}",
            300,  # 5 minutes TTL
            username
        )

    async def set_user_typing(self, user_id: int, room_id: int):
        """Mark user as typing"""
        await self.redis.setex(
            f"typing:{room_id}:{user_id}",
            settings.TYPING_TIMEOUT,
            "1"
        )

    async def get_online_users(self) -> list:
        """Get all online users"""
        keys = await self.redis.keys("user:online:*")
        if not keys:
            return []
        values = await self.redis.mget(keys)
        return [v for v in values if v]

# Global Redis client
redis_client = RedisClient()
```

### Step 8: Connection Manager (app/websocket/connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Set, List, Optional
import json
from collections import defaultdict
import asyncio
from datetime import datetime

from app.schemas.websocket import WSMessage, SystemMessage, ErrorMessage
from app.core.redis import redis_client

class ConnectionManager:
    """Manages WebSocket connections"""

    def __init__(self):
        # {user_id: {connection_id: WebSocket}}
        self.active_connections: Dict[int, Dict[str, WebSocket]] = defaultdict(dict)

        # {room_id: {user_id}}
        self.room_users: Dict[int, Set[int]] = defaultdict(set)

        # Connection metadata
        self.connection_metadata: Dict[str, dict] = {}

        # Rate limiting: {user_id: [timestamps]}
        self.message_timestamps: Dict[int, List[float]] = defaultdict(list)

    async def connect(
        self,
        websocket: WebSocket,
        user_id: int,
        username: str,
        connection_id: str,
        room_id: Optional[int] = None
    ):
        """Accept new WebSocket connection"""
        await websocket.accept()

        # Store connection
        self.active_connections[user_id][connection_id] = websocket

        # Store metadata
        self.connection_metadata[connection_id] = {
            "user_id": user_id,
            "username": username,
            "room_id": room_id,
            "connected_at": datetime.utcnow(),
            "message_count": 0
        }

        # Join room if specified
        if room_id:
            self.room_users[room_id].add(user_id)

        # Mark user as online in Redis
        await redis_client.set_user_online(user_id, username)

        print(f"User {username} ({user_id}) connected. Connection: {connection_id}")

    def disconnect(self, connection_id: str):
        """Remove WebSocket connection"""
        metadata = self.connection_metadata.get(connection_id)
        if not metadata:
            return

        user_id = metadata["user_id"]
        room_id = metadata.get("room_id")

        # Remove connection
        if connection_id in self.active_connections.get(user_id, {}):
            del self.active_connections[user_id][connection_id]

        # Remove from room
        if room_id and user_id in self.room_users.get(room_id, set()):
            self.room_users[room_id].discard(user_id)

        # Clean up empty dictionaries
        if user_id in self.active_connections and not self.active_connections[user_id]:
            del self.active_connections[user_id]

        if room_id in self.room_users and not self.room_users[room_id]:
            del self.room_users[room_id]

        # Remove metadata
        del self.connection_metadata[connection_id]

        print(f"User {user_id} disconnected. Connection: {connection_id}")

    async def send_personal_message(self, user_id: int, message: dict):
        """Send message to all connections of a specific user"""
        connections = self.active_connections.get(user_id, {})

        disconnected = []
        for conn_id, websocket in connections.items():
            try:
                await websocket.send_json(message)
            except Exception as e:
                print(f"Error sending to {conn_id}: {e}")
                disconnected.append(conn_id)

        # Clean up disconnected
        for conn_id in disconnected:
            self.disconnect(conn_id)

    async def broadcast_to_room(
        self,
        room_id: int,
        message: dict,
        exclude_user: Optional[int] = None
    ):
        """Broadcast message to all users in a room"""
        user_ids = self.room_users.get(room_id, set())

        for user_id in user_ids:
            if user_id != exclude_user:
                await self.send_personal_message(user_id, message)

    async def send_to_connection(self, connection_id: str, message: dict):
        """Send message to specific connection"""
        metadata = self.connection_metadata.get(connection_id)
        if not metadata:
            return

        user_id = metadata["user_id"]
        websocket = self.active_connections.get(user_id, {}).get(connection_id)

        if websocket:
            try:
                await websocket.send_json(message)
            except Exception as e:
                print(f"Error sending to {connection_id}: {e}")
                self.disconnect(connection_id)

    def check_rate_limit(self, user_id: int) -> bool:
        """Check if user is within rate limit"""
        from app.config import settings

        now = datetime.utcnow().timestamp()

        # Clean old timestamps (older than 1 second)
        self.message_timestamps[user_id] = [
            ts for ts in self.message_timestamps[user_id]
            if now - ts < 1.0
        ]

        # Check limit
        if len(self.message_timestamps[user_id]) >= settings.MESSAGE_RATE_LIMIT:
            return False

        # Add timestamp
        self.message_timestamps[user_id].append(now)
        return True

    def get_room_users(self, room_id: int) -> Set[int]:
        """Get all users in a room"""
        return self.room_users.get(room_id, set())

    def get_user_connection_count(self, user_id: int) -> int:
        """Get number of active connections for user"""
        return len(self.active_connections.get(user_id, {}))

    def is_user_online(self, user_id: int) -> bool:
        """Check if user has any active connections"""
        return user_id in self.active_connections

    def get_stats(self) -> dict:
        """Get connection statistics"""
        return {
            "total_connections": sum(
                len(conns) for conns in self.active_connections.values()
            ),
            "unique_users": len(self.active_connections),
            "active_rooms": len(self.room_users),
            "rooms": {
                room_id: len(users)
                for room_id, users in self.room_users.items()
            }
        }

# Global connection manager
manager = ConnectionManager()
```

### Step 9: Authentication (app/core/security.py)
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """Create JWT access token"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

def decode_access_token(token: str) -> Optional[dict]:
    """Decode JWT token"""
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        return payload
    except JWTError:
        return None
```

### Step 10: WebSocket Handler (app/websocket/handlers.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json
import uuid
from datetime import datetime

from app.websocket.connection_manager import manager
from app.core.security import decode_access_token
from app.db.session import get_db
from app.models.user import User
from app.models.message import Message
from app.models.room import Room
from app.schemas.websocket import (
    ChatMessage, ChatMessageResponse, TypingIndicator,
    UserJoined, UserLeft, SystemMessage, ErrorMessage
)
from app.core.redis import redis_client

async def get_current_user_ws(
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Get current user from WebSocket token"""
    payload = decode_access_token(token)
    if not payload:
        raise HTTPException(status_code=401, detail="Invalid token")

    user_id = payload.get("sub")
    if not user_id:
        raise HTTPException(status_code=401, detail="Invalid token")

    result = await db.execute(select(User).where(User.id == int(user_id)))
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return user

async def handle_chat_room(
    websocket: WebSocket,
    room_id: int,
    current_user: User = Depends(get_current_user_ws),
    db: AsyncSession = Depends(get_db)
):
    """Handle chat room WebSocket connection"""
    connection_id = str(uuid.uuid4())

    # Verify room exists
    result = await db.execute(select(Room).where(Room.id == room_id))
    room = result.scalar_one_or_none()

    if not room:
        await websocket.close(code=1008, reason="Room not found")
        return

    # Connect
    await manager.connect(
        websocket,
        current_user.id,
        current_user.username,
        connection_id,
        room_id
    )

    # Send user joined message
    join_msg = UserJoined(
        user_id=current_user.id,
        username=current_user.username,
        room_id=room_id
    )
    await manager.broadcast_to_room(
        room_id,
        join_msg.dict(),
        exclude_user=current_user.id
    )

    try:
        while True:
            # Receive message
            data = await websocket.receive_text()
            message_data = json.loads(data)

            msg_type = message_data.get("type")

            if msg_type == "message":
                # Handle chat message
                await handle_chat_message(
                    message_data,
                    current_user,
                    room_id,
                    connection_id,
                    db
                )

            elif msg_type == "typing":
                # Handle typing indicator
                await handle_typing_indicator(
                    current_user,
                    room_id,
                    message_data.get("is_typing", True)
                )

    except WebSocketDisconnect:
        manager.disconnect(connection_id)

        # Send user left message
        leave_msg = UserLeft(
            user_id=current_user.id,
            username=current_user.username,
            room_id=room_id
        )
        await manager.broadcast_to_room(room_id, leave_msg.dict())

    except Exception as e:
        print(f"Error in WebSocket: {e}")
        manager.disconnect(connection_id)

async def handle_chat_message(
    message_data: dict,
    user: User,
    room_id: int,
    connection_id: str,
    db: AsyncSession
):
    """Handle incoming chat message"""
    # Rate limiting
    if not manager.check_rate_limit(user.id):
        error_msg = ErrorMessage(
            message="Rate limit exceeded",
            code="RATE_LIMIT"
        )
        await manager.send_to_connection(connection_id, error_msg.dict())
        return

    content = message_data.get("content", "").strip()
    if not content:
        return

    # Save to database
    db_message = Message(
        content=content,
        sender_id=user.id,
        room_id=room_id,
        message_type="text"
    )
    db.add(db_message)
    await db.commit()
    await db.refresh(db_message)

    # Broadcast to room
    response = ChatMessageResponse(
        id=db_message.id,
        content=content,
        sender_id=user.id,
        sender_username=user.username,
        room_id=room_id,
        created_at=db_message.created_at
    )

    await manager.broadcast_to_room(room_id, response.dict())

    # Publish to Redis for scaling
    await redis_client.publish(
        f"room:{room_id}",
        json.dumps(response.dict(), default=str)
    )

async def handle_typing_indicator(user: User, room_id: int, is_typing: bool):
    """Handle typing indicator"""
    if is_typing:
        await redis_client.set_user_typing(user.id, room_id)

    typing_msg = TypingIndicator(
        user_id=user.id,
        username=user.username,
        room_id=room_id,
        is_typing=is_typing
    )

    await manager.broadcast_to_room(
        room_id,
        typing_msg.dict(),
        exclude_user=user.id
    )
```

### Step 11: Main Application (app/main.py)
```python
from fastapi import FastAPI, WebSocket, Depends
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from contextlib import asynccontextmanager

from app.config import settings
from app.db.session import engine
from app.db.base import Base
from app.core.redis import redis_client
from app.websocket.handlers import handle_chat_room, get_current_user_ws
from app.api import auth, rooms, messages

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifespan events"""
    # Startup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    await redis_client.connect()
    print("Application started")

    yield

    # Shutdown
    await redis_client.disconnect()
    await engine.dispose()
    print("Application shutdown")

app = FastAPI(
    title=settings.APP_NAME,
    version=settings.APP_VERSION,
    lifespan=lifespan
)

# Include routers
app.include_router(auth.router, prefix="/api/auth", tags=["auth"])
app.include_router(rooms.router, prefix="/api/rooms", tags=["rooms"])
app.include_router(messages.router, prefix="/api/messages", tags=["messages"])

# Mount static files
app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/")
async def root():
    """Root endpoint"""
    return {
        "message": "Real-time Chat API",
        "version": settings.APP_VERSION,
        "endpoints": {
            "docs": "/docs",
            "chat": "/chat",
            "websocket": "ws://localhost:8000/ws/chat/{room_id}?token=YOUR_TOKEN"
        }
    }

@app.websocket("/ws/chat/{room_id}")
async def websocket_chat_endpoint(
    websocket: WebSocket,
    room_id: int,
    current_user = Depends(get_current_user_ws),
    db = Depends(get_db)
):
    """WebSocket endpoint for chat rooms"""
    await handle_chat_room(websocket, room_id, current_user, db)

@app.get("/chat", response_class=HTMLResponse)
async def get_chat_ui():
    """Simple chat UI"""
    with open("static/chat.html", "r") as f:
        return HTMLResponse(content=f.read())

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "app.main:app",
        host="0.0.0.0",
        port=8000,
        reload=settings.DEBUG
    )
```

### Step 12: Frontend Demo (static/chat.html)
```html
<!DOCTYPE html>
<html>
<head>
    <title>Real-time Chat</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: Arial, sans-serif; background: #f0f0f0; }
        .container { max-width: 800px; margin: 20px auto; background: white; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .header { background: #0084ff; color: white; padding: 20px; }
        .messages { height: 500px; overflow-y: auto; padding: 20px; }
        .message { margin-bottom: 15px; }
        .message.own { text-align: right; }
        .message-content { display: inline-block; max-width: 70%; padding: 10px 15px; border-radius: 18px; background: #e4e6eb; }
        .message.own .message-content { background: #0084ff; color: white; }
        .message-sender { font-size: 12px; color: #666; margin-bottom: 5px; }
        .message-time { font-size: 11px; color: #999; margin-top: 5px; }
        .typing-indicator { color: #666; font-style: italic; padding: 0 20px; height: 20px; }
        .input-area { display: flex; padding: 20px; border-top: 1px solid #ddd; }
        input { flex: 1; padding: 12px; border: 1px solid #ddd; border-radius: 20px; font-size: 14px; }
        button { margin-left: 10px; padding: 12px 30px; background: #0084ff; color: white; border: none; border-radius: 20px; cursor: pointer; }
        button:hover { background: #0073e6; }
        .status { padding: 10px 20px; background: #f8f9fa; border-bottom: 1px solid #ddd; font-size: 12px; color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Real-time Chat Room</h1>
            <div id="room-info"></div>
        </div>
        <div class="status">
            <span id="status">Connecting...</span>
        </div>
        <div class="messages" id="messages"></div>
        <div class="typing-indicator" id="typing"></div>
        <div class="input-area">
            <input type="text" id="message-input" placeholder="Type a message..." />
            <button onclick="sendMessage()">Send</button>
        </div>
    </div>

    <script>
        let ws = null;
        let currentUser = null;
        let token = localStorage.getItem('auth_token') || 'demo-token';
        let roomId = 1;
        let typingTimeout = null;

        // Connect to WebSocket
        function connect() {
            ws = new WebSocket(`ws://localhost:8000/ws/chat/${roomId}?token=${token}`);

            ws.onopen = () => {
                document.getElementById('status').textContent = 'Connected';
                document.getElementById('status').style.color = 'green';
            };

            ws.onmessage = (event) => {
                const data = JSON.parse(event.data);
                handleMessage(data);
            };

            ws.onerror = (error) => {
                console.error('WebSocket error:', error);
                document.getElementById('status').textContent = 'Error';
                document.getElementById('status').style.color = 'red';
            };

            ws.onclose = () => {
                document.getElementById('status').textContent = 'Disconnected';
                document.getElementById('status').style.color = 'red';
                setTimeout(connect, 3000); // Reconnect after 3s
            };
        }

        function handleMessage(data) {
            switch(data.type) {
                case 'message':
                    displayMessage(data);
                    break;
                case 'typing':
                    displayTyping(data);
                    break;
                case 'user_joined':
                    displaySystemMessage(`${data.username} joined the room`);
                    break;
                case 'user_left':
                    displaySystemMessage(`${data.username} left the room`);
                    break;
                case 'system':
                    displaySystemMessage(data.content);
                    break;
                case 'error':
                    alert(`Error: ${data.message}`);
                    break;
            }
        }

        function displayMessage(msg) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message' + (msg.sender_id === currentUser?.id ? ' own' : '');

            const time = new Date(msg.created_at).toLocaleTimeString();

            messageDiv.innerHTML = `
                <div class="message-sender">${msg.sender_username}</div>
                <div class="message-content">${escapeHtml(msg.content)}</div>
                <div class="message-time">${time}</div>
            `;

            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }

        function displaySystemMessage(content) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.style.textAlign = 'center';
            messageDiv.style.color = '#666';
            messageDiv.style.fontSize = '12px';
            messageDiv.style.margin = '10px 0';
            messageDiv.textContent = content;
            messagesDiv.appendChild(messageDiv);
        }

        function displayTyping(data) {
            const typingDiv = document.getElementById('typing');
            if (data.is_typing) {
                typingDiv.textContent = `${data.username} is typing...`;
            } else {
                typingDiv.textContent = '';
            }
        }

        function sendMessage() {
            const input = document.getElementById('message-input');
            const content = input.value.trim();

            if (content && ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({
                    type: 'message',
                    content: content
                }));
                input.value = '';

                // Stop typing indicator
                sendTypingIndicator(false);
            }
        }

        function sendTypingIndicator(isTyping) {
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({
                    type: 'typing',
                    is_typing: isTyping
                }));
            }
        }

        // Handle typing
        document.getElementById('message-input').addEventListener('input', function() {
            sendTypingIndicator(true);

            // Clear previous timeout
            if (typingTimeout) {
                clearTimeout(typingTimeout);
            }

            // Stop typing after 3 seconds of inactivity
            typingTimeout = setTimeout(() => {
                sendTypingIndicator(false);
            }, 3000);
        });

        // Send message on Enter
        document.getElementById('message-input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });

        function escapeHtml(text) {
            const div = document.createElement('div');
            div.textContent = text;
            return div.innerHTML;
        }

        // Connect on load
        connect();
    </script>
</body>
</html>
```

### Step 13: Run the Application
```bash
# Start Redis (in separate terminal)
redis-server

# Run database migrations
alembic revision --autogenerate -m "Initial migration"
alembic upgrade head

# Start the application
uvicorn app.main:app --reload --port 8000

# Access the chat
# http://localhost:8000/chat
```

## Expected Outputs

### 1. WebSocket Connection
```json
{
  "type": "status",
  "message": "Connected to ML streaming API",
  "data": {
    "client_id": "550e8400-e29b-41d4-a716-446655440000"
  },
  "timestamp": 1699965432.123
}
```

### 2. Chat Message
```json
{
  "type": "message",
  "id": 42,
  "content": "Hello everyone!",
  "sender_id": 1,
  "sender_username": "john_doe",
  "room_id": 1,
  "created_at": "2024-01-15T10:30:00"
}
```

### 3. Typing Indicator
```json
{
  "type": "typing",
  "user_id": 2,
  "username": "jane_smith",
  "room_id": 1,
  "is_typing": true
}
```

### 4. User Joined
```json
{
  "type": "user_joined",
  "user_id": 3,
  "username": "alice",
  "room_id": 1
}
```

## Bonus Challenges

- [ ] Add end-to-end encryption for private messages
- [ ] Implement voice/video call signaling
- [ ] Add emoji reactions to messages
- [ ] Create message threading/replies
- [ ] Implement message search functionality
- [ ] Add file upload and sharing
- [ ] Create user mentions (@username)
- [ ] Add message editing and deletion
- [ ] Implement user blocking and reporting
- [ ] Create admin moderation tools
- [ ] Add message translation
- [ ] Implement read receipts
- [ ] Create chat bots
- [ ] Add GIF and sticker support
- [ ] Implement message scheduling
- [ ] Add push notifications
- [ ] Create chat analytics dashboard

## Resources

- [FastAPI WebSockets Documentation](https://fastapi.tiangolo.com/advanced/websockets/)
- [WebSocket Protocol RFC 6455](https://tools.ietf.org/html/rfc6455)
- [Redis Pub/Sub](https://redis.io/topics/pubsub)
- [SQLAlchemy Async](https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html)
- [JWT Authentication](https://jwt.io/introduction)
- [WebSocket Security](https://owasp.org/www-community/vulnerabilities/WebSocket_Security)

## Success Criteria

- [ ] Users can connect and authenticate via WebSocket
- [ ] Messages are sent and received in real-time
- [ ] Multiple users can chat in the same room
- [ ] Typing indicators work correctly
- [ ] User presence is tracked accurately
- [ ] Message history is persisted in database
- [ ] Rate limiting prevents spam
- [ ] Graceful handling of disconnections
- [ ] UI updates in real-time without page refresh
- [ ] No message loss during normal operation
- [ ] System handles at least 100 concurrent users
- [ ] Average message latency < 100ms
- [ ] Reconnection works automatically
- [ ] All WebSocket errors are handled properly
