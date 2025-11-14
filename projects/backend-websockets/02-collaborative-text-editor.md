# Project 2: Build Live Collaborative Text Editor

## Overview
Create a real-time collaborative text editor similar to Google Docs, where multiple users can edit the same document simultaneously with live cursor positions, selections, and operational transformation (OT) or CRDT-based conflict resolution. This project teaches advanced WebSocket patterns for real-time collaboration.

## Difficulty Level
Advanced

## Learning Objectives
- Implement Operational Transformation (OT) or CRDT algorithms
- Handle concurrent edits from multiple users
- Synchronize document state across clients
- Implement cursor tracking and user presence
- Build conflict resolution mechanisms
- Handle document versioning and history
- Optimize for low-latency collaboration
- Implement undo/redo functionality

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **OT/CRDT**: py-ot or automerge-py
- **Database**: PostgreSQL (documents) + Redis (real-time state)
- **Storage**: Document versioning and snapshots
- **Frontend**: Quill.js or CodeMirror
- **Protocol**: WebSocket with custom sync protocol
- **Testing**: pytest-asyncio, concurrent testing

## Project Requirements

### 1. Core Features
- Real-time multi-user editing
- Live cursor positions and selections
- User presence indicators
- Conflict-free concurrent editing
- Document versioning and snapshots
- Undo/redo with proper OT
- Document permissions (view/edit)
- Auto-save functionality

### 2. WebSocket Endpoints
- `WS /ws/document/{doc_id}` - Document collaboration
- `GET /api/documents` - List documents
- `POST /api/documents` - Create document
- `GET /api/documents/{doc_id}/history` - Get version history
- `POST /api/documents/{doc_id}/snapshot` - Create snapshot

### 3. Operation Types
- Insert text
- Delete text
- Format text (bold, italic, etc.)
- Cursor movement
- Selection changes
- Document metadata updates

### 4. Synchronization
- Operational Transformation for conflict resolution
- Client-side buffering
- Server-side operation queue
- State vector for causality
- Snapshot mechanism for new joins

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir collaborative-editor
cd collaborative-editor

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis
pip install pydantic pydantic-settings
pip install python-jose[cryptography]
pip install pytest pytest-asyncio

# Create requirements.txt
cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
aiosqlite==0.19.0
redis==5.0.1
python-jose[cryptography]==3.3.0
pydantic==2.5.0
pydantic-settings==2.1.0
pytest==7.4.3
pytest-asyncio==0.21.1
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,document,user,operation}.py

# Schemas
touch app/schemas/{__init__,document,operation,websocket}.py

# Services
touch app/services/{__init__,ot_engine,document_sync}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,handlers}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/editor.html

# Tests
mkdir tests
touch tests/test_ot.py tests/test_collaboration.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "Collaborative Editor"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./editor.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Editor Settings
    MAX_DOCUMENT_SIZE: int = 1_000_000  # 1MB
    SNAPSHOT_INTERVAL: int = 100  # operations
    OPERATION_RETENTION: int = 1000  # keep last N ops
    MAX_COLLABORATORS: int = 50

    # WebSocket
    PING_INTERVAL: int = 30
    SYNC_INTERVAL: float = 0.1  # 100ms batching

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/document.py**:
```python
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Document(Base):
    __tablename__ = "documents"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(255), nullable=False)
    content = Column(Text, default="")
    owner_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    is_public = Column(Boolean, default=False)
    version = Column(Integer, default=0)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    owner = relationship("User", back_populates="documents")
    operations = relationship("Operation", back_populates="document")
    snapshots = relationship("DocumentSnapshot", back_populates="document")
    collaborators = relationship("DocumentCollaborator", back_populates="document")

class DocumentSnapshot(Base):
    __tablename__ = "document_snapshots"

    id = Column(Integer, primary_key=True, index=True)
    document_id = Column(Integer, ForeignKey("documents.id"), nullable=False)
    content = Column(Text, nullable=False)
    version = Column(Integer, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    document = relationship("Document", back_populates="snapshots")

class DocumentCollaborator(Base):
    __tablename__ = "document_collaborators"

    id = Column(Integer, primary_key=True, index=True)
    document_id = Column(Integer, ForeignKey("documents.id"), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    permission = Column(String(20), default="edit")  # view, edit, admin
    joined_at = Column(DateTime, default=datetime.utcnow)

    document = relationship("Document", back_populates="collaborators")
    user = relationship("User")
```

**app/models/operation.py**:
```python
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, JSON
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Operation(Base):
    __tablename__ = "operations"

    id = Column(Integer, primary_key=True, index=True)
    document_id = Column(Integer, ForeignKey("documents.id"), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    operation_type = Column(String(20), nullable=False)  # insert, delete, retain
    position = Column(Integer, nullable=False)
    content = Column(Text)
    length = Column(Integer)
    version = Column(Integer, nullable=False)
    metadata = Column(JSON)  # formatting, attributes
    created_at = Column(DateTime, default=datetime.utcnow, index=True)

    document = relationship("Document", back_populates="operations")
    user = relationship("User")
```

### Step 5: Operational Transformation Engine (app/services/ot_engine.py)
```python
from typing import List, Dict, Optional, Tuple
from dataclasses import dataclass
from enum import Enum

class OpType(Enum):
    INSERT = "insert"
    DELETE = "delete"
    RETAIN = "retain"

@dataclass
class Operation:
    """Represents a text operation"""
    op_type: OpType
    position: int
    content: Optional[str] = None
    length: Optional[int] = None
    attributes: Optional[Dict] = None

    def __str__(self):
        if self.op_type == OpType.INSERT:
            return f"Insert('{self.content}' at {self.position})"
        elif self.op_type == OpType.DELETE:
            return f"Delete({self.length} at {self.position})"
        else:
            return f"Retain({self.length} at {self.position})"

class OTEngine:
    """Operational Transformation Engine"""

    @staticmethod
    def transform(op1: Operation, op2: Operation) -> Tuple[Operation, Operation]:
        """
        Transform two concurrent operations against each other.
        Returns (op1', op2') where op1' and op2' are transformed versions.

        The transformation ensures:
        apply(op1, apply(op2, doc)) == apply(op2', apply(op1', doc))
        """
        # INSERT vs INSERT
        if op1.op_type == OpType.INSERT and op2.op_type == OpType.INSERT:
            if op1.position < op2.position:
                # op1 is before op2
                op1_prime = op1
                op2_prime = Operation(
                    op_type=op2.op_type,
                    position=op2.position + len(op1.content),
                    content=op2.content,
                    attributes=op2.attributes
                )
            elif op1.position > op2.position:
                # op1 is after op2
                op1_prime = Operation(
                    op_type=op1.op_type,
                    position=op1.position + len(op2.content),
                    content=op1.content,
                    attributes=op1.attributes
                )
                op2_prime = op2
            else:
                # Same position - use tiebreaker (e.g., user_id)
                op1_prime = op1
                op2_prime = Operation(
                    op_type=op2.op_type,
                    position=op2.position + len(op1.content),
                    content=op2.content,
                    attributes=op2.attributes
                )

        # INSERT vs DELETE
        elif op1.op_type == OpType.INSERT and op2.op_type == OpType.DELETE:
            if op1.position <= op2.position:
                # Insert before delete
                op1_prime = op1
                op2_prime = Operation(
                    op_type=op2.op_type,
                    position=op2.position + len(op1.content),
                    length=op2.length
                )
            else:
                # Insert after delete
                if op1.position >= op2.position + op2.length:
                    # Insert after deleted region
                    op1_prime = Operation(
                        op_type=op1.op_type,
                        position=op1.position - op2.length,
                        content=op1.content,
                        attributes=op1.attributes
                    )
                else:
                    # Insert inside deleted region
                    op1_prime = Operation(
                        op_type=op1.op_type,
                        position=op2.position,
                        content=op1.content,
                        attributes=op1.attributes
                    )
                op2_prime = op2

        # DELETE vs INSERT
        elif op1.op_type == OpType.DELETE and op2.op_type == OpType.INSERT:
            op2_prime, op1_prime = OTEngine.transform(op2, op1)

        # DELETE vs DELETE
        elif op1.op_type == OpType.DELETE and op2.op_type == OpType.DELETE:
            if op1.position < op2.position:
                if op1.position + op1.length <= op2.position:
                    # op1 before op2
                    op1_prime = op1
                    op2_prime = Operation(
                        op_type=op2.op_type,
                        position=op2.position - op1.length,
                        length=op2.length
                    )
                else:
                    # Overlapping deletes
                    overlap = min(op1.position + op1.length, op2.position + op2.length) - op2.position
                    op1_prime = op1
                    op2_prime = Operation(
                        op_type=op2.op_type,
                        position=op1.position,
                        length=max(0, op2.length - overlap)
                    )
            else:
                # op1 after op2
                op2_prime, op1_prime = OTEngine.transform(op2, op1)

        else:
            # RETAIN operations (no-op for now)
            op1_prime = op1
            op2_prime = op2

        return op1_prime, op2_prime

    @staticmethod
    def apply(operation: Operation, text: str) -> str:
        """Apply an operation to text"""
        if operation.op_type == OpType.INSERT:
            return (
                text[:operation.position] +
                operation.content +
                text[operation.position:]
            )
        elif operation.op_type == OpType.DELETE:
            return (
                text[:operation.position] +
                text[operation.position + operation.length:]
            )
        else:  # RETAIN
            return text

    @staticmethod
    def compose(op1: Operation, op2: Operation) -> List[Operation]:
        """Compose two sequential operations"""
        # Simplified composition
        return [op1, op2]

    @staticmethod
    def invert(operation: Operation, text: str) -> Operation:
        """Invert an operation (for undo)"""
        if operation.op_type == OpType.INSERT:
            return Operation(
                op_type=OpType.DELETE,
                position=operation.position,
                length=len(operation.content)
            )
        elif operation.op_type == OpType.DELETE:
            deleted_text = text[operation.position:operation.position + operation.length]
            return Operation(
                op_type=OpType.INSERT,
                position=operation.position,
                content=deleted_text
            )
        else:
            return operation
```

### Step 6: Document Sync Service (app/services/document_sync.py)
```python
from typing import List, Dict, Optional
import asyncio
from collections import deque

from app.services.ot_engine import OTEngine, Operation, OpType
from app.models.document import Document
from app.models.operation import Operation as DBOperation
from sqlalchemy.ext.asyncio import AsyncSession

class DocumentSync:
    """Manages document synchronization"""

    def __init__(self, document_id: int):
        self.document_id = document_id
        self.version = 0
        self.pending_operations: deque = deque()
        self.ot_engine = OTEngine()
        self.lock = asyncio.Lock()

    async def apply_operation(
        self,
        operation: Operation,
        client_version: int,
        db: AsyncSession
    ) -> Dict:
        """Apply client operation and return transformed result"""
        async with self.lock:
            # Get current document
            result = await db.execute(
                select(Document).where(Document.id == self.document_id)
            )
            document = result.scalar_one_or_none()

            if not document:
                raise ValueError("Document not found")

            # Get operations since client version
            missed_ops = await self._get_operations_since(client_version, db)

            # Transform client operation against missed operations
            transformed_op = operation
            for missed_op in missed_ops:
                transformed_op, _ = self.ot_engine.transform(
                    transformed_op,
                    missed_op
                )

            # Apply transformed operation
            new_content = self.ot_engine.apply(transformed_op, document.content)

            # Update document
            document.content = new_content
            document.version += 1
            self.version = document.version

            # Save operation to database
            db_op = DBOperation(
                document_id=self.document_id,
                operation_type=transformed_op.op_type.value,
                position=transformed_op.position,
                content=transformed_op.content,
                length=transformed_op.length,
                version=document.version
            )
            db.add(db_op)
            await db.commit()

            return {
                "operation": transformed_op,
                "version": document.version,
                "content": new_content
            }

    async def _get_operations_since(
        self,
        version: int,
        db: AsyncSession
    ) -> List[Operation]:
        """Get operations since given version"""
        result = await db.execute(
            select(DBOperation)
            .where(
                DBOperation.document_id == self.document_id,
                DBOperation.version > version
            )
            .order_by(DBOperation.version)
        )
        db_ops = result.scalars().all()

        operations = []
        for db_op in db_ops:
            op_type = OpType(db_op.operation_type)
            op = Operation(
                op_type=op_type,
                position=db_op.position,
                content=db_op.content,
                length=db_op.length
            )
            operations.append(op)

        return operations

    async def create_snapshot(self, db: AsyncSession):
        """Create document snapshot"""
        from app.models.document import DocumentSnapshot

        result = await db.execute(
            select(Document).where(Document.id == self.document_id)
        )
        document = result.scalar_one_or_none()

        if document:
            snapshot = DocumentSnapshot(
                document_id=self.document_id,
                content=document.content,
                version=document.version
            )
            db.add(snapshot)
            await db.commit()
```

### Step 7: WebSocket Connection Manager (app/websocket/connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Set
import json
from collections import defaultdict

class CollaborationManager:
    """Manages collaborative editing sessions"""

    def __init__(self):
        # {document_id: {user_id: {connection_id: WebSocket}}}
        self.connections: Dict[int, Dict[int, Dict[str, WebSocket]]] = defaultdict(
            lambda: defaultdict(dict)
        )

        # {document_id: {user_id: cursor_position}}
        self.cursors: Dict[int, Dict[int, Dict]] = defaultdict(dict)

        # {document_id: {user_id: selection}}
        self.selections: Dict[int, Dict[int, Dict]] = defaultdict(dict)

    async def connect(
        self,
        websocket: WebSocket,
        document_id: int,
        user_id: int,
        connection_id: str
    ):
        """Connect user to document"""
        await websocket.accept()
        self.connections[document_id][user_id][connection_id] = websocket

    def disconnect(self, document_id: int, user_id: int, connection_id: str):
        """Disconnect user from document"""
        if connection_id in self.connections.get(document_id, {}).get(user_id, {}):
            del self.connections[document_id][user_id][connection_id]

        # Clean up empty dictionaries
        if not self.connections[document_id][user_id]:
            del self.connections[document_id][user_id]
            if user_id in self.cursors.get(document_id, {}):
                del self.cursors[document_id][user_id]
            if user_id in self.selections.get(document_id, {}):
                del self.selections[document_id][user_id]

        if not self.connections[document_id]:
            del self.connections[document_id]

    async def broadcast_operation(
        self,
        document_id: int,
        operation: Dict,
        exclude_user: int = None
    ):
        """Broadcast operation to all users on document"""
        for user_id, connections in self.connections.get(document_id, {}).items():
            if user_id != exclude_user:
                for websocket in connections.values():
                    try:
                        await websocket.send_json(operation)
                    except:
                        pass

    async def update_cursor(self, document_id: int, user_id: int, cursor: Dict):
        """Update and broadcast cursor position"""
        self.cursors[document_id][user_id] = cursor

        cursor_msg = {
            "type": "cursor",
            "user_id": user_id,
            "position": cursor.get("position"),
            "username": cursor.get("username")
        }

        await self.broadcast_operation(document_id, cursor_msg, exclude_user=user_id)

    async def update_selection(self, document_id: int, user_id: int, selection: Dict):
        """Update and broadcast selection"""
        self.selections[document_id][user_id] = selection

        selection_msg = {
            "type": "selection",
            "user_id": user_id,
            "start": selection.get("start"),
            "end": selection.get("end"),
            "username": selection.get("username")
        }

        await self.broadcast_operation(document_id, selection_msg, exclude_user=user_id)

    def get_active_users(self, document_id: int) -> List[int]:
        """Get list of active users on document"""
        return list(self.connections.get(document_id, {}).keys())

# Global manager
collaboration_manager = CollaborationManager()
```

### Step 8: WebSocket Handlers (app/websocket/handlers.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json
import uuid

from app.websocket.connection_manager import collaboration_manager
from app.services.document_sync import DocumentSync
from app.services.ot_engine import Operation, OpType
from app.models.document import Document
from app.models.user import User
from app.db.session import get_db
from app.core.security import decode_access_token

# Cache document sync instances
document_syncs: Dict[int, DocumentSync] = {}

async def get_document_sync(document_id: int) -> DocumentSync:
    """Get or create document sync instance"""
    if document_id not in document_syncs:
        document_syncs[document_id] = DocumentSync(document_id)
    return document_syncs[document_id]

async def handle_document_collaboration(
    websocket: WebSocket,
    document_id: int,
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
):
    """Handle collaborative editing WebSocket"""
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

    # Verify document access
    result = await db.execute(select(Document).where(Document.id == document_id))
    document = result.scalar_one_or_none()

    if not document:
        await websocket.close(code=1008, reason="Document not found")
        return

    connection_id = str(uuid.uuid4())

    # Connect
    await collaboration_manager.connect(websocket, document_id, user_id, connection_id)

    # Get document sync
    doc_sync = await get_document_sync(document_id)

    # Send initial state
    await websocket.send_json({
        "type": "init",
        "content": document.content,
        "version": document.version,
        "active_users": collaboration_manager.get_active_users(document_id)
    })

    # Broadcast user joined
    await collaboration_manager.broadcast_operation(
        document_id,
        {
            "type": "user_joined",
            "user_id": user_id,
            "username": user.username
        },
        exclude_user=user_id
    )

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "operation":
                # Handle text operation
                op_data = message.get("operation")
                operation = Operation(
                    op_type=OpType(op_data["op_type"]),
                    position=op_data["position"],
                    content=op_data.get("content"),
                    length=op_data.get("length")
                )

                client_version = message.get("version")

                # Apply operation
                result = await doc_sync.apply_operation(
                    operation,
                    client_version,
                    db
                )

                # Broadcast to other clients
                await collaboration_manager.broadcast_operation(
                    document_id,
                    {
                        "type": "operation",
                        "operation": {
                            "op_type": result["operation"].op_type.value,
                            "position": result["operation"].position,
                            "content": result["operation"].content,
                            "length": result["operation"].length
                        },
                        "version": result["version"],
                        "user_id": user_id
                    },
                    exclude_user=user_id
                )

            elif msg_type == "cursor":
                # Handle cursor update
                await collaboration_manager.update_cursor(
                    document_id,
                    user_id,
                    {
                        "position": message.get("position"),
                        "username": user.username
                    }
                )

            elif msg_type == "selection":
                # Handle selection update
                await collaboration_manager.update_selection(
                    document_id,
                    user_id,
                    {
                        "start": message.get("start"),
                        "end": message.get("end"),
                        "username": user.username
                    }
                )

    except WebSocketDisconnect:
        collaboration_manager.disconnect(document_id, user_id, connection_id)

        # Broadcast user left
        await collaboration_manager.broadcast_operation(
            document_id,
            {
                "type": "user_left",
                "user_id": user_id,
                "username": user.username
            }
        )
```

### Step 9: Main Application (app/main.py)
```python
from fastapi import FastAPI, WebSocket, Depends
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from contextlib import asynccontextmanager

from app.config import settings
from app.db.session import engine
from app.db.base import Base
from app.websocket.handlers import handle_document_collaboration

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    print("Collaborative Editor Started")

    yield

    # Shutdown
    await engine.dispose()

app = FastAPI(
    title=settings.APP_NAME,
    version=settings.APP_VERSION,
    lifespan=lifespan
)

app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/")
async def root():
    return {
        "message": "Collaborative Text Editor API",
        "websocket": "ws://localhost:8000/ws/document/{document_id}?token=TOKEN"
    }

@app.websocket("/ws/document/{document_id}")
async def websocket_endpoint(
    websocket: WebSocket,
    document_id: int,
    db = Depends(get_db)
):
    await handle_document_collaboration(websocket, document_id, db)

@app.get("/editor", response_class=HTMLResponse)
async def get_editor():
    with open("static/editor.html", "r") as f:
        return HTMLResponse(content=f.read())

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

### Step 10: Run the Application
```bash
# Start the server
uvicorn app.main:app --reload --port 8000

# Open multiple browser tabs to test
# http://localhost:8000/editor
```

## Expected Outputs

### 1. Initial Connection
```json
{
  "type": "init",
  "content": "Hello world",
  "version": 5,
  "active_users": [1, 2, 3]
}
```

### 2. Operation Message
```json
{
  "type": "operation",
  "operation": {
    "op_type": "insert",
    "position": 5,
    "content": " there"
  },
  "version": 6,
  "user_id": 2
}
```

### 3. Cursor Update
```json
{
  "type": "cursor",
  "user_id": 2,
  "position": 10,
  "username": "john_doe"
}
```

## Bonus Challenges

- [ ] Add rich text formatting (bold, italic, headings)
- [ ] Implement comments and suggestions
- [ ] Add document branching and merging
- [ ] Create conflict resolution UI
- [ ] Implement offline editing support
- [ ] Add collaborative cursor indicators
- [ ] Create presence awareness (who's viewing)
- [ ] Implement document locking mechanisms
- [ ] Add real-time code highlighting
- [ ] Create version comparison view
- [ ] Implement permission levels
- [ ] Add document templates
- [ ] Create export to various formats
- [ ] Implement full-text search
- [ ] Add collaborative drawing/diagrams

## Resources

- [Operational Transformation](https://operational-transformation.github.io/)
- [CRDTs: Consistency without Concurrency Control](https://arxiv.org/abs/0907.0929)
- [Google Wave OT Paper](https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html)
- [Quill.js Documentation](https://quilljs.com/)
- [CodeMirror 6](https://codemirror.net/6/)
- [Yjs - CRDT Framework](https://docs.yjs.dev/)

## Success Criteria

- [ ] Multiple users can edit simultaneously
- [ ] All edits appear correctly on all clients
- [ ] No conflicts or data loss
- [ ] Cursor positions track accurately
- [ ] Operations apply in correct order
- [ ] Document state converges across clients
- [ ] Undo/redo works correctly
- [ ] Low latency (< 200ms for operations)
- [ ] Handles network partitions gracefully
- [ ] Supports at least 20 concurrent editors
- [ ] Document saves automatically
- [ ] Full version history maintained
