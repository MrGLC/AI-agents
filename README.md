# AI Agents Repository

A comprehensive collection of specialized AI agents and practice projects for building production-ready machine learning systems, full-stack applications, and personalized AI companions.

## 📋 Table of Contents

- [Overview](#overview)
- [Specialized Agents](#specialized-agents)
  - [ML Deployment & Infrastructure](#ml-deployment--infrastructure)
  - [Frontend Development](#frontend-development)
  - [Backend & Real-time Systems](#backend--real-time-systems)
  - [AI/LLM Systems](#aillm-systems)
  - [Terminal & CLI Tools](#terminal--cli-tools)
- [Practice Projects](#practice-projects)
- [Getting Started](#getting-started)
- [Repository Structure](#repository-structure)

## 🎯 Overview

This repository contains:
- **14 specialized AI agents** - Expert agents for different domains of software development and AI/ML
- **100+ practice projects** - Hands-on projects organized by technology and use case
- **Claude Code integration** - All agents are available as Claude Code agents in `.claude/agents/`

## 🤖 Specialized Agents

### ML Deployment & Infrastructure

#### 1. **ML Model Deployment Agent**
**File:** `.claude/agents/ml-model-deploy.md`

Expert in deploying machine learning models to production environments.

**Capabilities:**
- Containerizing ML models with Docker
- Building deployment pipelines (Flask, FastAPI, Django REST)
- Serverless deployment (AWS Lambda, Google Cloud Functions, Azure Functions)
- Model serving frameworks (TensorFlow Serving, TorchServe, ONNX Runtime)
- Model optimization (quantization, pruning, ONNX conversion)
- CI/CD pipelines for ML systems
- API versioning and A/B testing
- Production monitoring and logging

**Use when:** Deploying trained models to production, setting up ML infrastructure

---

#### 2. **Model API Builder Agent**
**File:** `.claude/agents/model-api-builder.md`

Specialist in building robust, scalable REST APIs around machine learning models.

**Capabilities:**
- FastAPI and Flask API development
- Request/response validation (Pydantic)
- Authentication and authorization (JWT, API keys)
- Rate limiting and throttling
- Caching strategies (Redis, in-memory)
- Batch processing and async inference
- API documentation (OpenAPI/Swagger)
- Comprehensive error handling and logging

**Use when:** Building production APIs for ML models, creating RESTful endpoints

---

#### 3. **TensorFlow.js Converter Agent**
**File:** `.claude/agents/tensorflowjs-converter.md`

Expert at converting ML models to TensorFlow.js for browser and Node.js deployment.

**Capabilities:**
- Converting models from TensorFlow, Keras, PyTorch to TensorFlow.js
- Model optimization (quantization, pruning)
- WebGL and WASM backend optimization
- Model size reduction techniques
- Browser compatibility testing
- In-browser inference implementation

**Use when:** Deploying models to browsers, building client-side ML applications

---

### Frontend Development

#### 4. **React Frontend Developer Agent**
**File:** `.claude/agents/react-frontend-developer.md`

Expert React developer with deep knowledge of modern React patterns and ecosystem.

**Capabilities:**
- React 18+ with hooks and concurrent features
- TypeScript with React
- State management (Context, Redux, Zustand, Jotai)
- React Router for navigation
- Form handling (React Hook Form, Formik)
- Data fetching (React Query, SWR, RTK Query)
- Styling solutions (CSS Modules, Styled Components, Tailwind)
- Component libraries (MUI, Ant Design, shadcn/ui)
- Performance optimization
- Testing (Jest, React Testing Library)

**Use when:** Building React applications, creating modern web UIs

---

#### 5. **ML Frontend Integrator Agent**
**File:** `.claude/agents/ml-frontend-integrator.md`

Specialist in integrating machine learning models with modern frontend frameworks.

**Capabilities:**
- React, Vue, Angular, and Svelte integration
- API integration for ML backends
- Real-time updates (WebSockets, SSE)
- TensorFlow.js in-browser inference
- File uploads and preprocessing
- Progressive Web Apps for offline ML
- Performance optimization for ML apps
- Visualization of ML results

**Use when:** Integrating ML models with frontend, building ML-powered UIs

---

### Backend & Real-time Systems

#### 6. **Backend WebSocket Developer Agent**
**File:** `.claude/agents/backend-websocket-developer.md`

Expert in real-time communication systems and WebSocket implementations.

**Capabilities:**
- WebSocket protocols (RFC 6455)
- FastAPI WebSocket support
- Socket.IO for advanced features
- Async/await patterns in Python
- Real-time messaging architectures
- Pub/Sub patterns (Redis, RabbitMQ)
- Connection management and scaling
- Authentication and security
- Horizontal scaling with Redis

**Use when:** Building real-time features, chat systems, live dashboards, collaborative tools

---

### AI/LLM Systems

#### 7. **LangChain Agent Builder**
**File:** `.claude/agents/langchain-agent-builder.md`

Expert in building AI agents and agentic systems using LangChain and LangGraph.

**Capabilities:**
- LangChain framework (chains, agents, memory)
- LangGraph for complex agent workflows
- Agent architectures (ReAct, Plan-and-Execute, Reflexion)
- Tool creation and integration
- Memory systems (conversation, entity, summary)
- Vector stores and RAG patterns
- Multi-agent systems and orchestration
- Prompt engineering and optimization
- LangSmith debugging and monitoring

**Use when:** Building AI agents, implementing RAG systems, creating autonomous workflows

---

#### 8. **Conversational AI Designer Agent**
**File:** `.claude/agents/conversational-ai-designer.md` (39KB)

Specialist in designing and implementing sophisticated conversational AI systems.

**Capabilities:**
- Multi-turn conversation management
- Context and state tracking
- Intent recognition and dialogue flow
- Natural conversation patterns
- Error recovery and fallbacks
- Personality and tone consistency
- Conversation testing and evaluation

**Use when:** Building chatbots, voice assistants, conversational interfaces

---

#### 9. **LLM Personalization Specialist Agent**
**File:** `.claude/agents/llm-personalization-specialist.md` (37KB)

Expert in creating personalized AI experiences that adapt to individual users.

**Capabilities:**
- User preference learning and adaptation
- Personal communication style matching
- Context-aware personalization
- Learning from user feedback
- Privacy-preserving personalization
- A/B testing personalization strategies
- Multi-modal personalization

**Use when:** Building personalized AI companions, adaptive learning systems

---

#### 10. **Memory & Context Manager Agent**
**File:** `.claude/agents/memory-context-manager.md` (37KB)

Specialist in managing long-term memory and context for AI systems.

**Capabilities:**
- Long-term memory architecture
- Conversation history management
- Entity and fact extraction
- Memory retrieval and ranking
- Context window optimization
- Memory consolidation strategies
- Privacy and data retention
- Vector database integration

**Use when:** Implementing memory for AI agents, managing long-term context

---

#### 11. **User Profiling & Analytics Agent**
**File:** `.claude/agents/user-profiling-analytics.md` (40KB)

Expert in building user profiling systems and behavioral analytics for AI.

**Capabilities:**
- User behavior tracking and analysis
- Interest and preference modeling
- Engagement metrics and analytics
- Segmentation and clustering
- Predictive analytics for user behavior
- Privacy-compliant tracking
- Real-time profile updates
- Recommendation system integration

**Use when:** Building user profiles, implementing analytics, creating recommendations

---

#### 12. **Personal Growth Coach Agent**
**File:** `.claude/agents/personal-growth-coach.md` (42KB)

AI agent specialized in personal development, goal setting, and growth coaching.

**Capabilities:**
- Goal setting and tracking
- Habit formation strategies
- Progress monitoring and feedback
- Motivational support
- Personalized learning paths
- Reflection and self-assessment
- Accountability systems
- Growth mindset cultivation

**Use when:** Building personal development apps, coaching systems, learning platforms

---

### Terminal & CLI Tools

#### 13. **Rich Python Developer Agent**
**File:** `.claude/agents/rich-python-developer.md`

Expert in creating beautiful, feature-rich terminal applications using Rich library.

**Capabilities:**
- Rich library API and components
- Terminal text formatting and styling
- Tables, trees, and layouts
- Progress bars and spinners
- Live displays and dynamic updates
- Console logging and debugging
- Markdown and syntax highlighting
- Panels, boxes, and borders
- Color systems and themes

**Use when:** Building CLI tools, terminal UIs, command-line applications

---

#### 14. **Data Visualization Agent**
**File:** `.claude/agents/data-visualization.md`

Expert in creating interactive, beautiful visualizations for ML data and results.

**Capabilities:**
- Python visualization (Matplotlib, Seaborn, Plotly, Bokeh)
- JavaScript libraries (D3.js, Chart.js, Plotly.js, Recharts)
- Dashboard frameworks (Streamlit, Dash, Gradio)
- ML-specific visualizations (confusion matrices, ROC curves, feature importance)
- Real-time data streaming
- Responsive design
- Accessibility in data visualization

**Use when:** Creating data visualizations, building dashboards, presenting ML results

---

## 📚 Practice Projects

The repository includes 100+ hands-on practice projects organized by domain:

### 1. **ML Deployment Projects** (10 projects)
**Location:** `projects/ml-deployment/`

Practice projects for deploying machine learning models to production:
- Flask + scikit-learn deployment
- PyTorch + FastAPI + Docker
- AWS Lambda serverless deployment
- TensorFlow Serving
- Google Cloud Run deployment
- XGBoost model versioning
- Ensemble model load balancing
- A/B testing deployment
- Redis caching for real-time inference
- Batch prediction with Celery

---

### 2. **LangChain Agent Projects** (10 projects)
**Location:** `projects/langchain-agents/`

Building various types of AI agents using LangChain and LangGraph.

---

### 3. **React Frontend Projects** (10 projects)
**Location:** `projects/react-frontend/`

Modern React applications with various patterns and integrations.

---

### 4. **Backend WebSocket Projects** (10 projects)
**Location:** `projects/backend-websockets/`

Real-time communication systems using WebSockets.

---

### 5. **Rich Python CLI Projects** (10 projects)
**Location:** `projects/rich-python/`

Beautiful terminal applications including:
- File explorer TUI
- System monitor dashboard
- Package manager TUI
- Git CLI client
- Task/TODO manager
- Log viewer
- Database query tool
- Music player interface
- Code snippet manager
- Server deployment tool

---

### 6. **ML API Building Projects** (10 projects)
**Location:** `projects/ml-api-building/`

Building production-ready APIs for machine learning models.

---

### 7. **ML Frontend Integration Projects** (10 projects)
**Location:** `projects/ml-frontend-integration/`

Integrating ML models with modern frontend frameworks.

---

### 8. **TensorFlow.js Conversion Projects** (10 projects)
**Location:** `projects/tensorflowjs-conversion/`

Converting and deploying models to browsers with TensorFlow.js.

---

### 9. **Data Visualization Practice** (10 projects)
**Location:** `projects/data-visualization-practice/`

Creating various types of data visualizations and dashboards.

---

### 10. **Room Visualization Projects** (11 projects)
**Location:** `projects/room-visualization/`

Interior design and 3D room visualization projects.

---

### 11. **Companion LLM System** (11 projects)
**Location:** `projects/companion-llm-system/`

Complete system for building a personalized AI companion:
- System architecture
- RAG-based personalized chatbot
- Fine-tuning small LLMs for personal style
- Emotion tracking and empathy system
- Goal tracking and accountability
- And 6 more comprehensive projects

This is a complete production-ready system with architecture documentation included!

---

## 🚀 Getting Started

### Using the Agents

All agents are available in Claude Code. To use an agent:

1. Navigate to `.claude/agents/` to see all available agents
2. Each agent has a dedicated markdown file with full instructions
3. Agents are context-aware and can guide you through complex tasks

### Working with Projects

Each project folder contains:
- Detailed project descriptions
- Technical requirements
- Step-by-step implementation guides
- Learning objectives
- Difficulty levels

### Example Workflow

```bash
# 1. Choose a domain (e.g., ML deployment)
cd projects/ml-deployment

# 2. Pick a project (e.g., Flask deployment)
cat 01-flask-sklearn-deployment.md

# 3. Use the corresponding agent
# Open .claude/agents/ml-model-deploy.md in Claude Code

# 4. Build the project following the guide
```

---

## 📁 Repository Structure

```
AI-agents/
├── .claude/
│   ├── agents/              # 14 specialized AI agents
│   ├── commands/            # Custom commands
│   └── skills/              # Additional skills
├── projects/
│   ├── ml-deployment/       # 10 ML deployment projects
│   ├── langchain-agents/    # 10 LangChain projects
│   ├── react-frontend/      # 10 React projects
│   ├── backend-websockets/  # 10 WebSocket projects
│   ├── rich-python/         # 10 Rich CLI projects
│   ├── ml-api-building/     # 10 API building projects
│   ├── ml-frontend-integration/  # 10 integration projects
│   ├── tensorflowjs-conversion/  # 10 TensorFlow.js projects
│   ├── data-visualization-practice/  # 10 visualization projects
│   ├── room-visualization/  # 11 room design projects
│   └── companion-llm-system/     # 11 companion AI projects
└── README.md               # This file
```

---

## 🎓 Skill Levels

Projects are categorized by difficulty:

- **Beginner**: Basic concepts, straightforward implementation
- **Intermediate**: Multiple technologies, moderate complexity
- **Advanced**: Complex systems, production-ready implementations
- **Expert**: Cutting-edge techniques, research-level implementations

---

## 🛠️ Technology Stack

This repository covers:

**ML/AI:**
- TensorFlow, PyTorch, scikit-learn
- LangChain, LangGraph
- OpenAI, Anthropic APIs
- TensorFlow.js

**Backend:**
- FastAPI, Flask, Django
- WebSockets, Socket.IO
- Redis, PostgreSQL
- Docker, Kubernetes

**Frontend:**
- React, TypeScript
- React Query, Zustand
- TailwindCSS, Material-UI
- Recharts, Plotly.js

**DevOps:**
- Docker, Docker Compose
- AWS, Google Cloud, Azure
- CI/CD pipelines
- Monitoring and logging

**Tools:**
- Rich (Python terminal UI)
- Streamlit, Gradio
- pytest, Jest
- Git, GitHub Actions

---

## 💡 Use Cases

This repository is perfect for:

- **Learning ML deployment**: Go from training to production
- **Building AI agents**: Create autonomous AI systems with LangChain
- **Full-stack ML apps**: Integrate ML into web applications
- **Personal AI projects**: Build your own AI companion
- **CLI tool development**: Create beautiful terminal applications
- **Real-time systems**: Implement WebSocket-based features
- **Data visualization**: Present data and ML results effectively

---

## 🤝 Contributing

This is a personal learning repository, but suggestions and improvements are welcome!

---

## 📝 License

This repository is for educational purposes. Individual projects may use different libraries with their own licenses.

---

## 🌟 Highlights

**Most Comprehensive Projects:**
- **Companion LLM System** (11 projects, includes full architecture)
- **Room Visualization** (11 projects)

**Largest Agent Files:**
- Personal Growth Coach (42KB)
- User Profiling & Analytics (40KB)
- Conversational AI Designer (39KB)
- LLM Personalization Specialist (37KB)
- Memory & Context Manager (37KB)

**Quick Wins:**
- Rich Python CLI projects (beautiful results fast)
- Data visualization practice (immediate visual feedback)
- ML deployment basics (get models in production)

---

## 📧 Questions?

Each agent file contains detailed documentation, best practices, and code examples. Start with any agent that matches your current project needs!

Happy building! 🚀
