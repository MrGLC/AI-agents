# Project 9: Build Live Stock Ticker with WebSocket Streaming

## Overview
Create a real-time stock market data streaming service that delivers live price updates, market data, and financial indicators via WebSockets. This project demonstrates building high-frequency data streams, handling market data feeds, and creating financial dashboards similar to Bloomberg or Yahoo Finance.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Stream high-frequency market data
- Handle multiple simultaneous data feeds
- Implement data throttling and batching
- Build real-time price calculations
- Create market watchlists
- Handle after-hours trading
- Implement price alerts
- Build candlestick chart data
- Handle data validation and anomalies

## Technical Stack
- **Framework**: FastAPI with WebSockets
- **Data Sources**: Alpha Vantage, IEX Cloud, or Yahoo Finance API
- **Database**: PostgreSQL (historical data) + TimescaleDB
- **Cache**: Redis (real-time quotes, sorted sets)
- **Charts**: TradingView, Chart.js, or D3.js
- **Queue**: Celery (data fetching)
- **Testing**: pytest-asyncio, market simulation

## Project Requirements

### 1. Core Features
- Real-time stock price streaming
- Multiple symbol tracking
- Historical price data
- Price change calculations
- Volume tracking
- Market cap and fundamentals
- Price alerts and notifications
- Watchlist management
- Portfolio tracking

### 2. Data Types
- Real-time quotes (price, volume)
- OHLC data (candlesticks)
- Market indices (S&P 500, NASDAQ)
- News and events
- Technical indicators (MA, RSI, MACD)
- Order book data (bid/ask)

### 3. WebSocket Endpoints
- `WS /ws/ticker/{symbol}` - Single stock stream
- `WS /ws/market/live` - Multi-stock stream
- `WS /ws/watchlist/{id}` - Watchlist stream
- `GET /api/stocks/{symbol}` - Stock details
- `GET /api/stocks/{symbol}/history` - Historical data
- `POST /api/watchlist` - Create watchlist

### 4. Features
- Subscribe/unsubscribe to symbols
- Real-time price updates
- Historical chart data
- Price alerts
- Market hours detection
- After-hours quotes

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create project
mkdir stock-ticker-streaming
cd stock-ticker-streaming

# Virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg aiosqlite
pip install redis aioredis celery
pip install pandas numpy
pip install httpx  # For API calls
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
pandas==2.1.3
numpy==1.26.0
httpx==0.25.2
pydantic==2.5.0
python-jose[cryptography]==3.3.0
pytest==7.4.3
pytest-asyncio==0.21.1
python-dotenv==1.0.0
EOF
```

### Step 2: Project Structure
```bash
mkdir -p app/{api,core,models,schemas,services,websocket,tasks,data}
touch app/__init__.py app/main.py app/config.py

# Models
touch app/models/{__init__,stock,watchlist,alert}.py

# Schemas
touch app/schemas/{__init__,stock,websocket,market}.py

# Services
touch app/services/{__init__,market_data,stock_service,alert_service}.py

# WebSocket
touch app/websocket/{__init__,connection_manager,ticker_handler}.py

# Tasks
touch app/tasks/{__init__,data_fetcher,price_updater}.py

# Data providers
touch app/data/{__init__,alpha_vantage,yahoo_finance,mock_provider}.py

# Core
touch app/core/{__init__,redis,security,celery}.py

# Database
mkdir app/db
touch app/db/{__init__,session,base}.py

# Frontend
mkdir static
touch static/{ticker.html,dashboard.html}

# Tests
mkdir tests
touch tests/{test_streaming,test_data,test_alerts}.py
```

### Step 3: Configuration (app/config.py)
```python
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    APP_NAME: str = "Stock Ticker Streaming"
    APP_VERSION: str = "1.0.0"

    # Database
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@localhost/stocks"
    # For SQLite: "sqlite+aiosqlite:///./stocks.db"

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Celery
    CELERY_BROKER_URL: str = "redis://localhost:6379/1"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/2"

    # Market Data APIs
    ALPHA_VANTAGE_API_KEY: Optional[str] = None
    IEX_CLOUD_API_KEY: Optional[str] = None
    FINNHUB_API_KEY: Optional[str] = None

    # Data Provider
    DATA_PROVIDER: str = "mock"  # mock, alpha_vantage, iex, yahoo

    # Market Hours (EST)
    MARKET_OPEN_HOUR: int = 9
    MARKET_OPEN_MINUTE: int = 30
    MARKET_CLOSE_HOUR: int = 16
    MARKET_CLOSE_MINUTE: int = 0

    # Streaming Settings
    UPDATE_INTERVAL: float = 1.0  # 1 second
    BATCH_INTERVAL: float = 0.1  # 100ms batching
    MAX_SYMBOLS_PER_CONNECTION: int = 50
    PRICE_CACHE_TTL: int = 60  # seconds

    # Historical Data
    MAX_HISTORY_DAYS: int = 365
    DEFAULT_HISTORY_DAYS: int = 30

    # Alerts
    ALERT_CHECK_INTERVAL: int = 5  # seconds
    MAX_ALERTS_PER_USER: int = 100

    # WebSocket
    PING_INTERVAL: int = 30
    MAX_CONNECTIONS: int = 10000

    # JWT
    SECRET_KEY: str = "change-this-secret-key"
    ALGORITHM: str = "HS256"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models

**app/models/stock.py**:
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Index
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class Stock(Base):
    __tablename__ = "stocks"

    id = Column(Integer, primary_key=True, index=True)
    symbol = Column(String(10), unique=True, index=True, nullable=False)
    name = Column(String(255), nullable=False)
    exchange = Column(String(50))
    sector = Column(String(100))
    industry = Column(String(100))
    market_cap = Column(Float)
    is_active = Column(Boolean, default=True)
    updated_at = Column(DateTime, default=datetime.utcnow)

class StockPrice(Base):
    __tablename__ = "stock_prices"
    __table_args__ = (
        Index('idx_symbol_timestamp', 'symbol', 'timestamp'),
    )

    id = Column(Integer, primary_key=True, index=True)
    symbol = Column(String(10), nullable=False, index=True)
    timestamp = Column(DateTime, nullable=False, index=True)

    # OHLCV data
    open = Column(Float, nullable=False)
    high = Column(Float, nullable=False)
    low = Column(Float, nullable=False)
    close = Column(Float, nullable=False)
    volume = Column(Float, nullable=False)

    # Additional
    adjusted_close = Column(Float)
    dividend = Column(Float)
    split_coefficient = Column(Float)

class StockQuote(Base):
    """Latest real-time quote"""
    __tablename__ = "stock_quotes"

    id = Column(Integer, primary_key=True, index=True)
    symbol = Column(String(10), unique=True, index=True)
    price = Column(Float, nullable=False)
    change = Column(Float, nullable=False)
    change_percent = Column(Float, nullable=False)
    volume = Column(Float)
    bid = Column(Float)
    ask = Column(Float)
    bid_size = Column(Integer)
    ask_size = Column(Integer)
    day_high = Column(Float)
    day_low = Column(Float)
    day_open = Column(Float)
    previous_close = Column(Float)
    timestamp = Column(DateTime, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

**app/models/watchlist.py**:
```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey, Table
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

watchlist_stocks = Table(
    'watchlist_stocks',
    Base.metadata,
    Column('watchlist_id', Integer, ForeignKey('watchlists.id')),
    Column('stock_symbol', String(10), ForeignKey('stocks.symbol'))
)

class Watchlist(Base):
    __tablename__ = "watchlists"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"))
    is_public = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="watchlists")
    stocks = relationship("Stock", secondary=watchlist_stocks)
```

**app/models/alert.py**:
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from datetime import datetime
from app.db.base import Base

class PriceAlert(Base):
    __tablename__ = "price_alerts"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    symbol = Column(String(10), nullable=False, index=True)
    condition = Column(String(20), nullable=False)  # above, below, change_percent
    target_price = Column(Float, nullable=False)
    is_active = Column(Boolean, default=True)
    triggered_at = Column(DateTime, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User")
```

### Step 5: Market Data Service (app/services/market_data.py)
```python
from typing import Dict, List, Optional
from datetime import datetime, timedelta
import asyncio
import json
import random

from app.core.redis import redis_client
from app.config import settings

class MarketDataService:
    """Service for fetching and streaming market data"""

    def __init__(self):
        self.subscriptions: Dict[str, int] = {}  # {symbol: subscriber_count}
        self.price_cache: Dict[str, Dict] = {}

    async def subscribe(self, symbol: str):
        """Subscribe to symbol updates"""
        self.subscriptions[symbol] = self.subscriptions.get(symbol, 0) + 1

        # Start fetching if first subscriber
        if self.subscriptions[symbol] == 1:
            asyncio.create_task(self._fetch_and_stream(symbol))

    async def unsubscribe(self, symbol: str):
        """Unsubscribe from symbol"""
        if symbol in self.subscriptions:
            self.subscriptions[symbol] -= 1

            if self.subscriptions[symbol] <= 0:
                del self.subscriptions[symbol]

    async def _fetch_and_stream(self, symbol: str):
        """Fetch and stream data for symbol"""
        while symbol in self.subscriptions:
            try:
                # Fetch latest quote
                quote = await self._fetch_quote(symbol)

                if quote:
                    # Cache in Redis
                    await self._cache_quote(symbol, quote)

                    # Update in-memory cache
                    self.price_cache[symbol] = quote

                await asyncio.sleep(settings.UPDATE_INTERVAL)

            except Exception as e:
                print(f"Error fetching {symbol}: {e}")
                await asyncio.sleep(5)

    async def _fetch_quote(self, symbol: str) -> Optional[Dict]:
        """Fetch quote from data provider"""
        # Use configured provider
        if settings.DATA_PROVIDER == "mock":
            return await self._fetch_mock_quote(symbol)
        elif settings.DATA_PROVIDER == "alpha_vantage":
            return await self._fetch_alpha_vantage_quote(symbol)
        else:
            return await self._fetch_mock_quote(symbol)

    async def _fetch_mock_quote(self, symbol: str) -> Dict:
        """Generate mock quote data for testing"""
        # Get last price or start at 100
        last_price = self.price_cache.get(symbol, {}).get("price", 100.0)

        # Random walk
        change = random.uniform(-2, 2)
        new_price = max(1, last_price + change)

        change_amount = new_price - last_price
        change_percent = (change_amount / last_price) * 100 if last_price else 0

        return {
            "symbol": symbol,
            "price": round(new_price, 2),
            "change": round(change_amount, 2),
            "change_percent": round(change_percent, 2),
            "volume": random.randint(100000, 10000000),
            "bid": round(new_price - 0.01, 2),
            "ask": round(new_price + 0.01, 2),
            "day_high": round(new_price * 1.02, 2),
            "day_low": round(new_price * 0.98, 2),
            "timestamp": datetime.utcnow().isoformat()
        }

    async def _fetch_alpha_vantage_quote(self, symbol: str) -> Optional[Dict]:
        """Fetch from Alpha Vantage API"""
        # Implement Alpha Vantage API call
        # https://www.alphavantage.co/documentation/
        pass

    async def _cache_quote(self, symbol: str, quote: Dict):
        """Cache quote in Redis"""
        key = f"quote:{symbol}"
        await redis_client.redis.setex(
            key,
            settings.PRICE_CACHE_TTL,
            json.dumps(quote)
        )

        # Add to price time series
        ts_key = f"prices:{symbol}"
        await redis_client.redis.zadd(
            ts_key,
            {json.dumps(quote): datetime.utcnow().timestamp()}
        )

        # Keep only last hour
        cutoff = (datetime.utcnow() - timedelta(hours=1)).timestamp()
        await redis_client.redis.zremrangebyscore(ts_key, 0, cutoff)

    async def get_quote(self, symbol: str) -> Optional[Dict]:
        """Get latest quote"""
        # Try cache first
        key = f"quote:{symbol}"
        data = await redis_client.redis.get(key)

        if data:
            return json.loads(data)

        # Fetch fresh
        return await self._fetch_quote(symbol)

    async def get_price_history(
        self,
        symbol: str,
        start_time: datetime,
        end_time: datetime
    ) -> List[Dict]:
        """Get historical prices"""
        ts_key = f"prices:{symbol}"

        # Get from Redis
        data = await redis_client.redis.zrangebyscore(
            ts_key,
            start_time.timestamp(),
            end_time.timestamp()
        )

        history = []
        for item in data:
            quote = json.loads(item)
            history.append(quote)

        return history

    def is_market_open(self) -> bool:
        """Check if market is currently open"""
        now = datetime.now()

        # Check if weekend
        if now.weekday() >= 5:  # Saturday = 5, Sunday = 6
            return False

        # Check if market hours
        market_open = now.replace(
            hour=settings.MARKET_OPEN_HOUR,
            minute=settings.MARKET_OPEN_MINUTE,
            second=0
        )
        market_close = now.replace(
            hour=settings.MARKET_CLOSE_HOUR,
            minute=settings.MARKET_CLOSE_MINUTE,
            second=0
        )

        return market_open <= now <= market_close

# Global service
market_data_service = MarketDataService()
```

### Step 6: WebSocket Handler (app/websocket/ticker_handler.py)
```python
from fastapi import WebSocket, WebSocketDisconnect, Depends, Query
import json
import asyncio
from typing import Set

from app.websocket.connection_manager import ticker_manager
from app.services.market_data import market_data_service
from app.core.security import decode_access_token

async def handle_ticker_stream(
    websocket: WebSocket,
    token: str = Query(...)
):
    """Handle stock ticker WebSocket stream"""
    # Authenticate
    payload = decode_access_token(token)
    if not payload:
        await websocket.close(code=1008, reason="Invalid token")
        return

    user_id = int(payload.get("sub"))

    await websocket.accept()

    subscribed_symbols: Set[str] = set()

    # Send market status
    await websocket.send_json({
        "type": "market_status",
        "is_open": market_data_service.is_market_open()
    })

    try:
        # Start price streaming task
        stream_task = asyncio.create_task(
            stream_prices(websocket, subscribed_symbols)
        )

        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            msg_type = message.get("type")

            if msg_type == "subscribe":
                # Subscribe to symbols
                symbols = message.get("symbols", [])

                for symbol in symbols:
                    symbol = symbol.upper()
                    subscribed_symbols.add(symbol)
                    await market_data_service.subscribe(symbol)

                await websocket.send_json({
                    "type": "subscribed",
                    "symbols": list(subscribed_symbols)
                })

            elif msg_type == "unsubscribe":
                # Unsubscribe from symbols
                symbols = message.get("symbols", [])

                for symbol in symbols:
                    symbol = symbol.upper()
                    if symbol in subscribed_symbols:
                        subscribed_symbols.discard(symbol)
                        await market_data_service.unsubscribe(symbol)

                await websocket.send_json({
                    "type": "unsubscribed",
                    "symbols": symbols
                })

            elif msg_type == "get_quote":
                # Get instant quote
                symbol = message.get("symbol", "").upper()
                quote = await market_data_service.get_quote(symbol)

                if quote:
                    await websocket.send_json({
                        "type": "quote",
                        "data": quote
                    })

    except WebSocketDisconnect:
        # Unsubscribe from all
        for symbol in subscribed_symbols:
            await market_data_service.unsubscribe(symbol)

        stream_task.cancel()

async def stream_prices(websocket: WebSocket, symbols: Set[str]):
    """Stream price updates"""
    from app.config import settings

    try:
        while True:
            if symbols:
                updates = []

                # Get latest quotes for all subscribed symbols
                for symbol in symbols:
                    quote = await market_data_service.get_quote(symbol)
                    if quote:
                        updates.append(quote)

                # Send batch update
                if updates:
                    await websocket.send_json({
                        "type": "price_update",
                        "data": updates
                    })

            await asyncio.sleep(settings.BATCH_INTERVAL)

    except asyncio.CancelledError:
        pass
```

### Step 7: Frontend (static/ticker.html)
```html
<!DOCTYPE html>
<html>
<head>
    <title>Stock Ticker</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #0e0e0e; color: #fff; }
        .header { background: #1a1a1a; padding: 20px; border-bottom: 1px solid #333; }
        .ticker-board { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 20px; padding: 20px; }
        .ticker-card { background: #1a1a1a; border-radius: 8px; padding: 20px; border: 1px solid #333; }
        .symbol { font-size: 24px; font-weight: bold; margin-bottom: 10px; }
        .price { font-size: 36px; font-weight: bold; margin: 10px 0; }
        .change { font-size: 18px; padding: 5px 10px; border-radius: 4px; display: inline-block; }
        .change.positive { background: #0f5132; color: #75b798; }
        .change.negative { background: #58151c; color: #ea868f; }
        .details { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 15px; font-size: 14px; color: #aaa; }
        .subscribe-box { padding: 20px; background: #1a1a1a; margin: 20px; border-radius: 8px; }
        input { padding: 10px; background: #2a2a2a; border: 1px solid #444; color: #fff; border-radius: 4px; margin-right: 10px; }
        button { padding: 10px 20px; background: #0d6efd; color: white; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background: #0b5ed7; }
        .status { padding: 10px; margin: 0 20px; border-radius: 4px; }
        .status.open { background: #0f5132; color: #75b798; }
        .status.closed { background: #58151c; color: #ea868f; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Live Stock Ticker</h1>
    </div>

    <div class="status" id="market-status"></div>

    <div class="subscribe-box">
        <input type="text" id="symbol-input" placeholder="Enter symbol (e.g., AAPL)" />
        <button onclick="subscribe()">Subscribe</button>
        <button onclick="unsubscribeAll()">Clear All</button>
    </div>

    <div class="ticker-board" id="ticker-board"></div>

    <script>
        let ws = null;
        const symbols = new Set();

        function connect() {
            const token = localStorage.getItem('auth_token') || 'demo-token';
            ws = new WebSocket(`ws://localhost:8000/ws/ticker?token=${token}`);

            ws.onopen = () => {
                console.log('Connected to ticker');
                // Resubscribe to symbols
                if (symbols.size > 0) {
                    ws.send(JSON.stringify({
                        type: 'subscribe',
                        symbols: Array.from(symbols)
                    }));
                }
            };

            ws.onmessage = (event) => {
                const data = JSON.parse(event.data);
                handleMessage(data);
            };

            ws.onerror = (error) => {
                console.error('WebSocket error:', error);
            };

            ws.onclose = () => {
                console.log('Disconnected. Reconnecting...');
                setTimeout(connect, 3000);
            };
        }

        function handleMessage(data) {
            switch(data.type) {
                case 'market_status':
                    updateMarketStatus(data.is_open);
                    break;
                case 'price_update':
                    data.data.forEach(quote => updateTicker(quote));
                    break;
                case 'quote':
                    updateTicker(data.data);
                    break;
            }
        }

        function updateMarketStatus(isOpen) {
            const status = document.getElementById('market-status');
            if (isOpen) {
                status.textContent = 'Market is OPEN';
                status.className = 'status open';
            } else {
                status.textContent = 'Market is CLOSED';
                status.className = 'status closed';
            }
        }

        function updateTicker(quote) {
            let card = document.getElementById(`ticker-${quote.symbol}`);

            if (!card) {
                card = createTickerCard(quote.symbol);
                document.getElementById('ticker-board').appendChild(card);
            }

            // Update price
            card.querySelector('.price').textContent = `$${quote.price.toFixed(2)}`;

            // Update change
            const changeEl = card.querySelector('.change');
            const changeText = `${quote.change >= 0 ? '+' : ''}${quote.change.toFixed(2)} (${quote.change_percent.toFixed(2)}%)`;
            changeEl.textContent = changeText;
            changeEl.className = `change ${quote.change >= 0 ? 'positive' : 'negative'}`;

            // Update details
            card.querySelector('.volume').textContent = `Volume: ${quote.volume.toLocaleString()}`;
            card.querySelector('.high').textContent = `High: $${quote.day_high.toFixed(2)}`;
            card.querySelector('.low').textContent = `Low: $${quote.day_low.toFixed(2)}`;
            card.querySelector('.bid-ask').textContent = `Bid/Ask: $${quote.bid}/$${quote.ask}`;
        }

        function createTickerCard(symbol) {
            const card = document.createElement('div');
            card.className = 'ticker-card';
            card.id = `ticker-${symbol}`;
            card.innerHTML = `
                <div class="symbol">${symbol}</div>
                <div class="price">--</div>
                <div class="change">--</div>
                <div class="details">
                    <div class="volume">Volume: --</div>
                    <div class="high">High: --</div>
                    <div class="low">Low: --</div>
                    <div class="bid-ask">Bid/Ask: --</div>
                </div>
            `;
            return card;
        }

        function subscribe() {
            const input = document.getElementById('symbol-input');
            const symbol = input.value.trim().toUpperCase();

            if (symbol && !symbols.has(symbol)) {
                symbols.add(symbol);

                if (ws && ws.readyState === WebSocket.OPEN) {
                    ws.send(JSON.stringify({
                        type: 'subscribe',
                        symbols: [symbol]
                    }));
                }

                input.value = '';
            }
        }

        function unsubscribeAll() {
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({
                    type: 'unsubscribe',
                    symbols: Array.from(symbols)
                }));
            }

            symbols.clear();
            document.getElementById('ticker-board').innerHTML = '';
        }

        // Initialize
        connect();

        // Auto-subscribe to popular symbols
        setTimeout(() => {
            ws.send(JSON.stringify({
                type: 'subscribe',
                symbols: ['AAPL', 'GOOGL', 'MSFT', 'AMZN', 'TSLA']
            }));
            ['AAPL', 'GOOGL', 'MSFT', 'AMZN', 'TSLA'].forEach(s => symbols.add(s));
        }, 1000);
    </script>
</body>
</html>
```

### Step 8: Run Application
```bash
# Start Redis
redis-server

# Start Celery worker
celery -A app.core.celery worker --loglevel=info

# Start application
uvicorn app.main:app --reload --port 8000

# Open ticker
# http://localhost:8000/static/ticker.html
```

## Expected Outputs

### 1. Price Update
```json
{
  "type": "price_update",
  "data": [
    {
      "symbol": "AAPL",
      "price": 178.45,
      "change": 2.35,
      "change_percent": 1.33,
      "volume": 54289100,
      "bid": 178.44,
      "ask": 178.46,
      "day_high": 179.20,
      "day_low": 176.80,
      "timestamp": "2024-01-15T14:30:25"
    }
  ]
}
```

### 2. Market Status
```json
{
  "type": "market_status",
  "is_open": true
}
```

## Bonus Challenges

- [ ] Add technical indicators (RSI, MACD, Bollinger Bands)
- [ ] Implement candlestick charts
- [ ] Create portfolio tracking
- [ ] Add options chain data
- [ ] Implement market screeners
- [ ] Create price prediction ML models
- [ ] Add news sentiment analysis
- [ ] Implement paper trading
- [ ] Create heatmaps
- [ ] Add crypto currency support
- [ ] Implement order book visualization
- [ ] Create mobile app
- [ ] Add social trading features
- [ ] Implement backtesting
- [ ] Create custom indicators

## Resources

- [Alpha Vantage API](https://www.alphavantage.co/documentation/)
- [IEX Cloud](https://iexcloud.io/docs/)
- [Yahoo Finance API](https://github.com/ranaroussi/yfinance)
- [TradingView Charts](https://www.tradingview.com/widget/)
- [Technical Analysis Library](https://github.com/bukosabino/ta)
- [WebSocket Market Data](https://www.youtube.com/watch?v=example)

## Success Criteria

- [ ] Real-time prices stream smoothly
- [ ] Multiple symbols update simultaneously
- [ ] Price updates < 1 second latency
- [ ] Market hours detected correctly
- [ ] Historical data queries work
- [ ] Price alerts trigger accurately
- [ ] Watchlists sync across devices
- [ ] Handles 100+ symbols per user
- [ ] Charts render without lag
- [ ] Data integrity maintained
- [ ] No price anomalies
- [ ] After-hours quotes work
- [ ] Volume data accurate
- [ ] System handles 1000+ concurrent users
