# Project 09: Real-Time Collaborative ML App with WebSockets

## Overview
Build a real-time collaborative ML application where multiple users can interact with ML models simultaneously, share predictions, and see live updates using WebSockets. This project demonstrates building scalable real-time applications with Socket.IO, collaborative features, presence awareness, and synchronized ML inference results across multiple clients.

## Learning Objectives
- Master WebSocket communication with Socket.IO
- Implement real-time collaborative features
- Handle concurrent ML predictions
- Build presence and awareness systems
- Manage shared state across clients
- Implement operational transformation for collaboration
- Handle reconnection and network failures
- Create scalable real-time architecture

## Difficulty Level
**Advanced** - Requires understanding of WebSockets, real-time systems, concurrent programming, and distributed state management.

## Technical Stack
- **Frontend**: React or Vue 3
- **Real-time**: Socket.IO Client
- **Backend**: Node.js with Socket.IO Server
- **ML Backend**: Python FastAPI with Socket.IO support
- **State Management**: Redux or Zustand with middleware
- **Database**: Redis for pub/sub and caching
- **Queue**: Bull for job queue management
- **Authentication**: JWT tokens
- **Testing**: Jest, Socket.IO Test Utils

## Requirements

### Real-Time Features
1. Live prediction broadcasting
2. User presence indicators
3. Shared workspace/session
4. Live cursor tracking
5. Collaborative model parameter tuning
6. Real-time chat for discussions
7. Notification system
8. Live activity feed

### Collaboration Features
1. Multiple users in same session
2. Shared prediction history
3. Collaborative annotations
4. Role-based permissions (viewer, editor, admin)
5. Live updates without refresh
6. Conflict resolution
7. Undo/redo functionality
8. Export shared sessions

### WebSocket Events
1. Connection/disconnection
2. Prediction requests
3. Prediction results
4. User join/leave
5. Cursor movements
6. Parameter changes
7. Chat messages
8. Notifications

### Performance Requirements
1. Handle 100+ concurrent users per session
2. Message latency < 50ms
3. Reconnection handling
4. Message queuing for offline users
5. Efficient broadcasting
6. Memory-efficient state management

## Step-by-Step Implementation

### Step 1: Backend Setup (Node.js + Socket.IO)

```bash
# Create backend directory
mkdir ml-collaborative-backend
cd ml-collaborative-backend

npm init -y

# Install dependencies
npm install express socket.io
npm install redis ioredis
npm install bull
npm install jsonwebtoken
npm install cors dotenv
npm install axios
npm install winston # Logging
npm install -D @types/node @types/express typescript ts-node
```

Create `src/server.ts`:

```typescript
import express from 'express';
import { createServer } from 'http';
import { Server, Socket } from 'socket.io';
import Redis from 'ioredis';
import jwt from 'jsonwebtoken';
import cors from 'cors';

const app = express();
const httpServer = createServer(app);

// Redis clients
const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
});

const redisPub = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
});

const redisSub = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
});

// Socket.IO setup with Redis adapter
const io = new Server(httpServer, {
  cors: {
    origin: process.env.CLIENT_URL || 'http://localhost:3000',
    credentials: true,
  },
  transports: ['websocket', 'polling'],
});

// Middleware
app.use(cors());
app.use(express.json());

// Authentication middleware for Socket.IO
io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error('Authentication error'));
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET || 'secret');
    socket.data.user = decoded;
    next();
  } catch (err) {
    next(new Error('Authentication error'));
  }
});

// Session management
interface Session {
  id: string;
  name: string;
  users: Map<string, UserInfo>;
  predictions: any[];
  createdAt: number;
}

interface UserInfo {
  id: string;
  name: string;
  color: string;
  role: 'admin' | 'editor' | 'viewer';
  cursor?: { x: number; y: number };
}

const sessions = new Map<string, Session>();

// Socket.IO event handlers
io.on('connection', (socket: Socket) => {
  console.log('Client connected:', socket.id);

  const userId = socket.data.user.id;
  const userName = socket.data.user.name;

  // Join session
  socket.on('join-session', async ({ sessionId, role }) => {
    try {
      await socket.join(sessionId);

      // Create or get session
      if (!sessions.has(sessionId)) {
        sessions.set(sessionId, {
          id: sessionId,
          name: `Session ${sessionId}`,
          users: new Map(),
          predictions: [],
          createdAt: Date.now(),
        });
      }

      const session = sessions.get(sessionId)!;

      // Add user to session
      const userColor = `#${Math.floor(Math.random() * 16777215).toString(16)}`;
      const userInfo: UserInfo = {
        id: userId,
        name: userName,
        color: userColor,
        role: role || 'viewer',
      };

      session.users.set(socket.id, userInfo);

      // Notify other users
      socket.to(sessionId).emit('user-joined', {
        user: userInfo,
        timestamp: Date.now(),
      });

      // Send session state to new user
      socket.emit('session-state', {
        session: {
          id: session.id,
          name: session.name,
          users: Array.from(session.users.values()),
          predictions: session.predictions,
        },
      });

      // Broadcast updated user list
      io.to(sessionId).emit('users-updated', {
        users: Array.from(session.users.values()),
      });

      console.log(`User ${userName} joined session ${sessionId}`);
    } catch (error) {
      console.error('Error joining session:', error);
      socket.emit('error', { message: 'Failed to join session' });
    }
  });

  // Leave session
  socket.on('leave-session', async ({ sessionId }) => {
    const session = sessions.get(sessionId);
    if (!session) return;

    const user = session.users.get(socket.id);
    session.users.delete(socket.id);

    await socket.leave(sessionId);

    // Notify others
    socket.to(sessionId).emit('user-left', {
      user,
      timestamp: Date.now(),
    });

    // Broadcast updated user list
    io.to(sessionId).emit('users-updated', {
      users: Array.from(session.users.values()),
    });

    console.log(`User ${user?.name} left session ${sessionId}`);
  });

  // Run prediction
  socket.on('predict', async ({ sessionId, features, modelId }) => {
    try {
      const session = sessions.get(sessionId);
      if (!session) {
        socket.emit('error', { message: 'Session not found' });
        return;
      }

      const user = session.users.get(socket.id);
      if (!user) {
        socket.emit('error', { message: 'User not in session' });
        return;
      }

      // Send prediction to ML service (mock for now)
      const predictionId = `pred_${Date.now()}`;

      // Emit processing status
      io.to(sessionId).emit('prediction-processing', {
        predictionId,
        user: user.name,
        timestamp: Date.now(),
      });

      // Simulate ML prediction (replace with actual ML API call)
      setTimeout(() => {
        const result = {
          id: predictionId,
          userId: user.id,
          userName: user.name,
          features,
          prediction: Math.random() > 0.5 ? 'approved' : 'denied',
          confidence: 0.7 + Math.random() * 0.3,
          timestamp: Date.now(),
          modelId,
        };

        // Save to session
        session.predictions.push(result);

        // Save to Redis for persistence
        redis.lpush(`session:${sessionId}:predictions`, JSON.stringify(result));
        redis.ltrim(`session:${sessionId}:predictions`, 0, 99); // Keep last 100

        // Broadcast result to all users in session
        io.to(sessionId).emit('prediction-result', result);
      }, 1000 + Math.random() * 2000);
    } catch (error) {
      console.error('Prediction error:', error);
      socket.emit('error', { message: 'Prediction failed' });
    }
  });

  // Cursor movement
  socket.on('cursor-move', ({ sessionId, x, y }) => {
    const session = sessions.get(sessionId);
    if (!session) return;

    const user = session.users.get(socket.id);
    if (!user) return;

    user.cursor = { x, y };

    // Broadcast to others (not to sender)
    socket.to(sessionId).emit('cursor-update', {
      userId: user.id,
      userName: user.name,
      color: user.color,
      x,
      y,
    });
  });

  // Chat message
  socket.on('chat-message', ({ sessionId, message }) => {
    const session = sessions.get(sessionId);
    if (!session) return;

    const user = session.users.get(socket.id);
    if (!user) return;

    const chatMessage = {
      id: `msg_${Date.now()}`,
      userId: user.id,
      userName: user.name,
      userColor: user.color,
      message,
      timestamp: Date.now(),
    };

    // Broadcast to all users in session
    io.to(sessionId).emit('chat-message', chatMessage);

    // Save to Redis
    redis.lpush(`session:${sessionId}:chat`, JSON.stringify(chatMessage));
    redis.ltrim(`session:${sessionId}:chat`, 0, 99);
  });

  // Parameter change
  socket.on('parameter-change', ({ sessionId, parameter, value }) => {
    const session = sessions.get(sessionId);
    if (!session) return;

    const user = session.users.get(socket.id);
    if (!user || user.role === 'viewer') return; // Only editors and admins can change parameters

    // Broadcast to all users
    socket.to(sessionId).emit('parameter-changed', {
      parameter,
      value,
      changedBy: user.name,
      timestamp: Date.now(),
    });
  });

  // Disconnection
  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);

    // Remove from all sessions
    sessions.forEach((session, sessionId) => {
      if (session.users.has(socket.id)) {
        const user = session.users.get(socket.id);
        session.users.delete(socket.id);

        // Notify others
        io.to(sessionId).emit('user-left', {
          user,
          timestamp: Date.now(),
        });

        // Broadcast updated user list
        io.to(sessionId).emit('users-updated', {
          users: Array.from(session.users.values()),
        });
      }
    });
  });
});

// REST API endpoints
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.post('/api/sessions', (req, res) => {
  const sessionId = `session_${Date.now()}`;
  res.json({ sessionId });
});

// Start server
const PORT = process.env.PORT || 3001;
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Step 2: Frontend Setup (React + Socket.IO Client)

```bash
# Create React app
npx create-react-app ml-collaborative-frontend --template typescript
cd ml-collaborative-frontend

# Install dependencies
npm install socket.io-client
npm install zustand
npm install react-router-dom
npm install axios
npm install @headlessui/react # For UI components
npm install tailwindcss
```

Create `src/services/socket.ts`:

```typescript
import { io, Socket } from 'socket.io-client';

const SOCKET_URL = process.env.REACT_APP_SOCKET_URL || 'http://localhost:3001';

class SocketService {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(token: string): Socket {
    if (this.socket?.connected) {
      return this.socket;
    }

    this.socket = io(SOCKET_URL, {
      auth: { token },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      reconnectionAttempts: this.maxReconnectAttempts,
    });

    this.setupEventHandlers();

    return this.socket;
  }

  private setupEventHandlers() {
    if (!this.socket) return;

    this.socket.on('connect', () => {
      console.log('Connected to server');
      this.reconnectAttempts = 0;
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Disconnected:', reason);
    });

    this.socket.on('connect_error', (error) => {
      console.error('Connection error:', error);
      this.reconnectAttempts++;

      if (this.reconnectAttempts >= this.maxReconnectAttempts) {
        console.error('Max reconnection attempts reached');
      }
    });

    this.socket.on('error', (error) => {
      console.error('Socket error:', error);
    });
  }

  joinSession(sessionId: string, role: string = 'editor') {
    this.socket?.emit('join-session', { sessionId, role });
  }

  leaveSession(sessionId: string) {
    this.socket?.emit('leave-session', { sessionId });
  }

  predict(sessionId: string, features: any, modelId: string) {
    this.socket?.emit('predict', { sessionId, features, modelId });
  }

  sendCursorPosition(sessionId: string, x: number, y: number) {
    this.socket?.emit('cursor-move', { sessionId, x, y });
  }

  sendChatMessage(sessionId: string, message: string) {
    this.socket?.emit('chat-message', { sessionId, message });
  }

  changeParameter(sessionId: string, parameter: string, value: any) {
    this.socket?.emit('parameter-change', { sessionId, parameter, value });
  }

  on(event: string, handler: (...args: any[]) => void) {
    this.socket?.on(event, handler);
  }

  off(event: string, handler?: (...args: any[]) => void) {
    this.socket?.off(event, handler);
  }

  disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }

  getSocket(): Socket | null {
    return this.socket;
  }

  isConnected(): boolean {
    return this.socket?.connected || false;
  }
}

export const socketService = new SocketService();
```

### Step 3: Collaborative Store

Create `src/store/collaborationStore.ts`:

```typescript
import { create } from 'zustand';

export interface User {
  id: string;
  name: string;
  color: string;
  role: 'admin' | 'editor' | 'viewer';
  cursor?: { x: number; y: number };
}

export interface Prediction {
  id: string;
  userId: string;
  userName: string;
  features: any;
  prediction: string;
  confidence: number;
  timestamp: number;
  modelId: string;
}

export interface ChatMessage {
  id: string;
  userId: string;
  userName: string;
  userColor: string;
  message: string;
  timestamp: number;
}

interface CollaborationState {
  // Session
  sessionId: string | null;
  sessionName: string;
  isConnected: boolean;

  // Users
  users: User[];
  currentUser: User | null;

  // Predictions
  predictions: Prediction[];
  isProcessing: boolean;

  // Chat
  chatMessages: ChatMessage[];
  unreadCount: number;

  // Cursors
  cursors: Map<string, { x: number; y: number; userName: string; color: string }>;

  // Actions
  setSessionId: (id: string) => void;
  setSessionName: (name: string) => void;
  setConnected: (connected: boolean) => void;
  setCurrentUser: (user: User) => void;
  setUsers: (users: User[]) => void;
  addUser: (user: User) => void;
  removeUser: (userId: string) => void;
  addPrediction: (prediction: Prediction) => void;
  setPredictions: (predictions: Prediction[]) => void;
  setProcessing: (processing: boolean) => void;
  addChatMessage: (message: ChatMessage) => void;
  setChatMessages: (messages: ChatMessage[]) => void;
  incrementUnread: () => void;
  resetUnread: () => void;
  updateCursor: (userId: string, x: number, y: number, userName: string, color: string) => void;
  removeCursor: (userId: string) => void;
  reset: () => void;
}

export const useCollaborationStore = create<CollaborationState>((set) => ({
  // Initial state
  sessionId: null,
  sessionName: '',
  isConnected: false,
  users: [],
  currentUser: null,
  predictions: [],
  isProcessing: false,
  chatMessages: [],
  unreadCount: 0,
  cursors: new Map(),

  // Actions
  setSessionId: (id) => set({ sessionId: id }),

  setSessionName: (name) => set({ sessionName: name }),

  setConnected: (connected) => set({ isConnected: connected }),

  setCurrentUser: (user) => set({ currentUser: user }),

  setUsers: (users) => set({ users }),

  addUser: (user) =>
    set((state) => ({
      users: [...state.users, user],
    })),

  removeUser: (userId) =>
    set((state) => ({
      users: state.users.filter((u) => u.id !== userId),
    })),

  addPrediction: (prediction) =>
    set((state) => ({
      predictions: [prediction, ...state.predictions],
    })),

  setPredictions: (predictions) => set({ predictions }),

  setProcessing: (processing) => set({ isProcessing: processing }),

  addChatMessage: (message) =>
    set((state) => ({
      chatMessages: [...state.chatMessages, message],
    })),

  setChatMessages: (messages) => set({ chatMessages: messages }),

  incrementUnread: () =>
    set((state) => ({ unreadCount: state.unreadCount + 1 })),

  resetUnread: () => set({ unreadCount: 0 }),

  updateCursor: (userId, x, y, userName, color) =>
    set((state) => {
      const newCursors = new Map(state.cursors);
      newCursors.set(userId, { x, y, userName, color });
      return { cursors: newCursors };
    }),

  removeCursor: (userId) =>
    set((state) => {
      const newCursors = new Map(state.cursors);
      newCursors.delete(userId);
      return { cursors: newCursors };
    }),

  reset: () =>
    set({
      sessionId: null,
      sessionName: '',
      isConnected: false,
      users: [],
      currentUser: null,
      predictions: [],
      isProcessing: false,
      chatMessages: [],
      unreadCount: 0,
      cursors: new Map(),
    }),
}));
```

### Step 4: Collaborative Session Component

Create `src/components/CollaborativeSession.tsx`:

```typescript
import React, { useEffect, useRef } from 'react';
import { socketService } from '../services/socket';
import { useCollaborationStore } from '../store/collaborationStore';
import { UserList } from './UserList';
import { PredictionFeed } from './PredictionFeed';
import { ChatPanel } from './ChatPanel';
import { CursorOverlay } from './CursorOverlay';

interface Props {
  sessionId: string;
  userName: string;
  role: 'admin' | 'editor' | 'viewer';
}

export function CollaborativeSession({ sessionId, userName, role }: Props) {
  const containerRef = useRef<HTMLDivElement>(null);

  const {
    setConnected,
    setCurrentUser,
    setUsers,
    addUser,
    removeUser,
    addPrediction,
    setPredictions,
    setProcessing,
    addChatMessage,
    updateCursor,
    removeCursor,
  } = useCollaborationStore();

  useEffect(() => {
    // Connect to socket
    const token = localStorage.getItem('authToken') || 'demo-token';
    socketService.connect(token);

    // Join session
    socketService.joinSession(sessionId, role);

    // Setup event listeners
    socketService.on('session-state', ({ session }) => {
      setUsers(session.users);
      setPredictions(session.predictions);
    });

    socketService.on('user-joined', ({ user }) => {
      addUser(user);
    });

    socketService.on('user-left', ({ user }) => {
      removeUser(user.id);
      removeCursor(user.id);
    });

    socketService.on('users-updated', ({ users }) => {
      setUsers(users);
    });

    socketService.on('prediction-processing', () => {
      setProcessing(true);
    });

    socketService.on('prediction-result', (result) => {
      addPrediction(result);
      setProcessing(false);
    });

    socketService.on('cursor-update', ({ userId, userName, color, x, y }) => {
      updateCursor(userId, x, y, userName, color);
    });

    socketService.on('chat-message', (message) => {
      addChatMessage(message);
    });

    setConnected(true);

    // Track cursor movements
    const handleMouseMove = (e: MouseEvent) => {
      if (containerRef.current) {
        const rect = containerRef.current.getBoundingClientRect();
        const x = ((e.clientX - rect.left) / rect.width) * 100;
        const y = ((e.clientY - rect.top) / rect.height) * 100;

        socketService.sendCursorPosition(sessionId, x, y);
      }
    };

    window.addEventListener('mousemove', handleMouseMove);

    // Cleanup
    return () => {
      window.removeEventListener('mousemove', handleMouseMove);
      socketService.leaveSession(sessionId);
      socketService.disconnect();
      setConnected(false);
    };
  }, [sessionId, role]);

  return (
    <div ref={containerRef} className="relative h-screen flex">
      {/* Cursor overlay */}
      <CursorOverlay />

      {/* Main content */}
      <div className="flex-1 flex flex-col">
        {/* Header */}
        <div className="bg-white border-b p-4 flex justify-between items-center">
          <h1 className="text-xl font-bold">Session: {sessionId}</h1>
          <UserList />
        </div>

        {/* Prediction feed */}
        <div className="flex-1 overflow-y-auto p-4">
          <PredictionFeed />
        </div>
      </div>

      {/* Chat sidebar */}
      <ChatPanel sessionId={sessionId} />
    </div>
  );
}
```

### Step 5: Cursor Overlay Component

Create `src/components/CursorOverlay.tsx`:

```typescript
import React from 'react';
import { useCollaborationStore } from '../store/collaborationStore';

export function CursorOverlay() {
  const { cursors } = useCollaborationStore();

  return (
    <div className="absolute inset-0 pointer-events-none z-50">
      {Array.from(cursors.entries()).map(([userId, cursor]) => (
        <div
          key={userId}
          className="absolute transition-all duration-100"
          style={{
            left: `${cursor.x}%`,
            top: `${cursor.y}%`,
            transform: 'translate(-50%, -50%)',
          }}
        >
          <svg
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill={cursor.color}
          >
            <path d="M3 3l7.07 16.97 2.51-7.39 7.39-2.51L3 3z" />
          </svg>
          <div
            className="mt-1 px-2 py-1 rounded text-white text-xs whitespace-nowrap"
            style={{ backgroundColor: cursor.color }}
          >
            {cursor.userName}
          </div>
        </div>
      ))}
    </div>
  );
}
```

## Expected Outputs

1. **Real-Time Collaboration**:
   - Multiple users in same session
   - Live prediction updates
   - User presence indicators
   - Live cursor tracking

2. **Communication**:
   - Real-time chat
   - Activity notifications
   - User join/leave alerts

3. **ML Integration**:
   - Shared predictions
   - Collaborative parameter tuning
   - Live model performance

4. **Performance**:
   - Low latency (<50ms)
   - Smooth updates
   - Handle 100+ users
   - Efficient broadcasting

## Bonus Challenges

1. **Operational Transformation**: Resolve conflicts in real-time
2. **Audio/Video**: WebRTC for voice/video chat
3. **Screen Sharing**: Share ML dashboards
4. **Annotations**: Collaborative image annotations
5. **Version Control**: Track session history
6. **Permissions**: Fine-grained access control
7. **Analytics**: Track collaboration metrics
8. **Mobile Support**: React Native client

## Resources

- [Socket.IO Documentation](https://socket.io/docs/)
- [Redis Pub/Sub](https://redis.io/topics/pubsub)
- [WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [Real-time Collaboration](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)

## Success Criteria

### Functionality (40%)
- [ ] WebSocket connection working
- [ ] Real-time updates functional
- [ ] User presence tracking
- [ ] Collaborative features
- [ ] Chat system working

### Performance (30%)
- [ ] Low latency messaging
- [ ] Handle concurrent users
- [ ] Efficient broadcasting
- [ ] Reconnection handling
- [ ] Memory management

### Code Quality (20%)
- [ ] Clean architecture
- [ ] Error handling
- [ ] TypeScript typing
- [ ] State management
- [ ] Testing coverage

### User Experience (10%)
- [ ] Smooth real-time updates
- [ ] Clear presence indicators
- [ ] Intuitive collaboration
- [ ] Mobile responsive
- [ ] Accessible design
