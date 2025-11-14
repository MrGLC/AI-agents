# Project 3: Build Multiplayer Game Server with Rooms

## Overview
Create a real-time multiplayer game server with WebSocket support for room-based games, player matchmaking, game state synchronization, and anti-cheat mechanisms. This project demonstrates building scalable game infrastructure with FastAPI, handling real-time player interactions, and managing game sessions.

## Difficulty Level
Advanced

## Learning Objectives
- Implement room-based multiplayer architecture
- Handle real-time game state synchronization
- Build matchmaking and lobby systems
- Implement server-authoritative game logic
- Handle player latency and prediction
- Create anti-cheat mechanisms
- Manage game sessions and reconnections
- Build spectator mode
- Implement game replays

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Game Engine**: Custom (server-side)
- **Database**: PostgreSQL (games, players) + Redis (real-time state)
- **State Management**: Server-authoritative
- **Protocol**: WebSocket with binary support
- **Frontend**: HTML5 Canvas + JavaScript
- **Testing**: pytest-asyncio, load testing

## Project Requirements

### 1. Core Features
- Room creation and joining
- Player matchmaking
- Real-time game state sync
- Server-authoritative gameplay
- Player authentication
- Spectator mode
- Game replays
- Leaderboards
- Player statistics

### 2. Game Types (Example: Simple Battle Game)
- Turn-based combat
- Real-time action
- Team-based gameplay
- Free-for-all mode
- Custom game modes

### 3. WebSocket Endpoints
- `WS /ws/game/lobby` - Lobby connection
- `WS /ws/game/room/{room_id}` - Game room
- `WS /ws/game/spectate/{room_id}` - Spectate game
- `GET /api/rooms` - List active rooms
- `POST /api/rooms` - Create room
- `GET /api/leaderboard` - Get leaderboard

### 4. Message Types
- Player input (moves, actions)
- Game state updates
- Player joined/left
- Game started/ended
- Chat messages
- Game events (damage, score, etc.)

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir multiplayer-game-server
cd multiplayer-game-server

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
pip install numpy  # For game calculations

cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
aiosqlite==0.19.0
redis==5.0.1
python-jose[cryptography]==3.3.0
pydantic==2.5.0
numpy==1.26.0
pytest==7.4.3
pytest-asyncio==0.21.1
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket,game}
touch app/__init__.py app/main.py app/config.py

# Game logic
touch app/game/{__init__,engine,room,player,state}.py

# Models
touch app/models/{__init__,user,game_session,player_stats}.py

# Schemas
touch app/schemas/{__init__,game,websocket}.py

# Services
touch app/services/{__init__,matchmaking,game_manager}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,handlers}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/{game.html,lobby.html}

# Tests
mkdir tests
touch tests/{test_game,test_matchmaking,test_sync}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    APP_NAME: str = "Multiplayer Game Server"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "sqlite+aiosqlite:///./game.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Game Settings
    MAX_PLAYERS_PER_ROOM: int = 4
    MAX_SPECTATORS_PER_ROOM: int = 10
    GAME_TICK_RATE: float = 0.016  # ~60 FPS
    MAX_ROOMS: int = 1000
    ROOM_IDLE_TIMEOUT: int = 300  # 5 minutes

    # Matchmaking
    MATCHMAKING_TIMEOUT: int = 30
    SKILL_RANGE: int = 200  # ELO range for matchmaking

    # Game Mechanics (example: battle game)
    PLAYER_MAX_HP: int = 100
    ATTACK_COOLDOWN: float = 1.0
    GAME_DURATION: int = 180  # 3 minutes

    # WebSocket
    PING_INTERVAL: int = 30
    MAX_MESSAGE_SIZE: int = 1024 * 10  # 10KB

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/game_session.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, JSON, Float
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class GameSession(Base):
    __tablename__ = "game_sessions"

    id = Column(Integer, primary_key=True, index=True)
    room_id = Column(String(50), unique=True, index=True)
    game_mode = Column(String(50), default="battle")
    status = Column(String(20), default="waiting")  # waiting, playing, finished
    max_players = Column(Integer, default=4)
    current_players = Column(Integer, default=0)
    winner_id = Column(Integer, ForeignKey("users.id"), nullable=True)
    started_at = Column(DateTime, nullable=True)
    ended_at = Column(DateTime, nullable=True)
    duration = Column(Float, nullable=True)
    game_data = Column(JSON)  # Store replay data
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    players = relationship("PlayerSession", back_populates="game")
    winner = relationship("User", foreign_keys=[winner_id])

class PlayerSession(Base):
    __tablename__ = "player_sessions"

    id = Column(Integer, primary_key=True, index=True)
    game_id = Column(Integer, ForeignKey("game_sessions.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    team = Column(Integer, nullable=True)
    score = Column(Integer, default=0)
    kills = Column(Integer, default=0)
    deaths = Column(Integer, default=0)
    is_winner = Column(Boolean, default=False)
    joined_at = Column(DateTime, default=datetime.utcnow)
    left_at = Column(DateTime, nullable=True)

    # Relationships
    game = relationship("GameSession", back_populates="players")
    user = relationship("User")

class PlayerStats(Base):
    __tablename__ = "player_stats"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)
    games_played = Column(Integer, default=0)
    games_won = Column(Integer, default=0)
    total_kills = Column(Integer, default=0)
    total_deaths = Column(Integer, default=0)
    total_score = Column(Integer, default=0)
    elo_rating = Column(Integer, default=1000)
    updated_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="stats")
```

### Step 5: Game Engine (app/game/engine.py)
```python
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from datetime import datetime
import asyncio
import uuid

@dataclass
class PlayerState:
    """Player state in game"""
    id: int
    username: str
    position: tuple = (0, 0)
    hp: int = 100
    score: int = 0
    kills: int = 0
    deaths: int = 0
    is_alive: bool = True
    last_attack_time: float = 0
    team: Optional[int] = None

@dataclass
class GameState:
    """Overall game state"""
    room_id: str
    players: Dict[int, PlayerState] = field(default_factory=dict)
    status: str = "waiting"  # waiting, playing, finished
    started_at: Optional[datetime] = None
    ended_at: Optional[datetime] = None
    winner_id: Optional[int] = None
    current_tick: int = 0
    game_mode: str = "battle"

class GameEngine:
    """Server-authoritative game engine"""

    def __init__(self, room_id: str, game_mode: str = "battle"):
        self.room_id = room_id
        self.game_mode = game_mode
        self.state = GameState(room_id=room_id, game_mode=game_mode)
        self.tick_rate = 0.016  # ~60 FPS
        self.is_running = False
        self.game_loop_task: Optional[asyncio.Task] = None

    async def start_game(self):
        """Start game loop"""
        if self.is_running:
            return

        self.state.status = "playing"
        self.state.started_at = datetime.utcnow()
        self.is_running = True

        # Start game loop
        self.game_loop_task = asyncio.create_task(self._game_loop())

    async def stop_game(self):
        """Stop game loop"""
        self.is_running = False
        self.state.status = "finished"
        self.state.ended_at = datetime.utcnow()

        if self.game_loop_task:
            self.game_loop_task.cancel()
            try:
                await self.game_loop_task
            except asyncio.CancelledError:
                pass

    async def _game_loop(self):
        """Main game loop"""
        try:
            while self.is_running:
                start_time = asyncio.get_event_loop().time()

                # Update game state
                self.update()

                # Check win conditions
                winner = self.check_win_condition()
                if winner:
                    self.state.winner_id = winner
                    await self.stop_game()
                    break

                # Sleep for remaining tick time
                elapsed = asyncio.get_event_loop().time() - start_time
                sleep_time = max(0, self.tick_rate - elapsed)
                await asyncio.sleep(sleep_time)

                self.state.current_tick += 1

        except asyncio.CancelledError:
            pass

    def update(self):
        """Update game state (called every tick)"""
        # Physics updates
        self.update_positions()

        # Check collisions
        self.check_collisions()

        # Update timers
        self.update_timers()

    def update_positions(self):
        """Update player positions"""
        # Implement movement logic
        pass

    def check_collisions(self):
        """Check for collisions between players"""
        # Implement collision detection
        pass

    def update_timers(self):
        """Update cooldown timers"""
        current_time = asyncio.get_event_loop().time()

        for player in self.state.players.values():
            # Update attack cooldowns, etc.
            pass

    def handle_player_input(self, player_id: int, action: Dict) -> Dict:
        """
        Handle player input (server-authoritative)
        Returns the validated action result
        """
        player = self.state.players.get(player_id)
        if not player or not player.is_alive:
            return {"error": "Invalid player"}

        action_type = action.get("type")

        if action_type == "move":
            return self.handle_move(player_id, action)
        elif action_type == "attack":
            return self.handle_attack(player_id, action)
        elif action_type == "ability":
            return self.handle_ability(player_id, action)

        return {"error": "Unknown action"}

    def handle_move(self, player_id: int, action: Dict) -> Dict:
        """Handle player movement"""
        player = self.state.players[player_id]

        # Validate and apply movement
        direction = action.get("direction")  # {x, y}
        speed = 5

        # Validate movement (anti-cheat)
        if abs(direction.get("x", 0)) > 1 or abs(direction.get("y", 0)) > 1:
            return {"error": "Invalid movement"}

        # Update position
        new_x = player.position[0] + direction.get("x", 0) * speed
        new_y = player.position[1] + direction.get("y", 0) * speed

        # Clamp to bounds (0-1000)
        new_x = max(0, min(1000, new_x))
        new_y = max(0, min(1000, new_y))

        player.position = (new_x, new_y)

        return {
            "success": True,
            "position": player.position
        }

    def handle_attack(self, player_id: int, action: Dict) -> Dict:
        """Handle player attack"""
        from app.config import settings

        player = self.state.players[player_id]
        current_time = asyncio.get_event_loop().time()

        # Check cooldown
        if current_time - player.last_attack_time < settings.ATTACK_COOLDOWN:
            return {"error": "Attack on cooldown"}

        target_id = action.get("target_id")
        target = self.state.players.get(target_id)

        if not target or not target.is_alive:
            return {"error": "Invalid target"}

        # Check range (simple distance check)
        distance = self.calculate_distance(player.position, target.position)
        if distance > 100:  # Attack range
            return {"error": "Target out of range"}

        # Apply damage
        damage = 10
        target.hp -= damage

        # Update stats
        player.last_attack_time = current_time

        # Check if target died
        if target.hp <= 0:
            target.is_alive = False
            target.hp = 0
            player.kills += 1
            target.deaths += 1
            player.score += 100

        return {
            "success": True,
            "damage": damage,
            "target_id": target_id,
            "target_hp": target.hp,
            "kill": not target.is_alive
        }

    def handle_ability(self, player_id: int, action: Dict) -> Dict:
        """Handle special ability"""
        # Implement special abilities
        return {"success": True}

    def calculate_distance(self, pos1: tuple, pos2: tuple) -> float:
        """Calculate distance between two positions"""
        import math
        return math.sqrt((pos1[0] - pos2[0])**2 + (pos1[1] - pos2[1])**2)

    def check_win_condition(self) -> Optional[int]:
        """Check if someone won"""
        alive_players = [p for p in self.state.players.values() if p.is_alive]

        # Last player standing
        if len(alive_players) == 1:
            return alive_players[0].id

        # Time limit reached (highest score wins)
        if self.state.started_at:
            duration = (datetime.utcnow() - self.state.started_at).total_seconds()
            if duration > 180:  # 3 minutes
                # Find highest score
                max_score = max(p.score for p in self.state.players.values())
                winner = next(
                    p for p in self.state.players.values()
                    if p.score == max_score
                )
                return winner.id

        return None

    def add_player(self, user_id: int, username: str, team: Optional[int] = None):
        """Add player to game"""
        player = PlayerState(
            id=user_id,
            username=username,
            team=team,
            position=(50 + user_id * 100, 50 + user_id * 100)  # Spawn positions
        )
        self.state.players[user_id] = player

    def remove_player(self, user_id: int):
        """Remove player from game"""
        if user_id in self.state.players:
            del self.state.players[user_id]

    def get_state_snapshot(self) -> Dict:
        """Get full game state for sync"""
        return {
            "room_id": self.room_id,
            "status": self.state.status,
            "tick": self.state.current_tick,
            "players": {
                pid: {
                    "id": p.id,
                    "username": p.username,
                    "position": p.position,
                    "hp": p.hp,
                    "score": p.score,
                    "kills": p.kills,
                    "deaths": p.deaths,
                    "is_alive": p.is_alive,
                    "team": p.team
                }
                for pid, p in self.state.players.items()
            }
        }
```

### Step 6: Game Room Manager (app/game/room.py)
```python
from typing import Dict, Set, Optional
import uuid
from app.game.engine import GameEngine

class GameRoom:
    """Represents a game room"""

    def __init__(
        self,
        room_id: str,
        max_players: int = 4,
        game_mode: str = "battle"
    ):
        self.room_id = room_id
        self.max_players = max_players
        self.game_mode = game_mode
        self.players: Set[int] = set()
        self.spectators: Set[int] = set()
        self.engine = GameEngine(room_id, game_mode)
        self.is_full = False

    def add_player(self, user_id: int, username: str) -> bool:
        """Add player to room"""
        if len(self.players) >= self.max_players:
            return False

        self.players.add(user_id)
        self.engine.add_player(user_id, username)

        if len(self.players) >= self.max_players:
            self.is_full = True

        return True

    def remove_player(self, user_id: int):
        """Remove player from room"""
        if user_id in self.players:
            self.players.discard(user_id)
            self.engine.remove_player(user_id)
            self.is_full = False

    def add_spectator(self, user_id: int) -> bool:
        """Add spectator"""
        from app.config import settings

        if len(self.spectators) >= settings.MAX_SPECTATORS_PER_ROOM:
            return False

        self.spectators.add(user_id)
        return True

    def remove_spectator(self, user_id: int):
        """Remove spectator"""
        self.spectators.discard(user_id)

    async def start_game(self):
        """Start the game"""
        await self.engine.start_game()

    async def stop_game(self):
        """Stop the game"""
        await self.engine.stop_game()

    def is_empty(self) -> bool:
        """Check if room is empty"""
        return len(self.players) == 0 and len(self.spectators) == 0

class RoomManager:
    """Manages all game rooms"""

    def __init__(self):
        self.rooms: Dict[str, GameRoom] = {}

    def create_room(
        self,
        max_players: int = 4,
        game_mode: str = "battle"
    ) -> GameRoom:
        """Create new game room"""
        room_id = str(uuid.uuid4())[:8]
        room = GameRoom(room_id, max_players, game_mode)
        self.rooms[room_id] = room
        return room

    def get_room(self, room_id: str) -> Optional[GameRoom]:
        """Get room by ID"""
        return self.rooms.get(room_id)

    def delete_room(self, room_id: str):
        """Delete room"""
        if room_id in self.rooms:
            del self.rooms[room_id]

    def get_available_rooms(self) -> list:
        """Get list of available rooms"""
        return [
            {
                "room_id": room.room_id,
                "players": len(room.players),
                "max_players": room.max_players,
                "game_mode": room.game_mode,
                "status": room.engine.state.status
            }
            for room in self.rooms.values()
            if not room.is_full and room.engine.state.status == "waiting"
        ]

    def cleanup_empty_rooms(self):
        """Remove empty rooms"""
        empty_rooms = [
            room_id for room_id, room in self.rooms.items()
            if room.is_empty()
        ]

        for room_id in empty_rooms:
            self.delete_room(room_id)

# Global room manager
room_manager = RoomManager()
```

### Step 7: WebSocket Connection Manager (app/websocket/connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Set
from collections import defaultdict
import json

class GameConnectionManager:
    """Manages WebSocket connections for game"""

    def __init__(self):
        # {room_id: {user_id: WebSocket}}
        self.player_connections: Dict[str, Dict[int, WebSocket]] = defaultdict(dict)

        # {room_id: {user_id: WebSocket}}
        self.spectator_connections: Dict[str, Dict[int, WebSocket]] = defaultdict(dict)

    async def connect_player(self, room_id: str, user_id: int, websocket: WebSocket):
        """Connect player to room"""
        await websocket.accept()
        self.player_connections[room_id][user_id] = websocket

    async def connect_spectator(self, room_id: str, user_id: int, websocket: WebSocket):
        """Connect spectator to room"""
        await websocket.accept()
        self.spectator_connections[room_id][user_id] = websocket

    def disconnect_player(self, room_id: str, user_id: int):
        """Disconnect player"""
        if room_id in self.player_connections:
            self.player_connections[room_id].pop(user_id, None)
            if not self.player_connections[room_id]:
                del self.player_connections[room_id]

    def disconnect_spectator(self, room_id: str, user_id: int):
        """Disconnect spectator"""
        if room_id in self.spectator_connections:
            self.spectator_connections[room_id].pop(user_id, None)
            if not self.spectator_connections[room_id]:
                del self.spectator_connections[room_id]

    async def broadcast_to_room(self, room_id: str, message: Dict, include_spectators: bool = True):
        """Broadcast message to all in room"""
        # Send to players
        if room_id in self.player_connections:
            for websocket in self.player_connections[room_id].values():
                try:
                    await websocket.send_json(message)
                except:
                    pass

        # Send to spectators
        if include_spectators and room_id in self.spectator_connections:
            for websocket in self.spectator_connections[room_id].values():
                try:
                    await websocket.send_json(message)
                except:
                    pass

    async def send_to_player(self, room_id: str, user_id: int, message: Dict):
        """Send message to specific player"""
        websocket = self.player_connections.get(room_id, {}).get(user_id)
        if websocket:
            try:
                await websocket.send_json(message)
            except:
                pass

# Global connection manager
game_connection_manager = GameConnectionManager()
```

### Step 8: WebSocket Handlers (app/websocket/handlers.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Query
import json
import asyncio

from app.websocket.connection_manager import game_connection_manager
from app.game.room import room_manager
from app.core.security import decode_access_token

async def handle_game_room(
    websocket: WebSocket,
    room_id: str,
    token: str = Query(...)
):
    """Handle game room WebSocket"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    user_id = int(payload.get("sub"))
    username = payload.get("username", f"Player{user_id}")

    # Get or create room
    room = room_manager.get_room(room_id)
    if not room:
        await websocket.close(code=1008, reason="Room not found")
        return

    # Join room
    if not room.add_player(user_id, username):
        await websocket.close(code=1008, reason="Room is full")
        return

    # Connect WebSocket
    await game_connection_manager.connect_player(room_id, user_id, websocket)

    # Send initial state
    await websocket.send_json({
        "type": "joined",
        "room_id": room_id,
        "user_id": user_id,
        "state": room.engine.get_state_snapshot()
    })

    # Broadcast player joined
    await game_connection_manager.broadcast_to_room(
        room_id,
        {
            "type": "player_joined",
            "user_id": user_id,
            "username": username
        }
    )

    # Start game if room is full
    if room.is_full:
        await room.start_game()
        await game_connection_manager.broadcast_to_room(
            room_id,
            {"type": "game_started"}
        )

        # Start broadcast loop
        asyncio.create_task(broadcast_game_state(room_id))

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "action":
                # Handle player action
                action = message.get("action")
                result = room.engine.handle_player_input(user_id, action)

                # Send action result
                await game_connection_manager.send_to_player(
                    room_id,
                    user_id,
                    {
                        "type": "action_result",
                        "result": result
                    }
                )

                # Broadcast game event if successful
                if result.get("success"):
                    await game_connection_manager.broadcast_to_room(
                        room_id,
                        {
                            "type": "game_event",
                            "user_id": user_id,
                            "action": action,
                            "result": result
                        }
                    )

    except WebSocketDisconnect:
        game_connection_manager.disconnect_player(room_id, user_id)
        room.remove_player(user_id)

        # Broadcast player left
        await game_connection_manager.broadcast_to_room(
            room_id,
            {
                "type": "player_left",
                "user_id": user_id,
                "username": username
            }
        )

        # Clean up empty rooms
        if room.is_empty():
            await room.stop_game()
            room_manager.delete_room(room_id)

async def broadcast_game_state(room_id: str):
    """Periodically broadcast game state"""
    room = room_manager.get_room(room_id)
    if not room:
        return

    try:
        while room.engine.is_running:
            # Get game state
            state = room.engine.get_state_snapshot()

            # Broadcast to all
            await game_connection_manager.broadcast_to_room(
                room_id,
                {
                    "type": "state_update",
                    "state": state
                }
            )

            # Broadcast every 100ms
            await asyncio.sleep(0.1)

    except asyncio.CancelledError:
        pass

    # Game ended, broadcast final state
    if room.engine.state.status == "finished":
        await game_connection_manager.broadcast_to_room(
            room_id,
            {
                "type": "game_ended",
                "winner_id": room.engine.state.winner_id,
                "final_state": room.engine.get_state_snapshot()
            }
        )
```

### Step 9: Run the Application
```bash
# Start Redis
redis-server

# Start the application
uvicorn app.main:app --reload --port 8000

# Open browser
# http://localhost:8000/lobby
```

## Expected Outputs

### 1. Player Joined
```json
{
  "type": "joined",
  "room_id": "a1b2c3d4",
  "user_id": 123,
  "state": {
    "room_id": "a1b2c3d4",
    "status": "waiting",
    "players": {...}
  }
}
```

### 2. Game State Update
```json
{
  "type": "state_update",
  "state": {
    "tick": 1234,
    "players": {
      "123": {
        "id": 123,
        "position": [150, 200],
        "hp": 85,
        "score": 250
      }
    }
  }
}
```

### 3. Action Result
```json
{
  "type": "action_result",
  "result": {
    "success": true,
    "damage": 10,
    "target_id": 456,
    "kill": false
  }
}
```

## Bonus Challenges

- [ ] Add different game modes
- [ ] Implement skill-based matchmaking
- [ ] Add powerups and items
- [ ] Create tournament system
- [ ] Implement anti-cheat measures
- [ ] Add player profiles and avatars
- [ ] Create replay system
- [ ] Add voice chat integration
- [ ] Implement team formations
- [ ] Add seasonal rankings
- [ ] Create custom game lobbies
- [ ] Add spectator commentary
- [ ] Implement lag compensation
- [ ] Add mobile client support
- [ ] Create AI bots for practice

## Resources

- [Multiplayer Game Architecture](https://www.gabrielgambetta.com/client-server-game-architecture.html)
- [Fast-Paced Multiplayer](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- [Lag Compensation](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods)
- [State Synchronization](https://gafferongames.com/post/state_synchronization/)
- [WebSocket Performance](https://ably.com/topic/websocket-performance)

## Success Criteria

- [ ] Players can create and join rooms
- [ ] Real-time gameplay works smoothly
- [ ] Server validates all actions
- [ ] Game state syncs across clients
- [ ] Multiple concurrent games work
- [ ] Spectator mode functional
- [ ] Matchmaking finds appropriate players
- [ ] Leaderboard tracks rankings
- [ ] Games handle disconnections gracefully
- [ ] No cheating possible (server-authoritative)
- [ ] Latency < 100ms for actions
- [ ] Supports 100+ concurrent players
- [ ] Game replays work correctly
- [ ] All game events are logged
