# Companion LLM System - Complete Architecture

This document describes the complete architecture for building a personalized AI companion that learns from the user and helps them evolve.

## System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    User Interface Layer                          │
│  (Web App, Mobile App, CLI, Voice Interface)                     │
└────────────────┬─────────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────────────────────┐
│                  API Gateway & Router                            │
│  (FastAPI, Authentication, Rate Limiting, Load Balancing)        │
└────────────────┬─────────────────────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
┌───────▼──────┐  ┌──────▼────────────────────────────────────────┐
│ Conversation │  │     Core Intelligence Layer                   │
│   Handler    │  │  ┌──────────────────────────────────────┐    │
└───────┬──────┘  │  │  LLM Engine (GPT-4/Claude/Llama)     │    │
        │         │  │  + Personalization Layer              │    │
        │         │  └──────────────┬───────────────────────┘    │
        │         │                 │                             │
        │         │  ┌──────────────▼───────────────────────┐    │
        │         │  │  RAG System + Vector Database        │    │
        │         │  │  (User memories, knowledge, context) │    │
        │         │  └──────────────────────────────────────┘    │
        │         └───────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────────┐
│                    Data & Memory Layer                           │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │
│  │ PostgreSQL │  │ Redis Cache │  │ Vector DB (ChromaDB)     │ │
│  │ (Structured│  │ (Sessions)  │  │ (Semantic Search)        │ │
│  │  Data)     │  └─────────────┘  └──────────────────────────┘ │
│  └────────────┘                                                 │
└──────────────────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────────┐
│              Analytics & Growth Tracking Layer                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │ User Profile │  │ Goal Tracker │  │ Progress Analytics │    │
│  │  Analytics   │  │   Service    │  │     Dashboard      │    │
│  └──────────────┘  └──────────────┘  └────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. User Profile System

**Data Structure:**
```python
{
    "user_id": "uuid",
    "profile": {
        "name": "string",
        "preferred_name": "string",
        "communication_style": "casual|formal|technical",
        "interests": ["topic1", "topic2"],
        "goals": [
            {
                "goal_id": "uuid",
                "title": "string",
                "category": "health|career|learning|personal",
                "status": "active|paused|completed",
                "milestones": [],
                "progress": 0.0-1.0
            }
        ],
        "personality_traits": {
            "openness": 0.0-1.0,
            "extraversion": 0.0-1.0,
            "motivation_type": "achievement|affiliation|power"
        }
    },
    "preferences": {
        "reminder_frequency": "daily|weekly|custom",
        "tone": "supportive|direct|balanced",
        "privacy_level": "high|medium|low"
    },
    "metadata": {
        "created_at": "timestamp",
        "last_active": "timestamp",
        "total_conversations": 0,
        "total_messages": 0
    }
}
```

### 2. Memory Architecture

#### **A. Short-term Memory (Conversation Context)**
- **Duration**: Current conversation session
- **Storage**: Redis with TTL (1-24 hours)
- **Size**: Last 10-20 messages
- **Purpose**: Maintain conversation coherence

```python
conversation_context = {
    "session_id": "uuid",
    "messages": [
        {"role": "user", "content": "...", "timestamp": "..."},
        {"role": "assistant", "content": "...", "timestamp": "..."}
    ],
    "current_topic": "string",
    "emotional_state": "positive|neutral|negative",
    "ttl": 3600  # 1 hour
}
```

#### **B. Long-term Memory (User History)**
- **Duration**: Permanent (with decay)
- **Storage**: PostgreSQL + Vector DB
- **Size**: Unlimited with importance weighting
- **Purpose**: Remember user over time

```python
long_term_memory = {
    "memory_id": "uuid",
    "user_id": "uuid",
    "content": "string",
    "memory_type": "fact|preference|event|achievement",
    "importance": 0.0-1.0,  # Decays over time
    "last_accessed": "timestamp",
    "access_count": 0,
    "embedding": [...]  # For semantic search
}
```

#### **C. Episodic Memory (Key Moments)**
- **Duration**: Permanent
- **Storage**: PostgreSQL
- **Purpose**: Remember significant events

```python
episodic_memory = {
    "episode_id": "uuid",
    "title": "string",
    "description": "string",
    "date": "timestamp",
    "emotional_valence": -1.0 to 1.0,
    "entities": ["person", "place", "thing"],
    "tags": ["milestone", "achievement", "challenge"]
}
```

#### **D. Semantic Memory (Facts About User)**
- **Duration**: Permanent with updates
- **Storage**: PostgreSQL + Knowledge Graph
- **Purpose**: Structured knowledge

```python
semantic_memory = {
    "fact_id": "uuid",
    "subject": "user",
    "predicate": "likes|dislikes|knows|believes",
    "object": "string",
    "confidence": 0.0-1.0,
    "source": "explicit|inferred",
    "updated_at": "timestamp"
}
```

### 3. Personalization Engine

#### **Personalization Layers:**

**Layer 1: Dynamic Prompts (Fastest)**
- Inject user profile into system prompt
- Adjust tone, formality, references
- No model changes required
- Latency: +0ms

**Layer 2: RAG with User Context (Medium)**
- Retrieve relevant memories
- Add to prompt context
- Semantic search in vector DB
- Latency: +50-200ms

**Layer 3: Fine-tuned Adapter (Deepest)**
- LoRA weights per user
- Trained on conversation history
- Swap adapters per request
- Latency: +0ms (loaded in memory)

#### **Adaptation Strategy:**

```python
def personalize_response(user_id, message):
    # 1. Load user profile
    profile = get_user_profile(user_id)

    # 2. Retrieve relevant memories
    memories = vector_db.search(
        query=message,
        filter={"user_id": user_id},
        limit=5
    )

    # 3. Build personalized prompt
    system_prompt = f"""
You are a personal AI companion for {profile['name']}.

Communication Style: {profile['communication_style']}
Current Goals: {', '.join([g['title'] for g in profile['goals']])}

Relevant Context:
{format_memories(memories)}

Adapt your response to their style and reference their goals when relevant.
"""

    # 4. Generate response
    response = llm.generate(
        system=system_prompt,
        messages=conversation_history + [{"role": "user", "content": message}],
        user_adapter=get_lora_adapter(user_id)  # Optional
    )

    return response
```

### 4. Growth Tracking System

#### **Goal Framework:**

```python
class Goal:
    id: str
    user_id: str
    title: str
    description: str
    category: str  # health, career, learning, personal, financial

    # SMART criteria
    specific: str
    measurable: dict  # {"metric": "steps", "target": 10000, "unit": "daily"}
    achievable: bool
    relevant: str
    time_bound: datetime

    # Progress tracking
    milestones: List[Milestone]
    current_progress: float  # 0.0 to 1.0
    streak: int

    # Behavioral
    obstacles: List[str]
    strategies: List[str]
    accountability_partner: Optional[str]
```

#### **Progress Metrics:**

```python
growth_metrics = {
    "goal_completion_rate": 0.0-1.0,
    "average_progress_per_week": float,
    "streak_length": int,
    "engagement_score": 0.0-1.0,  # Based on check-in frequency
    "self_reported_satisfaction": 1-10,
    "behavioral_change_score": 0.0-1.0
}
```

#### **Reflection System:**

```python
def generate_reflection_prompt(user_id, timeframe="week"):
    progress = get_progress_summary(user_id, timeframe)

    prompts = [
        f"You made {progress['actions_taken']} progress steps this week. What felt most meaningful?",
        f"Your {progress['best_goal']} improved by {progress['best_improvement']}%. What contributed to this?",
        f"You mentioned {progress['obstacles']} as a challenge. What did you learn from that?",
        "Looking back, what's one thing you'd do differently next week?",
        "What are you most proud of from this period?"
    ]

    return select_prompt_based_on_context(prompts, progress)
```

### 5. Conversation Management

#### **State Machine:**

```
[Idle] ──user_message──> [Processing]
   ▲                           │
   │                           ▼
   └────response────── [Responding]
                              │
                              ▼
                        [Waiting_Feedback]
                              │
                              ▼
                     [Learning/Updating_Profile]
                              │
                              └────────> [Idle]
```

#### **Engagement Strategies:**

```python
class EngagementManager:
    def should_check_in(self, user_id):
        """Proactively reach out based on patterns"""
        last_interaction = get_last_interaction(user_id)
        user_pattern = get_typical_activity_pattern(user_id)
        goals = get_active_goals(user_id)

        # Check-in if:
        # 1. User typically engages daily but hasn't in 2 days
        # 2. Goal deadline approaching
        # 3. Milestone reached
        # 4. Pattern suggests optimal time (e.g., morning routine)

        if days_since(last_interaction) > user_pattern['typical_gap'] * 1.5:
            return True, "re-engagement"

        for goal in goals:
            if days_until(goal.deadline) <= 3:
                return True, f"deadline_reminder:{goal.id}"

        return False, None

    def generate_check_in_message(self, user_id, reason):
        if reason == "re-engagement":
            return "Hey! I noticed it's been a few days. How are things going?"
        elif reason.startswith("deadline_reminder"):
            goal_id = reason.split(":")[1]
            goal = get_goal(goal_id)
            return f"Just a heads up - your goal '{goal.title}' deadline is in {days_until(goal.deadline)} days. Need any help?"
```

### 6. Safety and Ethics

#### **Privacy Controls:**

```python
privacy_settings = {
    "data_retention": {
        "conversation_history": "30_days|90_days|1_year|forever",
        "auto_delete_sensitive": True,
        "pii_detection": True
    },
    "data_usage": {
        "personalization": True,
        "analytics": False,
        "model_training": False  # Opt-in only
    },
    "export_controls": {
        "allow_data_export": True,
        "allow_data_deletion": True  # GDPR right to be forgotten
    }
}
```

#### **Safety Guardrails:**

```python
def safety_check(message, context):
    """Multi-layer safety checking"""

    # 1. Crisis detection
    if detect_crisis(message):
        return {
            "safe": True,  # Allow response
            "priority": "urgent",
            "action": "crisis_protocol",
            "resources": get_crisis_resources()
        }

    # 2. Harmful content
    if detect_harmful_intent(message):
        return {
            "safe": False,
            "reason": "harmful_content",
            "alternative": "I notice this conversation is heading in a concerning direction..."
        }

    # 3. Privacy breach
    if requesting_private_data(message):
        return {
            "safe": False,
            "reason": "privacy_violation",
            "alternative": "I'm designed to protect your privacy and can't share that information."
        }

    return {"safe": True}
```

## Implementation Roadmap

### Phase 1: MVP (Weeks 1-4)

**Goals:**
- Basic conversational AI
- Simple user profiles
- Short-term memory only
- Single personality

**Deliverables:**
- FastAPI backend with OpenAI integration
- React frontend with chat interface
- PostgreSQL for user data
- Redis for sessions
- Basic authentication

### Phase 2: Personalization (Weeks 5-8)

**Goals:**
- RAG-based memory system
- User profiling and learning
- Communication style adaptation
- Long-term memory

**Deliverables:**
- ChromaDB/Pinecone integration
- Memory consolidation pipeline
- Profile learning from conversations
- Semantic search for context retrieval

### Phase 3: Growth Tracking (Weeks 9-12)

**Goals:**
- Goal setting and tracking
- Progress monitoring
- Reflection prompts
- Analytics dashboard

**Deliverables:**
- Goal management system
- Progress visualization
- Automated check-ins
- Growth analytics

### Phase 4: Advanced Features (Weeks 13+)

**Goals:**
- LoRA fine-tuning per user
- Multi-personality system
- Proactive engagement
- Advanced analytics

**Deliverables:**
- Fine-tuning pipeline
- Personality switching
- Predictive engagement
- A/B testing framework

## Technology Stack

### Core Technologies

**LLM Options:**
- **OpenAI GPT-4**: Best quality, expensive
- **Anthropic Claude**: Great for long context
- **Open-source (Llama 2, Mistral)**: Self-hosted, cheaper

**Vector Databases:**
- **ChromaDB**: Open-source, easy to start
- **Pinecone**: Managed, scalable
- **Weaviate**: Feature-rich, self-hosted option

**Frameworks:**
- **LangChain**: LLM orchestration
- **LlamaIndex**: RAG and indexing
- **Haystack**: Alternative to LangChain

**Backend:**
- **FastAPI**: Modern, async Python
- **PostgreSQL**: Structured data
- **Redis**: Caching and sessions

**Frontend:**
- **React** + TypeScript
- **TailwindCSS**: Styling
- **Recharts**: Analytics viz

**Analytics:**
- **PostHog**: Product analytics
- **Mixpanel**: User analytics
- **Custom**: In-house dashboard

## Evaluation Metrics

### User Satisfaction
- **Net Promoter Score (NPS)**: -100 to 100
- **User retention**: % active after 30/60/90 days
- **Session length**: Average time per session
- **Message frequency**: Messages per day/week

### Personalization Effectiveness
- **Context recall accuracy**: Can AI remember user details?
- **Response relevance**: User ratings per response
- **Style matching**: Similarity to user's style
- **Proactive engagement success**: % positive responses to check-ins

### Goal Achievement
- **Goal completion rate**: % of goals achieved
- **Average progress velocity**: Progress per week
- **Streak maintenance**: Average streak length
- **User-reported impact**: Self-assessment scores

### System Performance
- **Response latency**: P50, P95, P99
- **Error rate**: % of failed requests
- **Uptime**: % availability
- **Cost per user**: Monthly cost

## Example: Complete User Journey

```python
# Day 1: User signs up
user = create_user(email="user@example.com", name="Alex")
profile = initialize_profile(user, onboarding_answers={
    "goals": ["Get fit", "Learn Spanish"],
    "style": "casual",
    "interests": ["tech", "travel"]
})

# Day 2: First conversation
response = chat(
    user_id=user.id,
    message="Hey! I want to start working out but I'm not sure where to begin."
)
# AI: "That's awesome that you want to get fit! Since you mentioned this as a goal,
#      let's make it specific. What does 'get fit' mean to you? Losing weight,
#      building muscle, or just feeling more energetic?"

# User refines goal
goal = create_goal(
    user_id=user.id,
    title="Exercise 3x per week",
    measurable={"metric": "workouts", "target": 3, "period": "week"}
)

# Day 3: Check-in
if should_check_in(user.id):
    send_notification(user.id, "Morning! Did you get that workout in yesterday?")

# Week 2: Progress review
progress = analyze_progress(user.id, goal.id, timeframe="week")
reflection_prompt = generate_reflection(progress)
# "You hit your workout goal 2 out of 3 times this week! What made Wednesday tough?"

# Month 1: Pattern detected
patterns = analyze_behavior(user.id)
# Detected: User works out more on Mondays after weekend
# Recommendation: "I notice you have great energy on Mondays. Want to set that
#                  as your consistent workout day?"

# Month 3: Growth milestone
if goal.progress >= 1.0:
    celebrate_achievement(user.id, goal.id)
    generate_next_goal_suggestions(user.id)
```

## Conclusion

This architecture provides a comprehensive foundation for building a personalized AI companion that:
- Learns and adapts to each user
- Tracks growth and provides accountability
- Respects privacy and safety
- Scales to millions of users
- Continuously improves through feedback

Start with the MVP (Phase 1) and iterate based on user feedback and metrics. The system is designed to be modular - each component can be improved independently while maintaining the overall architecture.
