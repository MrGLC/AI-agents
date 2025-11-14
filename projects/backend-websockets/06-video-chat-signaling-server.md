# Project 6: Build Video Chat Signaling Server

## Overview
Create a WebRTC signaling server using WebSockets that enables peer-to-peer video/audio communication. This project demonstrates building the infrastructure for video conferencing applications, handling ICE candidates, SDP offers/answers, and managing call sessions.

## Difficulty Level
Advanced

## Learning Objectives
- Understand WebRTC signaling protocol
- Implement WebSocket-based signaling
- Handle ICE candidate exchange
- Manage SDP offer/answer negotiation
- Build room-based video calls
- Handle connection state management
- Implement TURN/STUN server integration
- Create call quality monitoring
- Handle reconnection scenarios

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Protocol**: WebRTC signaling
- **Database**: PostgreSQL (call history) + Redis (active sessions)
- **TURN/STUN**: coturn or Twilio TURN
- **Frontend**: WebRTC JavaScript API
- **Recording**: Optional media server integration
- **Testing**: pytest-asyncio, WebRTC testing tools

## Project Requirements

### 1. Core Features
- One-on-one video calls
- Group video calls (rooms)
- Screen sharing support
- Audio/video mute controls
- Call history and logs
- Connection quality monitoring
- Automatic reconnection
- TURN/STUN fallback
- Call recording (optional)

### 2. Signaling Messages
- Offer/Answer (SDP exchange)
- ICE candidates
- Call invite/accept/reject
- Hangup
- Media state changes
- Room join/leave
- Participant updates

### 3. WebSocket Endpoints
- `WS /ws/signal` - Main signaling endpoint
- `WS /ws/room/{room_id}` - Room-based signaling
- `GET /api/rooms` - List active rooms
- `POST /api/rooms` - Create room
- `GET /api/calls/history` - Call history

### 4. Call Types
- Direct peer-to-peer (1-on-1)
- Multi-party (SFU/MCU)
- Broadcasting
- Screen sharing sessions

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir webrtc-signaling-server
cd webrtc-signaling-server

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis
pip install pydantic pydantic-settings
pip install python-jose[cryptography]
pip install pytest pytest-asyncio

cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
aiosqlite==0.19.0
redis==5.0.1
python-jose[cryptography]==3.3.0
pydantic==2.5.0
pytest==7.4.3
pytest-asyncio==0.21.1
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,user,call,room}.py

# Schemas
touch app/schemas/{__init__,signaling,call}.py

# Services
touch app/services/{__init__,signaling_service,room_service}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,signaling_handler}.py

# Core
touch app/core/{__init__,redis,security}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/{video-call.html,room.html}
mkdir static/{js,css}
touch static/js/webrtc.js

# Tests
mkdir tests
touch tests/{test_signaling,test_rooms}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    APP_NAME: str = "WebRTC Signaling Server"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./webrtc.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # STUN/TURN Servers
    STUN_SERVERS: List[str] = [
        "stun:stun.l.google.com:19302",
        "stun:stun1.l.google.com:19302"
    ]
    TURN_SERVERS: List[dict] = []  # Add your TURN server config
    # Example:
    # [{"urls": "turn:turn.example.com:3478", "username": "user", "credential": "pass"}]

    # Call Settings
    MAX_ROOM_SIZE: int = 10
    MAX_CALL_DURATION: int = 3600  # 1 hour
    CALL_TIMEOUT: int = 60  # 1 minute for answering

    # WebSocket
    PING_INTERVAL: int = 30
    SIGNALING_TIMEOUT: int = 30

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"

    # Recording (optional)
    ENABLE_RECORDING: bool = False
    RECORDING_PATH: str = "./recordings"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/call.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Boolean, Float, JSON
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Call(Base):
    __tablename__ = "calls"

    id = Column(Integer, primary_key=True, index=True)
    call_id = Column(String(100), unique=True, index=True)
    call_type = Column(String(20), nullable=False)  # direct, group, broadcast
    initiator_id = Column(Integer, ForeignKey("users.id"))
    room_id = Column(String(100), nullable=True)
    status = Column(String(20), default="initiated")  # initiated, ringing, active, ended
    started_at = Column(DateTime, nullable=True)
    ended_at = Column(DateTime, nullable=True)
    duration = Column(Float, nullable=True)  # seconds
    created_at = Column(DateTime, default=datetime.utcnow)

    # Call quality metrics
    quality_metrics = Column(JSON)  # Store quality stats

    # Recording
    is_recorded = Column(Boolean, default=False)
    recording_url = Column(String(500), nullable=True)

    # Relationships
    initiator = relationship("User", foreign_keys=[initiator_id])
    participants = relationship("CallParticipant", back_populates="call")

class CallParticipant(Base):
    __tablename__ = "call_participants"

    id = Column(Integer, primary_key=True, index=True)
    call_id = Column(Integer, ForeignKey("calls.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    joined_at = Column(DateTime, default=datetime.utcnow)
    left_at = Column(DateTime, nullable=True)
    is_video_enabled = Column(Boolean, default=True)
    is_audio_enabled = Column(Boolean, default=True)
    connection_quality = Column(String(20))  # good, fair, poor

    call = relationship("Call", back_populates="participants")
    user = relationship("User")

class Room(Base):
    __tablename__ = "rooms"

    id = Column(Integer, primary_key=True, index=True)
    room_id = Column(String(100), unique=True, index=True)
    name = Column(String(255), nullable=False)
    created_by = Column(Integer, ForeignKey("users.id"))
    max_participants = Column(Integer, default=10)
    is_public = Column(Boolean, default=False)
    password = Column(String(255), nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    is_active = Column(Boolean, default=True)

    creator = relationship("User")
```

### Step 5: Signaling Schemas (app/schemas/signaling.py)
```python
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, Literal, List

class SignalingMessage(BaseModel):
    """Base signaling message"""
    type: str
    sender_id: Optional[int] = None
    target_id: Optional[int] = None
    room_id: Optional[str] = None

class OfferMessage(SignalingMessage):
    """WebRTC Offer"""
    type: Literal["offer"] = "offer"
    sdp: str
    call_id: str

class AnswerMessage(SignalingMessage):
    """WebRTC Answer"""
    type: Literal["answer"] = "answer"
    sdp: str
    call_id: str

class ICECandidateMessage(SignalingMessage):
    """ICE Candidate"""
    type: Literal["ice-candidate"] = "ice-candidate"
    candidate: Dict[str, Any]
    call_id: str

class CallInvite(SignalingMessage):
    """Call invitation"""
    type: Literal["call-invite"] = "call-invite"
    call_id: str
    caller_name: str
    call_type: str = "video"  # video, audio, screen

class CallAccept(SignalingMessage):
    """Accept call"""
    type: Literal["call-accept"] = "call-accept"
    call_id: str

class CallReject(SignalingMessage):
    """Reject call"""
    type: Literal["call-reject"] = "call-reject"
    call_id: str
    reason: Optional[str] = None

class CallHangup(SignalingMessage):
    """End call"""
    type: Literal["hangup"] = "hangup"
    call_id: str

class MediaStateChange(SignalingMessage):
    """Media state change (mute/unmute)"""
    type: Literal["media-state"] = "media-state"
    call_id: str
    is_video_enabled: bool
    is_audio_enabled: bool

class JoinRoom(SignalingMessage):
    """Join video room"""
    type: Literal["join-room"] = "join-room"
    room_id: str
    user_name: str

class LeaveRoom(SignalingMessage):
    """Leave video room"""
    type: Literal["leave-room"] = "leave-room"
    room_id: str

class RoomParticipants(SignalingMessage):
    """Room participants list"""
    type: Literal["room-participants"] = "room-participants"
    room_id: str
    participants: List[Dict[str, Any]]

class ErrorMessage(SignalingMessage):
    """Error message"""
    type: Literal["error"] = "error"
    message: str
    code: Optional[str] = None
```

### Step 6: Signaling Service (app/services/signaling_service.py)
```python
from typing import Dict, Optional, List
from datetime import datetime
import uuid

from app.models.call import Call, CallParticipant, Room
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

class SignalingService:
    """Service for managing call signaling"""

    @staticmethod
    async def create_call(
        initiator_id: int,
        call_type: str,
        room_id: Optional[str],
        db: AsyncSession
    ) -> Call:
        """Create a new call"""
        call = Call(
            call_id=str(uuid.uuid4()),
            call_type=call_type,
            initiator_id=initiator_id,
            room_id=room_id,
            status="initiated"
        )
        db.add(call)
        await db.commit()
        await db.refresh(call)
        return call

    @staticmethod
    async def add_participant(
        call_id: str,
        user_id: int,
        db: AsyncSession
    ) -> Optional[CallParticipant]:
        """Add participant to call"""
        # Get call
        result = await db.execute(
            select(Call).where(Call.call_id == call_id)
        )
        call = result.scalar_one_or_none()

        if not call:
            return None

        # Check if already participant
        result = await db.execute(
            select(CallParticipant).where(
                CallParticipant.call_id == call.id,
                CallParticipant.user_id == user_id
            )
        )
        existing = result.scalar_one_or_none()

        if existing:
            return existing

        # Add participant
        participant = CallParticipant(
            call_id=call.id,
            user_id=user_id
        )
        db.add(participant)
        await db.commit()
        await db.refresh(participant)

        return participant

    @staticmethod
    async def start_call(call_id: str, db: AsyncSession):
        """Mark call as started"""
        result = await db.execute(
            select(Call).where(Call.call_id == call_id)
        )
        call = result.scalar_one_or_none()

        if call:
            call.status = "active"
            call.started_at = datetime.utcnow()
            await db.commit()

    @staticmethod
    async def end_call(call_id: str, db: AsyncSession):
        """End call and calculate duration"""
        result = await db.execute(
            select(Call).where(Call.call_id == call_id)
        )
        call = result.scalar_one_or_none()

        if call:
            call.status = "ended"
            call.ended_at = datetime.utcnow()

            if call.started_at:
                duration = (call.ended_at - call.started_at).total_seconds()
                call.duration = duration

            await db.commit()

    @staticmethod
    async def remove_participant(
        call_id: str,
        user_id: int,
        db: AsyncSession
    ):
        """Remove participant from call"""
        result = await db.execute(
            select(Call).where(Call.call_id == call_id)
        )
        call = result.scalar_one_or_none()

        if call:
            result = await db.execute(
                select(CallParticipant).where(
                    CallParticipant.call_id == call.id,
                    CallParticipant.user_id == user_id
                )
            )
            participant = result.scalar_one_or_none()

            if participant:
                participant.left_at = datetime.utcnow()
                await db.commit()
```

### Step 7: Connection Manager (app/websocket/connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Set, Optional
import json

class SignalingConnectionManager:
    """Manages WebSocket connections for signaling"""

    def __init__(self):
        # {user_id: WebSocket}
        self.user_connections: Dict[int, WebSocket] = {}

        # {room_id: {user_id}}
        self.room_participants: Dict[str, Set[int]] = {}

        # {call_id: {user_id}}
        self.call_participants: Dict[str, Set[int]] = {}

    async def connect_user(self, user_id: int, websocket: WebSocket):
        """Connect user"""
        await websocket.accept()
        self.user_connections[user_id] = websocket

    def disconnect_user(self, user_id: int):
        """Disconnect user"""
        if user_id in self.user_connections:
            del self.user_connections[user_id]

        # Remove from all rooms
        for room_id, participants in list(self.room_participants.items()):
            if user_id in participants:
                participants.discard(user_id)
                if not participants:
                    del self.room_participants[room_id]

        # Remove from all calls
        for call_id, participants in list(self.call_participants.items()):
            if user_id in participants:
                participants.discard(user_id)
                if not participants:
                    del self.call_participants[call_id]

    async def send_to_user(self, user_id: int, message: Dict) -> bool:
        """Send message to specific user"""
        websocket = self.user_connections.get(user_id)
        if websocket:
            try:
                await websocket.send_json(message)
                return True
            except:
                self.disconnect_user(user_id)
                return False
        return False

    async def forward_to_target(
        self,
        sender_id: int,
        target_id: int,
        message: Dict
    ):
        """Forward signaling message to target user"""
        message["sender_id"] = sender_id
        await self.send_to_user(target_id, message)

    def join_room(self, user_id: int, room_id: str):
        """Add user to room"""
        if room_id not in self.room_participants:
            self.room_participants[room_id] = set()
        self.room_participants[room_id].add(user_id)

    def leave_room(self, user_id: int, room_id: str):
        """Remove user from room"""
        if room_id in self.room_participants:
            self.room_participants[room_id].discard(user_id)
            if not self.room_participants[room_id]:
                del self.room_participants[room_id]

    async def broadcast_to_room(
        self,
        room_id: str,
        message: Dict,
        exclude_user: Optional[int] = None
    ):
        """Broadcast message to all in room"""
        participants = self.room_participants.get(room_id, set())

        for user_id in participants:
            if user_id != exclude_user:
                await self.send_to_user(user_id, message)

    def get_room_participants(self, room_id: str) -> List[int]:
        """Get list of participants in room"""
        return list(self.room_participants.get(room_id, set()))

    def join_call(self, user_id: int, call_id: str):
        """Add user to call"""
        if call_id not in self.call_participants:
            self.call_participants[call_id] = set()
        self.call_participants[call_id].add(user_id)

    def leave_call(self, user_id: int, call_id: str):
        """Remove user from call"""
        if call_id in self.call_participants:
            self.call_participants[call_id].discard(user_id)
            if not self.call_participants[call_id]:
                del self.call_participants[call_id]

    def get_call_participants(self, call_id: str) -> List[int]:
        """Get participants in call"""
        return list(self.call_participants.get(call_id, set()))

# Global manager
signaling_manager = SignalingConnectionManager()
```

### Step 8: Signaling Handler (app/websocket/signaling_handler.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
import json

from app.websocket.connection_manager import signaling_manager
from app.services.signaling_service import SignalingService
from app.core.security import decode_access_token
from app.db.session import get_db
from app.models.user import User
from sqlalchemy import select

async def handle_signaling(
    websocket: WebSocket,
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
):
    """Handle WebRTC signaling WebSocket"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    user_id = int(payload.get("sub"))
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()

    if not user:
        await websocket.close(code=1008, reason="User not found")
        return

    # Connect
    await signaling_manager.connect_user(user_id, websocket)

    # Send ICE servers config
    from app.config import settings
    await websocket.send_json({
        "type": "config",
        "iceServers": settings.STUN_SERVERS + settings.TURN_SERVERS
    })

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "offer":
                # Forward offer to target
                await handle_offer(message, user_id, db)

            elif msg_type == "answer":
                # Forward answer to target
                await handle_answer(message, user_id)

            elif msg_type == "ice-candidate":
                # Forward ICE candidate
                await handle_ice_candidate(message, user_id)

            elif msg_type == "call-invite":
                # Send call invitation
                await handle_call_invite(message, user_id, user.username, db)

            elif msg_type == "call-accept":
                # Accept call
                await handle_call_accept(message, user_id, db)

            elif msg_type == "call-reject":
                # Reject call
                await handle_call_reject(message, user_id)

            elif msg_type == "hangup":
                # End call
                await handle_hangup(message, user_id, db)

            elif msg_type == "media-state":
                # Media state change
                await handle_media_state(message, user_id)

            elif msg_type == "join-room":
                # Join video room
                await handle_join_room(message, user_id, user.username, db)

            elif msg_type == "leave-room":
                # Leave room
                await handle_leave_room(message, user_id)

    except WebSocketDisconnect:
        signaling_manager.disconnect_user(user_id)
    except Exception as e:
        print(f"Error in signaling: {e}")
        signaling_manager.disconnect_user(user_id)

async def handle_offer(message: dict, sender_id: int, db: AsyncSession):
    """Handle WebRTC offer"""
    target_id = message.get("target_id")
    call_id = message.get("call_id")

    if target_id:
        # Add to call participants
        signaling_manager.join_call(sender_id, call_id)
        signaling_manager.join_call(target_id, call_id)

        # Forward offer
        await signaling_manager.forward_to_target(
            sender_id,
            target_id,
            message
        )

async def handle_answer(message: dict, sender_id: int):
    """Handle WebRTC answer"""
    target_id = message.get("target_id")

    if target_id:
        await signaling_manager.forward_to_target(
            sender_id,
            target_id,
            message
        )

async def handle_ice_candidate(message: dict, sender_id: int):
    """Handle ICE candidate"""
    target_id = message.get("target_id")

    if target_id:
        await signaling_manager.forward_to_target(
            sender_id,
            target_id,
            message
        )

async def handle_call_invite(
    message: dict,
    caller_id: int,
    caller_name: str,
    db: AsyncSession
):
    """Handle call invitation"""
    target_id = message.get("target_id")
    call_type = message.get("call_type", "video")

    # Create call
    call = await SignalingService.create_call(
        caller_id,
        "direct",
        None,
        db
    )

    # Send invitation to target
    await signaling_manager.send_to_user(
        target_id,
        {
            "type": "call-invite",
            "call_id": call.call_id,
            "caller_id": caller_id,
            "caller_name": caller_name,
            "call_type": call_type
        }
    )

async def handle_call_accept(message: dict, user_id: int, db: AsyncSession):
    """Handle call acceptance"""
    call_id = message.get("call_id")
    target_id = message.get("target_id")

    # Add participant
    await SignalingService.add_participant(call_id, user_id, db)

    # Start call
    await SignalingService.start_call(call_id, db)

    # Notify caller
    if target_id:
        await signaling_manager.send_to_user(
            target_id,
            {
                "type": "call-accept",
                "call_id": call_id,
                "user_id": user_id
            }
        )

async def handle_call_reject(message: dict, user_id: int):
    """Handle call rejection"""
    call_id = message.get("call_id")
    target_id = message.get("target_id")

    if target_id:
        await signaling_manager.send_to_user(
            target_id,
            {
                "type": "call-reject",
                "call_id": call_id,
                "user_id": user_id,
                "reason": message.get("reason")
            }
        )

async def handle_hangup(message: dict, user_id: int, db: AsyncSession):
    """Handle call hangup"""
    call_id = message.get("call_id")

    # Remove from call
    signaling_manager.leave_call(user_id, call_id)

    # End call in database
    await SignalingService.end_call(call_id, db)
    await SignalingService.remove_participant(call_id, user_id, db)

    # Notify other participants
    participants = signaling_manager.get_call_participants(call_id)
    for participant_id in participants:
        await signaling_manager.send_to_user(
            participant_id,
            {
                "type": "hangup",
                "call_id": call_id,
                "user_id": user_id
            }
        )

async def handle_media_state(message: dict, user_id: int):
    """Handle media state change"""
    call_id = message.get("call_id")

    # Broadcast to other participants
    participants = signaling_manager.get_call_participants(call_id)
    for participant_id in participants:
        if participant_id != user_id:
            await signaling_manager.send_to_user(
                participant_id,
                {
                    "type": "media-state",
                    "call_id": call_id,
                    "user_id": user_id,
                    "is_video_enabled": message.get("is_video_enabled"),
                    "is_audio_enabled": message.get("is_audio_enabled")
                }
            )

async def handle_join_room(
    message: dict,
    user_id: int,
    username: str,
    db: AsyncSession
):
    """Handle joining video room"""
    room_id = message.get("room_id")

    # Join room
    signaling_manager.join_room(user_id, room_id)

    # Get current participants
    participants = signaling_manager.get_room_participants(room_id)

    # Send participant list to joiner
    await signaling_manager.send_to_user(
        user_id,
        {
            "type": "room-participants",
            "room_id": room_id,
            "participants": [{"user_id": p} for p in participants if p != user_id]
        }
    )

    # Notify others
    await signaling_manager.broadcast_to_room(
        room_id,
        {
            "type": "user-joined",
            "room_id": room_id,
            "user_id": user_id,
            "username": username
        },
        exclude_user=user_id
    )

async def handle_leave_room(message: dict, user_id: int):
    """Handle leaving video room"""
    room_id = message.get("room_id")

    # Notify others before leaving
    await signaling_manager.broadcast_to_room(
        room_id,
        {
            "type": "user-left",
            "room_id": room_id,
            "user_id": user_id
        },
        exclude_user=user_id
    )

    # Leave room
    signaling_manager.leave_room(user_id, room_id)
```

### Step 9: Frontend WebRTC Client (static/js/webrtc.js)
```javascript
class WebRTCClient {
    constructor(signalingUrl, token) {
        this.signalingUrl = signalingUrl;
        this.token = token;
        this.ws = null;
        this.peerConnections = {};
        this.localStream = null;
        this.iceServers = [];
    }

    async connect() {
        return new Promise((resolve, reject) => {
            this.ws = new WebSocket(`${this.signalingUrl}?token=${this.token}`);

            this.ws.onopen = () => {
                console.log('Signaling connected');
                resolve();
            };

            this.ws.onmessage = (event) => {
                this.handleSignalingMessage(JSON.parse(event.data));
            };

            this.ws.onerror = (error) => {
                console.error('Signaling error:', error);
                reject(error);
            };

            this.ws.onclose = () => {
                console.log('Signaling closed');
            };
        });
    }

    async handleSignalingMessage(message) {
        switch(message.type) {
            case 'config':
                this.iceServers = message.iceServers;
                break;
            case 'call-invite':
                await this.handleCallInvite(message);
                break;
            case 'offer':
                await this.handleOffer(message);
                break;
            case 'answer':
                await this.handleAnswer(message);
                break;
            case 'ice-candidate':
                await this.handleIceCandidate(message);
                break;
            case 'hangup':
                this.handleHangup(message);
                break;
        }
    }

    async startCall(targetUserId, callType = 'video') {
        // Get local media
        await this.getLocalMedia(callType);

        // Send invitation
        this.ws.send(JSON.stringify({
            type: 'call-invite',
            target_id: targetUserId,
            call_type: callType
        }));
    }

    async getLocalMedia(callType) {
        const constraints = {
            video: callType === 'video' || callType === 'screen',
            audio: true
        };

        try {
            if (callType === 'screen') {
                this.localStream = await navigator.mediaDevices.getDisplayMedia(constraints);
            } else {
                this.localStream = await navigator.mediaDevices.getUserMedia(constraints);
            }
        } catch (error) {
            console.error('Error getting media:', error);
            throw error;
        }
    }

    async createPeerConnection(peerId) {
        const pc = new RTCPeerConnection({
            iceServers: this.iceServers.map(server =>
                typeof server === 'string' ? { urls: server } : server
            )
        });

        // Add local stream tracks
        if (this.localStream) {
            this.localStream.getTracks().forEach(track => {
                pc.addTrack(track, this.localStream);
            });
        }

        // Handle ICE candidates
        pc.onicecandidate = (event) => {
            if (event.candidate) {
                this.ws.send(JSON.stringify({
                    type: 'ice-candidate',
                    target_id: peerId,
                    candidate: event.candidate
                }));
            }
        };

        // Handle remote stream
        pc.ontrack = (event) => {
            this.onRemoteStream(event.streams[0], peerId);
        };

        this.peerConnections[peerId] = pc;
        return pc;
    }

    async handleOffer(message) {
        const pc = await this.createPeerConnection(message.sender_id);

        await pc.setRemoteDescription(new RTCSessionDescription({
            type: 'offer',
            sdp: message.sdp
        }));

        const answer = await pc.createAnswer();
        await pc.setLocalDescription(answer);

        this.ws.send(JSON.stringify({
            type: 'answer',
            target_id: message.sender_id,
            sdp: answer.sdp,
            call_id: message.call_id
        }));
    }

    async handleAnswer(message) {
        const pc = this.peerConnections[message.sender_id];
        if (pc) {
            await pc.setRemoteDescription(new RTCSessionDescription({
                type: 'answer',
                sdp: message.sdp
            }));
        }
    }

    async handleIceCandidate(message) {
        const pc = this.peerConnections[message.sender_id];
        if (pc) {
            await pc.addIceCandidate(new RTCIceCandidate(message.candidate));
        }
    }

    hangup(callId) {
        // Close all peer connections
        Object.values(this.peerConnections).forEach(pc => pc.close());
        this.peerConnections = {};

        // Stop local stream
        if (this.localStream) {
            this.localStream.getTracks().forEach(track => track.stop());
            this.localStream = null;
        }

        // Send hangup signal
        this.ws.send(JSON.stringify({
            type: 'hangup',
            call_id: callId
        }));
    }

    // Override in your implementation
    onRemoteStream(stream, peerId) {
        console.log('Remote stream received from', peerId);
    }
}
```

### Step 10: Run the Application
```bash
# Start Redis
redis-server

# Start the application
uvicorn app.main:app --reload --port 8000

# Open in browser
# http://localhost:8000/static/video-call.html
```

## Expected Outputs

### 1. Call Invitation
```json
{
  "type": "call-invite",
  "call_id": "abc-123",
  "caller_id": 1,
  "caller_name": "John Doe",
  "call_type": "video"
}
```

### 2. WebRTC Offer
```json
{
  "type": "offer",
  "sender_id": 1,
  "target_id": 2,
  "sdp": "v=0\\r\\no=- ...",
  "call_id": "abc-123"
}
```

### 3. ICE Candidate
```json
{
  "type": "ice-candidate",
  "sender_id": 1,
  "target_id": 2,
  "candidate": {
    "candidate": "candidate:...",
    "sdpMid": "0",
    "sdpMLineIndex": 0
  }
}
```

## Bonus Challenges

- [ ] Add call recording functionality
- [ ] Implement bandwidth adaptation
- [ ] Create virtual backgrounds
- [ ] Add noise cancellation
- [ ] Implement call transfer
- [ ] Create waiting rooms
- [ ] Add call analytics dashboard
- [ ] Implement end-to-end encryption
- [ ] Create call quality indicators
- [ ] Add screen annotation tools
- [ ] Implement breakout rooms
- [ ] Create call transcription
- [ ] Add live captions
- [ ] Implement call queuing
- [ ] Create mobile native apps

## Resources

- [WebRTC Documentation](https://webrtc.org/getting-started/overview)
- [MDN WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [Signaling Protocol](https://www.html5rocks.com/en/tutorials/webrtc/infrastructure/)
- [STUN/TURN Servers](https://www.twilio.com/docs/stun-turn)
- [WebRTC Samples](https://webrtc.github.io/samples/)
- [Coturn Server](https://github.com/coturn/coturn)

## Success Criteria

- [ ] Peer connections establish successfully
- [ ] Video/audio streams in real-time
- [ ] ICE candidates exchange correctly
- [ ] Both STUN and TURN work
- [ ] Group calls support 10+ participants
- [ ] Call quality is smooth
- [ ] Reconnection works automatically
- [ ] Screen sharing functional
- [ ] Media controls work (mute/unmute)
- [ ] Call history tracked
- [ ] Works across different networks
- [ ] NAT traversal successful
- [ ] Low latency (< 500ms)
- [ ] Browser compatibility verified
