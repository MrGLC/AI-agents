# Project 8: Build Collaborative Whiteboard Application

## Overview
Create a real-time collaborative whiteboard where multiple users can draw, annotate, and brainstorm together in real-time. This project demonstrates building interactive canvas applications with WebSocket synchronization, conflict-free drawing operations, and real-time collaboration features similar to Miro or Figma.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Synchronize canvas operations in real-time
- Handle concurrent drawing actions
- Implement operation batching and compression
- Build drawing tools and shapes
- Create layer management
- Handle canvas state synchronization
- Implement undo/redo across users
- Build collaborative cursors
- Optimize for performance

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Database**: PostgreSQL (boards, sessions) + Redis (real-time state)
- **Canvas**: HTML5 Canvas + Fabric.js or Konva.js
- **Protocol**: WebSocket with binary support
- **Storage**: S3 or local storage for exports
- **Frontend**: React or Vue.js
- **Testing**: pytest-asyncio, canvas testing

## Project Requirements

### 1. Core Features
- Real-time drawing sync
- Multiple drawing tools (pen, shapes, text)
- Color and style customization
- Layer management
- Object selection and manipulation
- Collaborative cursors
- Undo/redo
- Board export (PNG, PDF, SVG)
- Templates and backgrounds

### 2. Drawing Tools
- Freehand drawing (pen/pencil)
- Shapes (rectangle, circle, line, arrow)
- Text annotations
- Sticky notes
- Images and uploads
- Eraser
- Selection/move tool

### 3. WebSocket Endpoints
- `WS /ws/board/{board_id}` - Whiteboard collaboration
- `GET /api/boards` - List boards
- `POST /api/boards` - Create board
- `GET /api/boards/{id}/export` - Export board
- `POST /api/boards/{id}/snapshot` - Save snapshot

### 4. Collaboration Features
- Real-time cursor tracking
- User presence indicators
- Drawing locks (optional)
- Comments and reactions
- Version history
- Access control

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir collaborative-whiteboard
cd collaborative-whiteboard

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis
pip install pillow  # For image processing
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
pillow==10.1.0
pydantic==2.5.0
python-jose[cryptography]==3.3.0
pytest==7.4.3
pytest-asyncio==0.21.1
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,board,drawing,user}.py

# Schemas
touch app/schemas/{__init__,board,drawing,websocket}.py

# Services
touch app/services/{__init__,board_service,drawing_service,export_service}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,board_handler}.py

# Core
touch app/core/{__init__,redis,security}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/whiteboard.html
mkdir static/{js,css,images}
touch static/js/{canvas.js,tools.js,whiteboard.js}

# Storage
mkdir uploads exports

# Tests
mkdir tests
touch tests/{test_drawing,test_sync,test_export}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "Collaborative Whiteboard"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./whiteboard.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Canvas Settings
    MAX_CANVAS_WIDTH: int = 10000
    MAX_CANVAS_HEIGHT: int = 10000
    DEFAULT_CANVAS_WIDTH: int = 1920
    DEFAULT_CANVAS_HEIGHT: int = 1080

    # Drawing
    MAX_OPERATIONS_BUFFER: int = 100
    OPERATION_BATCH_INTERVAL: float = 0.05  # 50ms
    MAX_UNDO_STACK: int = 50

    # Collaboration
    MAX_COLLABORATORS: int = 50
    CURSOR_UPDATE_INTERVAL: float = 0.1  # 100ms
    AUTO_SAVE_INTERVAL: int = 30  # seconds

    # Storage
    UPLOAD_PATH: str = "./uploads"
    EXPORT_PATH: str = "./exports"
    MAX_UPLOAD_SIZE: int = 10 * 1024 * 1024  # 10MB

    # WebSocket
    PING_INTERVAL: int = 30

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/board.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Boolean, JSON, Text
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Board(Base):
    __tablename__ = "boards"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(255), nullable=False)
    description = Column(Text)
    owner_id = Column(Integer, ForeignKey("users.id"))
    is_public = Column(Boolean, default=False)

    # Canvas settings
    width = Column(Integer, default=1920)
    height = Column(Integer, default=1080)
    background_color = Column(String(20), default="#ffffff")
    background_image = Column(String(500))

    # Metadata
    thumbnail_url = Column(String(500))
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    owner = relationship("User", back_populates="boards")
    objects = relationship("DrawingObject", back_populates="board", cascade="all, delete-orphan")
    collaborators = relationship("BoardCollaborator", back_populates="board")
    snapshots = relationship("BoardSnapshot", back_populates="board")

class BoardCollaborator(Base):
    __tablename__ = "board_collaborators"

    id = Column(Integer, primary_key=True, index=True)
    board_id = Column(Integer, ForeignKey("boards.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    permission = Column(String(20), default="edit")  # view, edit, admin
    joined_at = Column(DateTime, default=datetime.utcnow)

    board = relationship("Board", back_populates="collaborators")
    user = relationship("User")

class BoardSnapshot(Base):
    __tablename__ = "board_snapshots"

    id = Column(Integer, primary_key=True, index=True)
    board_id = Column(Integer, ForeignKey("boards.id"))
    snapshot_data = Column(JSON, nullable=False)  # Full board state
    created_by = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)

    board = relationship("Board", back_populates="snapshots")
```

**app/models/drawing.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, JSON, Text, Float
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class DrawingObject(Base):
    __tablename__ = "drawing_objects"

    id = Column(Integer, primary_key=True, index=True)
    board_id = Column(Integer, ForeignKey("boards.id"), nullable=False, index=True)
    object_id = Column(String(100), unique=True, index=True)  # Client-generated UUID

    # Object properties
    object_type = Column(String(50), nullable=False)  # path, rect, circle, text, image
    data = Column(JSON, nullable=False)  # Object-specific data

    # Positioning
    x = Column(Float, default=0)
    y = Column(Float, default=0)
    width = Column(Float, nullable=True)
    height = Column(Float, nullable=True)
    rotation = Column(Float, default=0)

    # Styling
    stroke = Column(String(20))
    stroke_width = Column(Float)
    fill = Column(String(20))
    opacity = Column(Float, default=1.0)

    # Layer
    z_index = Column(Integer, default=0)
    layer = Column(String(50), default="default")

    # Metadata
    created_by = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    board = relationship("Board", back_populates="objects")
    creator = relationship("User")

class DrawingOperation(Base):
    __tablename__ = "drawing_operations"

    id = Column(Integer, primary_key=True, index=True)
    board_id = Column(Integer, ForeignKey("boards.id"), index=True)
    operation_type = Column(String(20), nullable=False)  # create, update, delete, move
    object_id = Column(String(100), nullable=False)
    operation_data = Column(JSON)
    user_id = Column(Integer, ForeignKey("users.id"))
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)
```

### Step 5: Drawing Schemas (app/schemas/drawing.py)
```python
from pydantic import BaseModel
from typing import Optional, Dict, Any, List, Literal
from datetime import datetime

class DrawingObjectBase(BaseModel):
    object_id: str
    object_type: str
    data: Dict[str, Any]
    x: float = 0
    y: float = 0
    width: Optional[float] = None
    height: Optional[float] = None
    rotation: float = 0
    stroke: Optional[str] = None
    stroke_width: Optional[float] = None
    fill: Optional[str] = None
    opacity: float = 1.0
    z_index: int = 0
    layer: str = "default"

class DrawingObjectCreate(DrawingObjectBase):
    pass

class DrawingObjectResponse(DrawingObjectBase):
    id: int
    board_id: int
    created_by: int
    created_at: datetime

    class Config:
        from_attributes = True

class DrawingOperation(BaseModel):
    type: Literal["create", "update", "delete", "move"]
    object_id: str
    data: Optional[Dict[str, Any]] = None
```

**app/schemas/websocket.py**:
```python
from pydantic import BaseModel
from typing import Optional, Dict, Any, List, Literal

class WSDrawingOperation(BaseModel):
    """Drawing operation message"""
    type: Literal["draw_operation"] = "draw_operation"
    operation: str  # create, update, delete, move
    object_id: str
    object_type: Optional[str] = None
    data: Dict[str, Any]
    user_id: int

class WSCursorUpdate(BaseModel):
    """Cursor position update"""
    type: Literal["cursor"] = "cursor"
    user_id: int
    username: str
    x: float
    y: float
    color: str

class WSUserJoined(BaseModel):
    """User joined board"""
    type: Literal["user_joined"] = "user_joined"
    user_id: int
    username: str
    color: str

class WSBoardState(BaseModel):
    """Full board state"""
    type: Literal["board_state"] = "board_state"
    objects: List[Dict[str, Any]]
    collaborators: List[Dict[str, Any]]

class WSUndo(BaseModel):
    """Undo operation"""
    type: Literal["undo"] = "undo"
    user_id: int
    object_id: Optional[str] = None

class WSRedo(BaseModel):
    """Redo operation"""
    type: Literal["redo"] = "redo"
    user_id: int
    object_id: Optional[str] = None
```

### Step 6: Drawing Service (app/services/drawing_service.py)
```python
from typing import List, Dict, Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_
from datetime import datetime
import json

from app.models.drawing import DrawingObject, DrawingOperation
from app.models.board import Board
from app.schemas.drawing import DrawingObjectCreate
from app.core.redis import redis_client

class DrawingService:
    """Service for managing drawing operations"""

    @staticmethod
    async def create_object(
        board_id: int,
        object_data: DrawingObjectCreate,
        user_id: int,
        db: AsyncSession
    ) -> DrawingObject:
        """Create a new drawing object"""
        # Create object
        obj = DrawingObject(
            board_id=board_id,
            object_id=object_data.object_id,
            object_type=object_data.object_type,
            data=object_data.data,
            x=object_data.x,
            y=object_data.y,
            width=object_data.width,
            height=object_data.height,
            rotation=object_data.rotation,
            stroke=object_data.stroke,
            stroke_width=object_data.stroke_width,
            fill=object_data.fill,
            opacity=object_data.opacity,
            z_index=object_data.z_index,
            layer=object_data.layer,
            created_by=user_id
        )

        db.add(obj)

        # Log operation
        operation = DrawingOperation(
            board_id=board_id,
            operation_type="create",
            object_id=object_data.object_id,
            operation_data=object_data.dict(),
            user_id=user_id
        )
        db.add(operation)

        await db.commit()
        await db.refresh(obj)

        # Cache in Redis
        await DrawingService._cache_object(obj)

        return obj

    @staticmethod
    async def update_object(
        object_id: str,
        updates: Dict,
        user_id: int,
        db: AsyncSession
    ) -> Optional[DrawingObject]:
        """Update drawing object"""
        result = await db.execute(
            select(DrawingObject).where(DrawingObject.object_id == object_id)
        )
        obj = result.scalar_one_or_none()

        if not obj:
            return None

        # Apply updates
        for key, value in updates.items():
            if hasattr(obj, key):
                setattr(obj, key, value)

        obj.updated_at = datetime.utcnow()

        # Log operation
        operation = DrawingOperation(
            board_id=obj.board_id,
            operation_type="update",
            object_id=object_id,
            operation_data=updates,
            user_id=user_id
        )
        db.add(operation)

        await db.commit()
        await db.refresh(obj)

        # Update cache
        await DrawingService._cache_object(obj)

        return obj

    @staticmethod
    async def delete_object(
        object_id: str,
        user_id: int,
        db: AsyncSession
    ) -> bool:
        """Delete drawing object"""
        result = await db.execute(
            select(DrawingObject).where(DrawingObject.object_id == object_id)
        )
        obj = result.scalar_one_or_none()

        if not obj:
            return False

        # Log operation before deleting
        operation = DrawingOperation(
            board_id=obj.board_id,
            operation_type="delete",
            object_id=object_id,
            user_id=user_id
        )
        db.add(operation)

        # Delete object
        await db.delete(obj)
        await db.commit()

        # Remove from cache
        await DrawingService._remove_from_cache(obj.board_id, object_id)

        return True

    @staticmethod
    async def get_board_objects(
        board_id: int,
        db: AsyncSession
    ) -> List[DrawingObject]:
        """Get all objects on board"""
        result = await db.execute(
            select(DrawingObject)
            .where(DrawingObject.board_id == board_id)
            .order_by(DrawingObject.z_index, DrawingObject.created_at)
        )
        return result.scalars().all()

    @staticmethod
    async def _cache_object(obj: DrawingObject):
        """Cache object in Redis"""
        key = f"board:{obj.board_id}:object:{obj.object_id}"
        data = {
            "id": obj.id,
            "object_id": obj.object_id,
            "object_type": obj.object_type,
            "data": obj.data,
            "x": obj.x,
            "y": obj.y,
            "width": obj.width,
            "height": obj.height,
            "rotation": obj.rotation,
            "stroke": obj.stroke,
            "stroke_width": obj.stroke_width,
            "fill": obj.fill,
            "opacity": obj.opacity,
            "z_index": obj.z_index
        }
        await redis_client.redis.setex(
            key,
            3600,  # 1 hour
            json.dumps(data)
        )

    @staticmethod
    async def _remove_from_cache(board_id: int, object_id: str):
        """Remove object from cache"""
        key = f"board:{board_id}:object:{object_id}"
        await redis_client.redis.delete(key)
```

### Step 7: WebSocket Handler (app/websocket/board_handler.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json
import uuid
from collections import defaultdict

from app.websocket.connection_manager import whiteboard_manager
from app.services.drawing_service import DrawingService
from app.models.board import Board
from app.models.user import User
from app.core.security import decode_access_token
from app.db.session import get_db
from app.schemas.drawing import DrawingObjectCreate

# User colors for cursors
USER_COLORS = [
    "#FF6B6B", "#4ECDC4", "#45B7D1", "#FFA07A",
    "#98D8C8", "#F7DC6F", "#BB8FCE", "#85C1E2"
]

color_assignments = {}

async def handle_whiteboard(
    websocket: WebSocket,
    board_id: int,
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
):
    """Handle whiteboard collaboration WebSocket"""
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

    # Get board
    result = await db.execute(select(Board).where(Board.id == board_id))
    board = result.scalar_one_or_none()

    if not board:
        await websocket.close(code=1008, reason="Board not found")
        return

    # Assign color
    if user_id not in color_assignments:
        color_assignments[user_id] = USER_COLORS[
            len(color_assignments) % len(USER_COLORS)
        ]

    user_color = color_assignments[user_id]

    # Connect
    connection_id = str(uuid.uuid4())
    await whiteboard_manager.connect(board_id, user_id, user.username, websocket, user_color)

    # Send initial board state
    objects = await DrawingService.get_board_objects(board_id, db)
    collaborators = whiteboard_manager.get_collaborators(board_id)

    await websocket.send_json({
        "type": "board_state",
        "objects": [
            {
                "object_id": obj.object_id,
                "object_type": obj.object_type,
                "data": obj.data,
                "x": obj.x,
                "y": obj.y,
                "width": obj.width,
                "height": obj.height,
                "rotation": obj.rotation,
                "stroke": obj.stroke,
                "stroke_width": obj.stroke_width,
                "fill": obj.fill,
                "opacity": obj.opacity,
                "z_index": obj.z_index
            }
            for obj in objects
        ],
        "collaborators": collaborators
    })

    # Notify others
    await whiteboard_manager.broadcast(
        board_id,
        {
            "type": "user_joined",
            "user_id": user_id,
            "username": user.username,
            "color": user_color
        },
        exclude_user=user_id
    )

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "draw_operation":
                # Handle drawing operation
                await handle_draw_operation(
                    message,
                    board_id,
                    user_id,
                    db
                )

            elif msg_type == "cursor":
                # Update cursor position
                await whiteboard_manager.broadcast(
                    board_id,
                    {
                        "type": "cursor",
                        "user_id": user_id,
                        "username": user.username,
                        "x": message.get("x"),
                        "y": message.get("y"),
                        "color": user_color
                    },
                    exclude_user=user_id
                )

            elif msg_type == "undo":
                # Handle undo
                # Implementation depends on undo strategy
                pass

            elif msg_type == "redo":
                # Handle redo
                pass

    except WebSocketDisconnect:
        whiteboard_manager.disconnect(board_id, user_id)

        # Notify others
        await whiteboard_manager.broadcast(
            board_id,
            {
                "type": "user_left",
                "user_id": user_id,
                "username": user.username
            }
        )

async def handle_draw_operation(
    message: dict,
    board_id: int,
    user_id: int,
    db: AsyncSession
):
    """Handle drawing operation"""
    operation = message.get("operation")
    object_id = message.get("object_id")
    data = message.get("data", {})

    if operation == "create":
        # Create new object
        obj_data = DrawingObjectCreate(
            object_id=object_id,
            object_type=message.get("object_type"),
            data=data,
            **{k: v for k, v in data.items() if k in [
                "x", "y", "width", "height", "rotation",
                "stroke", "stroke_width", "fill", "opacity", "z_index", "layer"
            ]}
        )
        await DrawingService.create_object(board_id, obj_data, user_id, db)

    elif operation == "update":
        # Update existing object
        await DrawingService.update_object(object_id, data, user_id, db)

    elif operation == "delete":
        # Delete object
        await DrawingService.delete_object(object_id, user_id, db)

    # Broadcast to other users
    await whiteboard_manager.broadcast(
        board_id,
        {
            "type": "draw_operation",
            "operation": operation,
            "object_id": object_id,
            "object_type": message.get("object_type"),
            "data": data,
            "user_id": user_id
        },
        exclude_user=user_id
    )
```

### Step 8: Run Application
```bash
# Start Redis
redis-server

# Start application
uvicorn app.main:app --reload --port 8000

# Open whiteboard
# http://localhost:8000/static/whiteboard.html
```

## Expected Outputs

### 1. Drawing Operation
```json
{
  "type": "draw_operation",
  "operation": "create",
  "object_id": "rect-123",
  "object_type": "rectangle",
  "data": {
    "x": 100,
    "y": 200,
    "width": 150,
    "height": 100,
    "fill": "#FF6B6B",
    "stroke": "#000000",
    "stroke_width": 2
  },
  "user_id": 1
}
```

### 2. Cursor Update
```json
{
  "type": "cursor",
  "user_id": 2,
  "username": "jane_doe",
  "x": 350,
  "y": 420,
  "color": "#4ECDC4"
}
```

## Bonus Challenges

- [ ] Add shape snapping/alignment
- [ ] Implement grouping/ungrouping
- [ ] Create drawing animations
- [ ] Add voice/video chat integration
- [ ] Implement presentation mode
- [ ] Create templates library
- [ ] Add real-time commenting
- [ ] Implement version control
- [ ] Create mobile touch support
- [ ] Add offline mode
- [ ] Implement permissions per layer
- [ ] Create drawing playback
- [ ] Add AI auto-complete shapes
- [ ] Implement collaborative filters
- [ ] Create board embedding

## Resources

- [HTML5 Canvas Tutorial](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)
- [Fabric.js Documentation](http://fabricjs.com/)
- [Konva.js](https://konvajs.org/)
- [WebSocket Canvas Sync](https://ably.com/blog/realtime-collaborative-whiteboard)
- [Operational Transformation](https://operational-transformation.github.io/)

## Success Criteria

- [ ] Drawing syncs in real-time
- [ ] Multiple users can draw simultaneously
- [ ] No drawing conflicts
- [ ] Cursors track accurately
- [ ] Undo/redo works correctly
- [ ] Objects maintain proper layering
- [ ] Export generates correct images
- [ ] Handles 20+ concurrent users
- [ ] Low latency (< 100ms)
- [ ] No canvas artifacts
- [ ] Smooth drawing performance
- [ ] Board state persists correctly
- [ ] Mobile/touch works properly
- [ ] All tools functional
