# Project 7: Build Authenticated API with JWT Tokens

## Overview
Create a secure ML prediction API with JWT (JSON Web Token) authentication, user management, role-based access control (RBAC), and API key support. This is essential for production APIs that need to control access and track usage per user.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Implement JWT authentication in FastAPI
- Create user registration and login system
- Handle password hashing securely (bcrypt)
- Implement role-based access control (RBAC)
- Generate and validate API keys
- Add OAuth2 password flow
- Implement token refresh mechanism
- Track API usage per user

## Technical Stack
- **Framework**: FastAPI with OAuth2
- **Authentication**: JWT (python-jose)
- **Password Hashing**: passlib with bcrypt
- **Database**: SQLite or PostgreSQL
- **ORM**: SQLAlchemy
- **Testing**: pytest with authenticated requests

## Project Requirements

### 1. Authentication Methods
- JWT token-based authentication
- API key authentication
- OAuth2 password flow
- Token refresh mechanism
- Password reset flow

### 2. API Endpoints

**Auth Endpoints:**
- `POST /auth/register` - User registration
- `POST /auth/login` - User login (get JWT)
- `POST /auth/refresh` - Refresh access token
- `POST /auth/logout` - Logout (invalidate token)
- `POST /auth/reset-password` - Request password reset

**User Endpoints:**
- `GET /users/me` - Get current user profile
- `PUT /users/me` - Update user profile
- `GET /users/me/usage` - Get API usage statistics

**Protected ML Endpoints:**
- `POST /predict` - Make prediction (authenticated)
- `GET /predictions/history` - Get user's prediction history
- `GET /models` - List available models

### 3. User Roles
- **Admin**: Full access, manage users
- **Premium**: Higher rate limits, all models
- **Basic**: Limited rate limits, basic models
- **Guest**: Read-only access

### 4. Security Features
- Password hashing with bcrypt
- JWT with expiration
- Secure token storage
- CORS configuration
- Input sanitization
- SQL injection prevention

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn python-jose[cryptography]
pip install passlib[bcrypt] python-multipart
pip install sqlalchemy databases aiosqlite
pip install torch torchvision pillow
pip install pytest httpx
```

### Step 2: Project Structure
```bash
mkdir jwt_ml_api
cd jwt_ml_api

touch main.py auth.py models.py database.py
touch security.py config.py schemas.py
touch test_auth.py
```

### Step 3: Configuration (config.py)
```python
from pydantic import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    # API Settings
    API_TITLE = "Authenticated ML API"
    API_VERSION = "1.0.0"

    # Security Settings
    SECRET_KEY: str = "your-secret-key-change-in-production"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    # Database
    DATABASE_URL: str = "sqlite:///./ml_api.db"

    # Rate Limits (requests per hour)
    RATE_LIMIT_GUEST: int = 10
    RATE_LIMIT_BASIC: int = 100
    RATE_LIMIT_PREMIUM: int = 1000
    RATE_LIMIT_ADMIN: int = -1  # Unlimited

    class Config:
        env_file = ".env"

settings = Settings()
```

### Step 4: Database Models (models.py)
```python
from sqlalchemy import Column, String, Integer, Boolean, DateTime, Enum, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

Base = declarative_base()

class UserRole(enum.Enum):
    GUEST = "guest"
    BASIC = "basic"
    PREMIUM = "premium"
    ADMIN = "admin"

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    username = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String)
    role = Column(Enum(UserRole), default=UserRole.BASIC)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    last_login = Column(DateTime, nullable=True)

    # Relationships
    api_keys = relationship("APIKey", back_populates="user")
    predictions = relationship("UserPrediction", back_populates="user")

class APIKey(Base):
    __tablename__ = "api_keys"

    id = Column(Integer, primary_key=True, index=True)
    key = Column(String, unique=True, index=True, nullable=False)
    name = Column(String)
    user_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)
    expires_at = Column(DateTime, nullable=True)
    is_active = Column(Boolean, default=True)
    last_used = Column(DateTime, nullable=True)

    user = relationship("User", back_populates="api_keys")

class UserPrediction(Base):
    __tablename__ = "user_predictions"

    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    prediction_result = Column(String)  # JSON string
    model_used = Column(String)
    processing_time = Column(Integer)  # milliseconds
    created_at = Column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="predictions")
```

### Step 5: Pydantic Schemas (schemas.py)
```python
from pydantic import BaseModel, EmailStr, Field, validator
from typing import Optional, List
from datetime import datetime
from enum import Enum

class UserRole(str, Enum):
    GUEST = "guest"
    BASIC = "basic"
    PREMIUM = "premium"
    ADMIN = "admin"

class UserCreate(BaseModel):
    """User registration schema"""
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)
    password: str = Field(..., min_length=8)
    full_name: Optional[str] = None

    @validator('password')
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v

class UserLogin(BaseModel):
    """Login schema"""
    username: str
    password: str

class UserResponse(BaseModel):
    """User response schema"""
    id: int
    email: str
    username: str
    full_name: Optional[str]
    role: UserRole
    is_active: bool
    created_at: datetime

    class Config:
        orm_mode = True

class Token(BaseModel):
    """JWT token response"""
    access_token: str
    refresh_token: Optional[str] = None
    token_type: str = "bearer"
    expires_in: int  # seconds

class TokenData(BaseModel):
    """Token payload data"""
    username: Optional[str] = None
    role: Optional[str] = None

class UserUsage(BaseModel):
    """User API usage statistics"""
    user_id: int
    total_predictions: int
    predictions_today: int
    predictions_this_month: int
    rate_limit: int
    remaining_requests: int

class PredictionRequest(BaseModel):
    """Prediction request"""
    image: str  # base64
    top_k: int = Field(default=5, ge=1, le=10)

class PredictionResponse(BaseModel):
    """Prediction response"""
    predictions: List[dict]
    top_prediction: str
    confidence: float
    processing_time: float
```

### Step 6: Security Functions (security.py)
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, APIKeyHeader
from sqlalchemy.orm import Session

from config import settings
from schemas import TokenData

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 scheme
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")

# API Key header
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)

def create_access_token(
    data: dict,
    expires_delta: Optional[timedelta] = None
) -> str:
    """Create JWT access token"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )

    to_encode.update({"exp": expire, "type": "access"})
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

def create_refresh_token(data: dict) -> str:
    """Create JWT refresh token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

def verify_token(token: str) -> TokenData:
    """Verify and decode JWT token"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        username: str = payload.get("sub")
        role: str = payload.get("role")

        if username is None:
            raise credentials_exception

        return TokenData(username=username, role=role)

    except JWTError:
        raise credentials_exception

def generate_api_key() -> str:
    """Generate random API key"""
    import secrets
    return f"sk_{secrets.token_urlsafe(32)}"
```

### Step 7: Database Setup (database.py)
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from typing import Generator

from config import settings
from models import Base

# Create engine
engine = create_engine(
    settings.DATABASE_URL,
    connect_args={"check_same_thread": False}  # SQLite only
)

# Session factory
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def init_db():
    """Initialize database"""
    Base.metadata.create_all(bind=engine)

def get_db() -> Generator[Session, None, None]:
    """Get database session"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Step 8: Authentication Module (auth.py)
```python
from sqlalchemy.orm import Session
from fastapi import Depends, HTTPException, status
from datetime import datetime

from database import get_db
from models import User, APIKey
from schemas import UserCreate, UserLogin, TokenData
from security import (
    verify_password, get_password_hash, verify_token,
    oauth2_scheme, api_key_header
)

def get_user_by_username(db: Session, username: str):
    """Get user by username"""
    return db.query(User).filter(User.username == username).first()

def get_user_by_email(db: Session, email: str):
    """Get user by email"""
    return db.query(User).filter(User.email == email).first()

def create_user(db: Session, user: UserCreate):
    """Create new user"""
    # Check if user exists
    if get_user_by_username(db, user.username):
        raise HTTPException(
            status_code=400,
            detail="Username already registered"
        )

    if get_user_by_email(db, user.email):
        raise HTTPException(
            status_code=400,
            detail="Email already registered"
        )

    # Create user
    db_user = User(
        email=user.email,
        username=user.username,
        hashed_password=get_password_hash(user.password),
        full_name=user.full_name
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def authenticate_user(db: Session, username: str, password: str):
    """Authenticate user"""
    user = get_user_by_username(db, username)

    if not user:
        return False

    if not verify_password(password, user.hashed_password):
        return False

    # Update last login
    user.last_login = datetime.utcnow()
    db.commit()

    return user

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """Get current authenticated user from JWT"""
    token_data = verify_token(token)

    user = get_user_by_username(db, token_data.username)

    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found"
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Inactive user"
        )

    return user

async def get_current_user_from_api_key(
    api_key: str = Depends(api_key_header),
    db: Session = Depends(get_db)
) -> User:
    """Get user from API key"""
    if not api_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key required"
        )

    db_api_key = db.query(APIKey).filter(APIKey.key == api_key).first()

    if not db_api_key or not db_api_key.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key"
        )

    # Check expiration
    if db_api_key.expires_at and db_api_key.expires_at < datetime.utcnow():
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key expired"
        )

    # Update last used
    db_api_key.last_used = datetime.utcnow()
    db.commit()

    return db_api_key.user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """Ensure user is active"""
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

def require_role(allowed_roles: list):
    """Decorator to require specific roles"""
    async def role_checker(current_user: User = Depends(get_current_user)):
        if current_user.role.value not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions"
            )
        return current_user
    return role_checker
```

### Step 9: Main Application (main.py)
```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from datetime import timedelta, datetime
from typing import List

from database import get_db, init_db
from models import User, UserPrediction
from schemas import (
    UserCreate, UserResponse, Token, UserUsage,
    PredictionRequest, PredictionResponse
)
from auth import (
    create_user, authenticate_user, get_current_user,
    get_current_active_user, require_role
)
from security import create_access_token, create_refresh_token
from config import settings

app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION
)

@app.on_event("startup")
async def startup_event():
    """Initialize database on startup"""
    init_db()
    print("Database initialized")

# Authentication Endpoints

@app.post("/auth/register", response_model=UserResponse)
async def register(user: UserCreate, db: Session = Depends(get_db)):
    """Register new user"""
    db_user = create_user(db, user)
    return db_user

@app.post("/auth/login", response_model=Token)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """
    Login and get JWT tokens

    OAuth2 compatible token endpoint
    """
    user = authenticate_user(db, form_data.username, form_data.password)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Create tokens
    access_token_expires = timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username, "role": user.role.value},
        expires_delta=access_token_expires
    )

    refresh_token = create_refresh_token(
        data={"sub": user.username, "role": user.role.value}
    )

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "expires_in": settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60
    }

# User Endpoints

@app.get("/users/me", response_model=UserResponse)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    """Get current user profile"""
    return current_user

@app.get("/users/me/usage", response_model=UserUsage)
async def get_user_usage(
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db)
):
    """Get user API usage statistics"""
    # Count predictions
    total = db.query(UserPrediction).filter(
        UserPrediction.user_id == current_user.id
    ).count()

    # Today's predictions
    today = datetime.utcnow().replace(hour=0, minute=0, second=0, microsecond=0)
    today_count = db.query(UserPrediction).filter(
        UserPrediction.user_id == current_user.id,
        UserPrediction.created_at >= today
    ).count()

    # Get rate limit based on role
    rate_limits = {
        "guest": settings.RATE_LIMIT_GUEST,
        "basic": settings.RATE_LIMIT_BASIC,
        "premium": settings.RATE_LIMIT_PREMIUM,
        "admin": settings.RATE_LIMIT_ADMIN
    }

    rate_limit = rate_limits.get(current_user.role.value, 100)
    remaining = max(0, rate_limit - today_count) if rate_limit > 0 else -1

    return UserUsage(
        user_id=current_user.id,
        total_predictions=total,
        predictions_today=today_count,
        predictions_this_month=0,  # Implement this
        rate_limit=rate_limit,
        remaining_requests=remaining
    )

# Protected ML Endpoints

@app.post("/predict", response_model=PredictionResponse)
async def predict(
    request: PredictionRequest,
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db)
):
    """
    Make ML prediction (authenticated)

    Requires valid JWT token or API key
    """
    # Check rate limit
    usage = await get_user_usage(current_user, db)
    if usage.rate_limit > 0 and usage.remaining_requests <= 0:
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Rate limit exceeded"
        )

    # Dummy prediction (replace with actual model)
    import time
    start = time.time()

    result = {
        'predictions': [
            {'golden_retriever': 0.87},
            {'labrador': 0.09},
            {'poodle': 0.02}
        ],
        'top_prediction': 'golden_retriever',
        'confidence': 0.87,
        'processing_time': time.time() - start
    }

    # Record prediction
    prediction_record = UserPrediction(
        user_id=current_user.id,
        prediction_result=str(result),
        model_used="resnet50",
        processing_time=int(result['processing_time'] * 1000)
    )
    db.add(prediction_record)
    db.commit()

    return PredictionResponse(**result)

@app.get("/predictions/history")
async def get_prediction_history(
    limit: int = 10,
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db)
):
    """Get user's prediction history"""
    predictions = db.query(UserPrediction).filter(
        UserPrediction.user_id == current_user.id
    ).order_by(UserPrediction.created_at.desc()).limit(limit).all()

    return {"predictions": predictions, "total": len(predictions)}

# Admin Endpoints

@app.get("/admin/users", dependencies=[Depends(require_role(["admin"]))])
async def list_all_users(db: Session = Depends(get_db)):
    """List all users (admin only)"""
    users = db.query(User).all()
    return {"users": users, "total": len(users)}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 10: Test Authentication
```bash
# Register user
curl -X POST "http://localhost:8000/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "username": "testuser",
    "password": "SecurePass123",
    "full_name": "Test User"
  }'

# Login
curl -X POST "http://localhost:8000/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=testuser&password=SecurePass123"

# Use token
curl -X GET "http://localhost:8000/users/me" \
  -H "Authorization: Bearer <your-access-token>"

# Make prediction
curl -X POST "http://localhost:8000/predict" \
  -H "Authorization: Bearer <your-access-token>" \
  -H "Content-Type: application/json" \
  -d '{"image": "<base64>", "top_k": 5}'
```

## Expected Outputs

### 1. Registration Response
```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "testuser",
  "full_name": "Test User",
  "role": "basic",
  "is_active": true,
  "created_at": "2025-11-14T10:00:00"
}
```

### 2. Login Response
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

### 3. User Usage Response
```json
{
  "user_id": 1,
  "total_predictions": 47,
  "predictions_today": 5,
  "predictions_this_month": 102,
  "rate_limit": 100,
  "remaining_requests": 95
}
```

## Bonus Challenges

- [ ] Add email verification on registration
- [ ] Implement password reset via email
- [ ] Add two-factor authentication (2FA)
- [ ] Create user invitation system
- [ ] Add OAuth2 social login (Google, GitHub)
- [ ] Implement token blacklist for logout
- [ ] Add session management
- [ ] Create usage analytics dashboard
- [ ] Implement IP-based rate limiting
- [ ] Add webhook notifications for events
- [ ] Create API key management UI
- [ ] Add audit logging for security events

## Resources

- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [JWT Introduction](https://jwt.io/introduction)
- [OAuth2 with Password Flow](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [Password Hashing Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [python-jose Documentation](https://python-jose.readthedocs.io/)

## Success Criteria

- [ ] User registration works correctly
- [ ] Login returns valid JWT tokens
- [ ] Protected endpoints require authentication
- [ ] Invalid tokens are rejected
- [ ] Passwords are hashed securely
- [ ] Role-based access control works
- [ ] Rate limiting is enforced
- [ ] API keys can be used as alternative auth
- [ ] Token expiration is handled properly
- [ ] All tests pass
