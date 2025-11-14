# Project 7: Build Real-time Auction Bidding System

## Overview
Create a real-time auction platform where users can place bids, watch live auctions, and receive instant updates on bid changes. This project demonstrates building time-sensitive WebSocket applications with transactional integrity, race condition handling, and real-time price updates.

## Difficulty Level
Advanced

## Learning Objectives
- Handle high-frequency real-time updates
- Implement optimistic locking for bids
- Build fair bidding mechanisms
- Handle race conditions in auctions
- Create auction timers and extensions
- Implement bid validation and anti-fraud
- Build real-time price discovery
- Handle auction state transitions
- Create bidding strategies and auto-bidding

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Database**: PostgreSQL (ACID transactions)
- **Cache**: Redis (real-time state, sorted sets)
- **Queue**: Celery (auction lifecycle)
- **Payment**: Stripe or PayPal integration (optional)
- **Frontend**: React or Vue.js with WebSocket
- **Testing**: pytest-asyncio, concurrent testing

## Project Requirements

### 1. Core Features
- Live auction streaming
- Real-time bid placement
- Automatic bid increments
- Proxy/auto-bidding
- Auction timers with extensions
- Reserve price handling
- Buy-now option
- Bid history tracking
- Winner determination

### 2. Auction Types
- English auction (ascending price)
- Dutch auction (descending price)
- Sealed-bid auction
- Reverse auction
- Multi-item auction

### 3. WebSocket Endpoints
- `WS /ws/auction/{auction_id}` - Live auction feed
- `WS /ws/auctions/live` - All live auctions
- `GET /api/auctions` - List auctions
- `POST /api/auctions` - Create auction
- `POST /api/auctions/{id}/bid` - Place bid
- `GET /api/auctions/{id}/bids` - Bid history

### 4. Bid Features
- Minimum bid increments
- Maximum bid (proxy bidding)
- Bid retraction (with penalties)
- Bid confirmation
- Bid notifications

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir auction-bidding-system
cd auction-bidding-system

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis celery
pip install pydantic pydantic-settings
pip install python-jose[cryptography]
pip install pytest pytest-asyncio
pip install decimal

cat > requirements.txt << EOF
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
aiosqlite==0.19.0
redis==5.0.1
celery==5.3.4
python-jose[cryptography]==3.3.0
pydantic==2.5.0
pytest==7.4.3
pytest-asyncio==0.21.1
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket,tasks}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,auction,bid,user}.py

# Schemas
touch app/schemas/{__init__,auction,bid,websocket}.py

# Services
touch app/services/{__init__,auction_service,bid_service,payment_service}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,auction_handler}.py

# Tasks
touch app/tasks/{__init__,auction_lifecycle}.py

# Core
touch app/core/{__init__,redis,security,celery}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/{auction.html,live-auctions.html}

# Tests
mkdir tests
touch tests/{test_bidding,test_auction,test_concurrency}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings
from decimal import Decimal

class Settings(BaseSettings):
    APP_NAME: str = "Auction Bidding System"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@localhost/auction"
    # For SQLite: "sqlite+aiosqlite:///./auction.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Celery
    CELERY_BROKER_URL: str = "redis://localhost:6379/1"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/2"

    # Auction Settings
    MIN_BID_INCREMENT: Decimal = Decimal("1.00")
    MIN_BID_INCREMENT_PERCENT: Decimal = Decimal("5.0")  # 5%
    AUCTION_EXTENSION_TIME: int = 120  # 2 minutes
    AUCTION_EXTENSION_TRIGGER: int = 60  # Extend if bid in last 1 minute
    MAX_AUCTION_DURATION: int = 86400 * 7  # 7 days

    # Bidding
    MAX_PROXY_BID_MULTIPLIER: int = 10
    BID_RETRACTION_PENALTY: Decimal = Decimal("10.00")
    BID_TIMEOUT: int = 5  # seconds to confirm bid

    # Payment
    PAYMENT_PROCESSING_FEE: Decimal = Decimal("2.9")  # 2.9%
    BUYER_PREMIUM: Decimal = Decimal("10.0")  # 10%

    # WebSocket
    PING_INTERVAL: int = 30
    MAX_CONNECTIONS_PER_AUCTION: int = 1000

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/auction.py**:
```python
from sqlalchemy import Column, Integer, String, Numeric, DateTime, ForeignKey, Boolean, Text, Enum as SQLEnum
from sqlalchemy.orm import relationship
from datetime import datetime
from decimal import Decimal
import enum
from app.db.base import Base

class AuctionStatus(enum.Enum):
    DRAFT = "draft"
    SCHEDULED = "scheduled"
    LIVE = "live"
    CLOSING = "closing"
    ENDED = "ended"
    CANCELLED = "cancelled"

class AuctionType(enum.Enum):
    ENGLISH = "english"  # Ascending price
    DUTCH = "dutch"  # Descending price
    SEALED = "sealed"  # Sealed bid
    REVERSE = "reverse"  # Buyers compete

class Auction(Base):
    __tablename__ = "auctions"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(255), nullable=False)
    description = Column(Text)
    seller_id = Column(Integer, ForeignKey("users.id"), nullable=False)

    # Type and status
    auction_type = Column(SQLEnum(AuctionType), default=AuctionType.ENGLISH)
    status = Column(SQLEnum(AuctionStatus), default=AuctionStatus.DRAFT, index=True)

    # Pricing
    starting_price = Column(Numeric(10, 2), nullable=False)
    current_price = Column(Numeric(10, 2), nullable=False)
    reserve_price = Column(Numeric(10, 2), nullable=True)
    buy_now_price = Column(Numeric(10, 2), nullable=True)
    increment = Column(Numeric(10, 2), nullable=False)

    # Timing
    start_time = Column(DateTime, nullable=False)
    end_time = Column(DateTime, nullable=False)
    extended_times = Column(Integer, default=0)

    # Winner
    winner_id = Column(Integer, ForeignKey("users.id"), nullable=True)
    winning_bid_id = Column(Integer, ForeignKey("bids.id"), nullable=True)

    # Flags
    has_reserve_met = Column(Boolean, default=False)
    allow_early_close = Column(Boolean, default=False)

    # Metadata
    view_count = Column(Integer, default=0)
    bid_count = Column(Integer, default=0)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    seller = relationship("User", foreign_keys=[seller_id], back_populates="auctions")
    winner = relationship("User", foreign_keys=[winner_id])
    bids = relationship("Bid", back_populates="auction", cascade="all, delete-orphan")
    winning_bid = relationship("Bid", foreign_keys=[winning_bid_id])
```

**app/models/bid.py**:
```python
from sqlalchemy import Column, Integer, Numeric, DateTime, ForeignKey, Boolean, String
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Bid(Base):
    __tablename__ = "bids"

    id = Column(Integer, primary_key=True, index=True)
    auction_id = Column(Integer, ForeignKey("auctions.id"), nullable=False, index=True)
    bidder_id = Column(Integer, ForeignKey("users.id"), nullable=False)

    # Bid amounts
    bid_amount = Column(Numeric(10, 2), nullable=False)
    max_bid_amount = Column(Numeric(10, 2), nullable=True)  # For proxy bidding
    actual_amount = Column(Numeric(10, 2), nullable=False)  # Amount actually paid

    # Status
    is_winning = Column(Boolean, default=False)
    is_outbid = Column(Boolean, default=False)
    is_proxy = Column(Boolean, default=False)
    is_retracted = Column(Boolean, default=False)

    # Timing
    placed_at = Column(DateTime, default=datetime.utcnow, index=True)
    retracted_at = Column(DateTime, nullable=True)

    # IP and fraud detection
    ip_address = Column(String(45), nullable=True)
    user_agent = Column(String(500), nullable=True)

    # Relationships
    auction = relationship("Auction", back_populates="bids")
    bidder = relationship("User", back_populates="bids")

class BidHistory(Base):
    __tablename__ = "bid_history"

    id = Column(Integer, primary_key=True, index=True)
    auction_id = Column(Integer, ForeignKey("auctions.id"), index=True)
    bid_id = Column(Integer, ForeignKey("bids.id"))
    event_type = Column(String(50), nullable=False)  # placed, outbid, won, retracted
    details = Column(String(500))
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
```

### Step 5: Bid Service (app/services/bid_service.py)
```python
from typing import Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_, func
from decimal import Decimal
from datetime import datetime, timedelta
import asyncio

from app.models.auction import Auction, AuctionStatus
from app.models.bid import Bid, BidHistory
from app.core.redis import redis_client
from app.config import settings

class BidService:
    """Service for managing bids"""

    @staticmethod
    async def place_bid(
        auction_id: int,
        bidder_id: int,
        bid_amount: Decimal,
        max_bid_amount: Optional[Decimal],
        db: AsyncSession
    ) -> dict:
        """Place a bid with race condition handling"""
        # Use database transaction for ACID guarantees
        async with db.begin():
            # Lock auction row for update
            result = await db.execute(
                select(Auction)
                .where(Auction.id == auction_id)
                .with_for_update()
            )
            auction = result.scalar_one_or_none()

            if not auction:
                return {"success": False, "error": "Auction not found"}

            # Validate auction status
            if auction.status != AuctionStatus.LIVE:
                return {"success": False, "error": "Auction is not active"}

            # Check if auction has ended
            if datetime.utcnow() > auction.end_time:
                return {"success": False, "error": "Auction has ended"}

            # Validate bidder is not seller
            if bidder_id == auction.seller_id:
                return {"success": False, "error": "Cannot bid on your own auction"}

            # Calculate minimum bid
            min_bid = await BidService._calculate_minimum_bid(auction)

            if bid_amount < min_bid:
                return {
                    "success": False,
                    "error": f"Bid must be at least {min_bid}"
                }

            # Check for proxy bidding
            if max_bid_amount and max_bid_amount < bid_amount:
                return {
                    "success": False,
                    "error": "Maximum bid must be >= bid amount"
                }

            # Get current highest bid
            current_high_bid = await BidService._get_highest_bid(auction_id, db)

            # Determine actual bid amount (for proxy bidding)
            actual_amount = bid_amount

            if current_high_bid and current_high_bid.max_bid_amount:
                # Handle proxy bid competition
                actual_amount = await BidService._calculate_proxy_bid(
                    bid_amount,
                    max_bid_amount,
                    current_high_bid,
                    auction.increment
                )

            # Create bid
            bid = Bid(
                auction_id=auction_id,
                bidder_id=bidder_id,
                bid_amount=bid_amount,
                max_bid_amount=max_bid_amount,
                actual_amount=actual_amount,
                is_proxy=max_bid_amount is not None
            )
            db.add(bid)

            # Update current high bid
            if current_high_bid:
                current_high_bid.is_winning = False
                current_high_bid.is_outbid = True

            bid.is_winning = True

            # Update auction
            auction.current_price = actual_amount
            auction.bid_count += 1

            # Check reserve price
            if auction.reserve_price and actual_amount >= auction.reserve_price:
                auction.has_reserve_met = True

            # Check for time extension
            time_left = (auction.end_time - datetime.utcnow()).total_seconds()
            if time_left < settings.AUCTION_EXTENSION_TRIGGER:
                auction.end_time = datetime.utcnow() + timedelta(
                    seconds=settings.AUCTION_EXTENSION_TIME
                )
                auction.extended_times += 1

            # Log bid history
            history = BidHistory(
                auction_id=auction_id,
                bid_id=bid.id,
                event_type="placed",
                details=f"Bid of {actual_amount} placed"
            )
            db.add(history)

            await db.flush()
            await db.refresh(bid)

            # Update Redis cache
            await BidService._update_redis_cache(auction, bid)

            return {
                "success": True,
                "bid": bid,
                "auction": auction,
                "time_extended": time_left < settings.AUCTION_EXTENSION_TRIGGER
            }

    @staticmethod
    async def _calculate_minimum_bid(auction: Auction) -> Decimal:
        """Calculate minimum bid amount"""
        if auction.current_price == auction.starting_price:
            return auction.starting_price

        # Use configured increment or percentage
        if settings.MIN_BID_INCREMENT_PERCENT:
            increment = auction.current_price * (
                settings.MIN_BID_INCREMENT_PERCENT / Decimal("100")
            )
            increment = max(increment, settings.MIN_BID_INCREMENT)
        else:
            increment = auction.increment or settings.MIN_BID_INCREMENT

        return auction.current_price + increment

    @staticmethod
    async def _get_highest_bid(auction_id: int, db: AsyncSession) -> Optional[Bid]:
        """Get current highest bid"""
        result = await db.execute(
            select(Bid)
            .where(
                and_(
                    Bid.auction_id == auction_id,
                    Bid.is_winning == True
                )
            )
            .order_by(Bid.placed_at.desc())
            .limit(1)
        )
        return result.scalar_one_or_none()

    @staticmethod
    async def _calculate_proxy_bid(
        bid_amount: Decimal,
        max_bid: Optional[Decimal],
        current_high: Bid,
        increment: Decimal
    ) -> Decimal:
        """Calculate actual bid amount with proxy bidding"""
        if not max_bid:
            return bid_amount

        if not current_high.max_bid_amount:
            # Current high doesn't have proxy, use our bid
            return min(bid_amount, current_high.actual_amount + increment)

        # Both have proxy bids
        if max_bid > current_high.max_bid_amount:
            # We win, bid just above them
            return min(max_bid, current_high.max_bid_amount + increment)
        else:
            # They win, we bid our max
            return max_bid

    @staticmethod
    async def _update_redis_cache(auction: Auction, bid: Bid):
        """Update Redis cache with latest bid"""
        import json

        # Update auction cache
        auction_key = f"auction:{auction.id}"
        auction_data = {
            "id": auction.id,
            "current_price": str(auction.current_price),
            "bid_count": auction.bid_count,
            "end_time": auction.end_time.isoformat()
        }
        await redis_client.redis.setex(
            auction_key,
            300,  # 5 minutes
            json.dumps(auction_data)
        )

        # Add bid to sorted set
        bids_key = f"auction:{auction.id}:bids"
        await redis_client.redis.zadd(
            bids_key,
            {json.dumps({"bid_id": bid.id, "amount": str(bid.actual_amount)}): bid.placed_at.timestamp()}
        )

    @staticmethod
    async def get_bid_history(
        auction_id: int,
        db: AsyncSession,
        limit: int = 50
    ) -> list:
        """Get bid history for auction"""
        result = await db.execute(
            select(Bid)
            .where(Bid.auction_id == auction_id)
            .order_by(Bid.placed_at.desc())
            .limit(limit)
        )
        return result.scalars().all()

    @staticmethod
    async def retract_bid(
        bid_id: int,
        bidder_id: int,
        db: AsyncSession
    ) -> dict:
        """Retract a bid (with restrictions)"""
        result = await db.execute(
            select(Bid).where(Bid.id == bid_id)
        )
        bid = result.scalar_one_or_none()

        if not bid:
            return {"success": False, "error": "Bid not found"}

        if bid.bidder_id != bidder_id:
            return {"success": False, "error": "Not your bid"}

        if bid.is_retracted:
            return {"success": False, "error": "Bid already retracted"}

        # Check time limit (e.g., can only retract within 5 minutes)
        if (datetime.utcnow() - bid.placed_at).total_seconds() > 300:
            return {"success": False, "error": "Cannot retract bid after 5 minutes"}

        bid.is_retracted = True
        bid.retracted_at = datetime.utcnow()
        bid.is_winning = False

        await db.commit()

        return {"success": True}
```

### Step 6: WebSocket Connection Manager (app/websocket/connection_manager.py)
```python
from fastapi import WebSocket
from typing import Dict, Set
from collections import defaultdict
import json

class AuctionConnectionManager:
    """Manages WebSocket connections for auctions"""

    def __init__(self):
        # {auction_id: {user_id: WebSocket}}
        self.auction_connections: Dict[int, Dict[int, WebSocket]] = defaultdict(dict)

        # Global live auctions feed
        self.live_feed_connections: Dict[int, WebSocket] = {}

    async def connect_to_auction(
        self,
        auction_id: int,
        user_id: int,
        websocket: WebSocket
    ):
        """Connect user to auction"""
        await websocket.accept()
        self.auction_connections[auction_id][user_id] = websocket

    async def connect_to_live_feed(self, user_id: int, websocket: WebSocket):
        """Connect to live auctions feed"""
        await websocket.accept()
        self.live_feed_connections[user_id] = websocket

    def disconnect_from_auction(self, auction_id: int, user_id: int):
        """Disconnect from auction"""
        if auction_id in self.auction_connections:
            self.auction_connections[auction_id].pop(user_id, None)
            if not self.auction_connections[auction_id]:
                del self.auction_connections[auction_id]

    def disconnect_from_live_feed(self, user_id: int):
        """Disconnect from live feed"""
        self.live_feed_connections.pop(user_id, None)

    async def broadcast_to_auction(self, auction_id: int, message: dict):
        """Broadcast message to all watching auction"""
        connections = self.auction_connections.get(auction_id, {})

        disconnected = []
        for user_id, websocket in connections.items():
            try:
                await websocket.send_json(message)
            except:
                disconnected.append(user_id)

        # Clean up disconnected
        for user_id in disconnected:
            self.disconnect_from_auction(auction_id, user_id)

    async def broadcast_to_live_feed(self, message: dict):
        """Broadcast to live feed watchers"""
        disconnected = []
        for user_id, websocket in self.live_feed_connections.items():
            try:
                await websocket.send_json(message)
            except:
                disconnected.append(user_id)

        # Clean up
        for user_id in disconnected:
            self.disconnect_from_live_feed(user_id)

    def get_viewer_count(self, auction_id: int) -> int:
        """Get number of viewers for auction"""
        return len(self.auction_connections.get(auction_id, {}))

# Global manager
auction_manager = AuctionConnectionManager()
```

### Step 7: WebSocket Handler (app/websocket/auction_handler.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import json
from decimal import Decimal

from app.websocket.connection_manager import auction_manager
from app.services.bid_service import BidService
from app.models.auction import Auction
from app.models.user import User
from app.core.security import decode_access_token
from app.db.session import get_db

async def handle_auction_stream(
    websocket: WebSocket,
    auction_id: int,
    token: str = Query(...),
    db: AsyncSession = Depends(get_db)
):
    """Handle auction WebSocket stream"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    user_id = int(payload.get("sub"))

    # Get auction
    result = await db.execute(select(Auction).where(Auction.id == auction_id))
    auction = result.scalar_one_or_none()

    if not auction:
        await websocket.close(code=1008, reason="Auction not found")
        return

    # Connect
    await auction_manager.connect_to_auction(auction_id, user_id, websocket)

    # Send initial auction state
    await websocket.send_json({
        "type": "auction_state",
        "auction": {
            "id": auction.id,
            "title": auction.title,
            "current_price": str(auction.current_price),
            "bid_count": auction.bid_count,
            "end_time": auction.end_time.isoformat(),
            "status": auction.status.value
        },
        "viewer_count": auction_manager.get_viewer_count(auction_id)
    })

    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "place_bid":
                # Place bid
                bid_amount = Decimal(str(message.get("bid_amount")))
                max_bid = message.get("max_bid_amount")
                max_bid_amount = Decimal(str(max_bid)) if max_bid else None

                result = await BidService.place_bid(
                    auction_id,
                    user_id,
                    bid_amount,
                    max_bid_amount,
                    db
                )

                if result["success"]:
                    # Broadcast new bid to all watchers
                    await auction_manager.broadcast_to_auction(
                        auction_id,
                        {
                            "type": "new_bid",
                            "bid": {
                                "id": result["bid"].id,
                                "amount": str(result["bid"].actual_amount),
                                "bidder_id": result["bid"].bidder_id,
                                "is_proxy": result["bid"].is_proxy
                            },
                            "auction": {
                                "current_price": str(result["auction"].current_price),
                                "bid_count": result["auction"].bid_count,
                                "end_time": result["auction"].end_time.isoformat()
                            },
                            "time_extended": result.get("time_extended", False)
                        }
                    )

                    # Send success to bidder
                    await websocket.send_json({
                        "type": "bid_placed",
                        "success": True,
                        "bid_id": result["bid"].id
                    })
                else:
                    # Send error to bidder
                    await websocket.send_json({
                        "type": "bid_error",
                        "error": result["error"]
                    })

            elif msg_type == "get_bid_history":
                # Get bid history
                bids = await BidService.get_bid_history(auction_id, db)

                await websocket.send_json({
                    "type": "bid_history",
                    "bids": [
                        {
                            "id": bid.id,
                            "amount": str(bid.actual_amount),
                            "placed_at": bid.placed_at.isoformat()
                        }
                        for bid in bids
                    ]
                })

    except WebSocketDisconnect:
        auction_manager.disconnect_from_auction(auction_id, user_id)
    except Exception as e:
        print(f"Error in auction WebSocket: {e}")
        auction_manager.disconnect_from_auction(auction_id, user_id)
```

### Step 8: Run Application
```bash
# Start Redis
redis-server

# Start Celery worker
celery -A app.core.celery worker --loglevel=info

# Start Celery beat (for scheduled tasks)
celery -A app.core.celery beat --loglevel=info

# Start application
uvicorn app.main:app --reload --port 8000
```

## Expected Outputs

### 1. New Bid
```json
{
  "type": "new_bid",
  "bid": {
    "id": 123,
    "amount": "150.00",
    "bidder_id": 456,
    "is_proxy": true
  },
  "auction": {
    "current_price": "150.00",
    "bid_count": 15,
    "end_time": "2024-01-15T18:00:00"
  },
  "time_extended": true
}
```

### 2. Auction State
```json
{
  "type": "auction_state",
  "auction": {
    "id": 1,
    "title": "Vintage Watch",
    "current_price": "145.00",
    "bid_count": 14,
    "end_time": "2024-01-15T17:58:00",
    "status": "live"
  },
  "viewer_count": 42
}
```

## Bonus Challenges

- [ ] Add auction categories
- [ ] Implement watchlist/favorites
- [ ] Create auction templates
- [ ] Add image galleries
- [ ] Implement seller ratings
- [ ] Create buyer verification
- [ ] Add shipping calculator
- [ ] Implement escrow service
- [ ] Create auto-bidding strategies
- [ ] Add analytics dashboard
- [ ] Implement fraud detection
- [ ] Create mobile apps
- [ ] Add social sharing
- [ ] Implement bid groups
- [ ] Create auction alerts

## Resources

- [Auction Theory](https://en.wikipedia.org/wiki/Auction_theory)
- [PostgreSQL ACID](https://www.postgresql.org/docs/current/tutorial-transactions.html)
- [Redis Sorted Sets](https://redis.io/topics/data-types-intro#sorted-sets)
- [Race Conditions](https://www.postgresql.org/docs/current/applevel-consistency.html)
- [Payment Integration](https://stripe.com/docs)

## Success Criteria

- [ ] Bids process without race conditions
- [ ] Real-time updates work smoothly
- [ ] Proxy bidding functions correctly
- [ ] Time extensions work properly
- [ ] No duplicate bids accepted
- [ ] Reserve prices enforced
- [ ] Auction timers accurate
- [ ] High-frequency bidding supported
- [ ] Handles 1000+ concurrent bidders
- [ ] Transaction integrity maintained
- [ ] Bid history accurate
- [ ] Winner determined correctly
- [ ] Payment processing works
- [ ] Fraud detection effective
