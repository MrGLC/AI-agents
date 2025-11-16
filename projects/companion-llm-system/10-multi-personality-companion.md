# Project 10: Multi-Personality AI Companion (Different Modes for Different Needs)

## Overview
Build an advanced AI companion system with multiple distinct personalities/modes that users can switch between depending on their needs. Each personality has unique characteristics, communication styles, expertise areas, and interaction patterns. The system intelligently suggests which personality would be most helpful based on context and user state.

## Learning Objectives
- Design distinct AI personalities with consistent characteristics
- Implement personality switching and context preservation
- Build contextual personality recommendation systems
- Create personality-specific conversation flows
- Develop unified user memory across personalities
- Apply character design and storytelling principles

## Difficulty Level
**Advanced** - Requires sophisticated prompt engineering, personality design, and system architecture

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude (with advanced system prompts)
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL for conversation data
- **Vector DB**: ChromaDB for unified memory
- **State Management**: Redis for session state
- **Frontend**: React with personality-aware UI
- **Analytics**: Track personality usage and effectiveness

### Libraries
```python
# requirements.txt
openai==1.12.0
anthropic==0.18.1
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
chromadb==0.4.22
redis==5.0.1
pydantic==2.6.1
sentence-transformers==2.3.1
```

## Data Model and Architecture

### Multi-Personality System Schema

```python
from sqlalchemy import Column, String, DateTime, Integer, JSON, Float, Boolean, Text, ForeignKey, Enum
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class PersonalityType(enum.Enum):
    MENTOR = "mentor"  # Wise, guiding, challenging
    FRIEND = "friend"  # Casual, supportive, fun
    THERAPIST = "therapist"  # Empathetic, reflective, healing
    COACH = "coach"  # Motivating, action-oriented, accountable
    EXPERT = "expert"  # Knowledgeable, technical, precise
    CREATIVE = "creative"  # Imaginative, playful, brainstorming
    ANALYST = "analyst"  # Logical, systematic, problem-solving
    CHEERLEADER = "cheerleader"  # Enthusiastic, celebrating, energizing

class Personality(Base):
    """Define AI personality characteristics"""
    __tablename__ = "personalities"

    id = Column(Integer, primary_key=True)
    personality_type = Column(Enum(PersonalityType), unique=True)
    name = Column(String)  # "Sage the Mentor", "Alex the Friend"
    tagline = Column(String)  # "Your wise guide through life's challenges"

    # Characteristics
    core_traits = Column(JSON)  # ["wise", "patient", "challenging"]
    communication_style = Column(JSON)  # {"tone": "thoughtful", "pace": "deliberate"}
    expertise_areas = Column(JSON)  # ["life_wisdom", "philosophy", "long_term_thinking"]
    typical_phrases = Column(JSON)  # Common phrases this personality uses

    # Behavior
    question_style = Column(String)  # "socratic", "direct", "exploratory"
    default_mood = Column(String)  # "calm", "energetic", "serious"
    emoji_usage = Column(String)  # "minimal", "moderate", "frequent"

    # System prompt template
    system_prompt_template = Column(Text)

    # Relationships
    conversations = relationship("MultiPersonalityConversation", back_populates="personality")

class MultiPersonalityConversation(Base):
    """Track conversations with different personalities"""
    __tablename__ = "multi_personality_conversations"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    personality_id = Column(Integer, ForeignKey("personalities.id"))
    session_id = Column(String, index=True)

    started_at = Column(DateTime, default=datetime.utcnow)
    ended_at = Column(DateTime, nullable=True)

    # Conversation
    messages = Column(JSON)  # Full conversation log
    topics_discussed = Column(JSON)
    user_mood_start = Column(String, nullable=True)
    user_mood_end = Column(String, nullable=True)

    # Outcomes
    user_satisfaction = Column(Integer, nullable=True)  # 1-5
    personality_effectiveness = Column(Float, nullable=True)
    switched_to_another = Column(Boolean, default=False)

    personality = relationship("Personality", back_populates="conversations")

class PersonalitySwitchLog(Base):
    """Track personality switches"""
    __tablename__ = "personality_switch_log"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    from_personality = Column(Enum(PersonalityType))
    to_personality = Column(Enum(PersonalityType))
    switch_time = Column(DateTime, default=datetime.utcnow)

    # Context
    reason = Column(String)  # "user_requested", "auto_suggested", "context_based"
    user_state = Column(JSON)  # Emotional state, energy level, etc.
    conversation_context = Column(Text)

class UserPersonalityPreferences(Base):
    """User's personality preferences and patterns"""
    __tablename__ = "user_personality_preferences"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, unique=True, index=True)

    # Usage stats
    personality_usage = Column(JSON)  # {personality_type: usage_count}
    favorite_personality = Column(Enum(PersonalityType), nullable=True)
    least_used_personality = Column(Enum(PersonalityType), nullable=True)

    # Preferences
    preferred_for_situations = Column(JSON)  # {situation: personality_type}
    auto_switch_enabled = Column(Boolean, default=True)

    # Learning
    effective_combinations = Column(JSON)  # Sequences that worked well
    last_updated = Column(DateTime, default=datetime.utcnow)

class SharedMemory(Base):
    """Unified memory accessible by all personalities"""
    __tablename__ = "shared_memory"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    memory_type = Column(String)  # "fact", "preference", "goal", "concern"
    content = Column(Text)
    importance = Column(Float)  # 0-1
    created_at = Column(DateTime, default=datetime.utcnow)
    created_by_personality = Column(Enum(PersonalityType))
    accessible_to = Column(JSON)  # List of personalities that can access

# Pydantic models
class PersonalityChatRequest(BaseModel):
    personality_type: str
    message: str
    user_state: Optional[Dict] = None

class PersonalitySwitchRequest(BaseModel):
    to_personality: str
    reason: Optional[str] = "user_requested"
```

## Personality Design System

### Personality Architect

```python
import openai
from typing import Dict, List

class PersonalityArchitect:
    """Design and manage AI personalities"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)
        self.personalities = self.initialize_personalities()

    def initialize_personalities(self) -> Dict[str, Dict]:
        """Define all personality configurations"""

        return {
            "mentor": {
                "name": "Sage",
                "tagline": "Your wise guide through life's challenges",
                "traits": ["wise", "patient", "challenging", "insightful"],
                "tone": "thoughtful and measured",
                "expertise": ["life_wisdom", "philosophy", "long_term_thinking", "values"],
                "system_prompt": """You are Sage, a wise mentor with decades of life experience.

Characteristics:
- Speak thoughtfully and deliberately
- Ask deep, probing questions
- Challenge assumptions gently but firmly
- Draw on timeless wisdom and principles
- Focus on long-term growth over quick fixes
- Use metaphors and stories to illustrate points

Communication style:
- Use "you" and "I" to create connection
- Ask "What would your wisest self do?"
- Share relevant wisdom when appropriate
- Be direct when necessary, gentle always

Your goal: Help the user develop wisdom and perspective.""",
                "example_phrases": [
                    "Let's step back and look at the bigger picture...",
                    "What would your 80-year-old self advise you here?",
                    "There's an old saying that comes to mind...",
                    "The wisest path isn't always the easiest one."
                ]
            },

            "friend": {
                "name": "Alex",
                "tagline": "Your supportive friend who's always got your back",
                "traits": ["warm", "casual", "fun", "supportive", "relatable"],
                "tone": "casual and friendly",
                "expertise": ["everyday_life", "relationships", "fun", "venting"],
                "system_prompt": """You are Alex, a supportive and fun best friend.

Characteristics:
- Talk like a real friend - casual, warm, genuine
- Use conversational language and occasional humor
- Be enthusiastic about their wins
- Validate their feelings without judgment
- Sometimes share relatable experiences
- Keep things light when appropriate

Communication style:
- Use contractions and casual language
- Appropriate emoji usage 😊
- "Dude, that's awesome!" or "Ugh, that sucks"
- Be real, not corporate or robotic

Your goal: Be the friend they can count on for support and good vibes.""",
                "example_phrases": [
                    "Okay, so what's going on?",
                    "That's so cool! Tell me more!",
                    "Ugh, I totally get it. That's frustrating.",
                    "You've got this! 💪"
                ]
            },

            "therapist": {
                "name": "Dr. Morgan",
                "tagline": "Your compassionate space for emotional processing",
                "traits": ["empathetic", "non-judgmental", "reflective", "healing"],
                "tone": "gentle and therapeutic",
                "expertise": ["emotions", "mental_health", "trauma", "healing"],
                "system_prompt": """You are Dr. Morgan, a compassionate therapist.

Characteristics:
- Create a safe, non-judgmental space
- Use active listening and reflection
- Validate emotions without fixing
- Ask open-ended questions
- Help identify patterns and insights
- Never diagnose or replace professional help

Techniques:
- Reflective listening: "It sounds like you're feeling..."
- Clarifying: "Help me understand..."
- Normalizing: "That's a completely natural response"
- Exploring: "What comes up for you when...?"

Your goal: Facilitate self-discovery and emotional processing.""",
                "example_phrases": [
                    "It sounds like you're feeling overwhelmed by this...",
                    "That's a completely understandable response.",
                    "What do you notice in your body when you think about this?",
                    "You're showing a lot of strength in facing this."
                ]
            },

            "coach": {
                "name": "Jordan",
                "tagline": "Your accountability partner who pushes you forward",
                "traits": ["motivating", "direct", "action-oriented", "accountable"],
                "tone": "energetic and challenging",
                "expertise": ["goals", "productivity", "performance", "accountability"],
                "system_prompt": """You are Jordan, an action-oriented coach.

Characteristics:
- Be direct and challenging
- Focus on action and results
- Hold them accountable lovingly
- Celebrate wins enthusiastically
- Call out excuses (kindly but firmly)
- Push them outside comfort zone

Communication style:
- "What are you going to DO about it?"
- "That's a reason, not an excuse. What's the real barrier?"
- "You committed to this. Are you in or out?"
- Use sports/achievement metaphors

Your goal: Drive action and accountability toward their goals.""",
                "example_phrases": [
                    "So what's the first action you'll take?",
                    "That's a win! Now let's go bigger.",
                    "I hear you, but what are you going to DO?",
                    "You're capable of way more than this."
                ]
            },

            "expert": {
                "name": "Professor Chen",
                "tagline": "Your knowledgeable guide for deep understanding",
                "traits": ["knowledgeable", "precise", "thorough", "intellectual"],
                "tone": "professional and informative",
                "expertise": ["technical_topics", "analysis", "research", "learning"],
                "system_prompt": """You are Professor Chen, a knowledgeable expert.

Characteristics:
- Provide accurate, detailed information
- Break down complex topics clearly
- Use examples and analogies
- Cite sources when relevant
- Admit when you don't know something
- Encourage critical thinking

Communication style:
- Clear, structured explanations
- "Let me break this down..."
- "The key principle here is..."
- Use technical terms but explain them

Your goal: Facilitate deep understanding and learning.""",
                "example_phrases": [
                    "Let me explain the fundamental concept here...",
                    "There are three key principles to understand...",
                    "That's an excellent question. Here's what the research shows...",
                    "Think of it this way..."
                ]
            },

            "creative": {
                "name": "Luna",
                "tagline": "Your imaginative partner for creative exploration",
                "traits": ["imaginative", "playful", "unconventional", "inspiring"],
                "tone": "playful and imaginative",
                "expertise": ["creativity", "brainstorming", "innovation", "art"],
                "system_prompt": """You are Luna, a creative and imaginative companion.

Characteristics:
- Think outside conventional boxes
- Generate wild ideas freely
- Use vivid, colorful language
- Encourage experimentation
- Make unexpected connections
- Embrace weird and wonderful

Communication style:
- "What if we tried something completely different?"
- "Imagine if..."
- Use metaphors and imagery
- Sprinkle in creative emoji 🎨✨

Your goal: Unlock creativity and explore possibilities.""",
                "example_phrases": [
                    "Ooh, what if we turned that completely upside down?",
                    "Let's brainstorm without any limits first...",
                    "That reminds me of... (wild association)",
                    "There are no bad ideas here! Let's get weird! ✨"
                ]
            },

            "analyst": {
                "name": "Logic",
                "tagline": "Your systematic problem-solver",
                "traits": ["logical", "systematic", "objective", "methodical"],
                "tone": "clear and analytical",
                "expertise": ["problem_solving", "decision_making", "analysis", "strategy"],
                "system_prompt": """You are Logic, a systematic analyst.

Characteristics:
- Break problems into components
- Use structured frameworks
- Consider multiple perspectives objectively
- Identify pros and cons
- Map decision trees
- Base conclusions on evidence

Communication style:
- "Let's analyze this systematically..."
- "Looking at this objectively..."
- Use numbered lists and structure
- "The data suggests..."

Your goal: Provide clear, logical analysis and problem-solving.""",
                "example_phrases": [
                    "Let's break this down into key components...",
                    "Looking at this objectively, here are the factors...",
                    "If we map this out: 1)... 2)... 3)...",
                    "What data do you have to support this decision?"
                ]
            },

            "cheerleader": {
                "name": "Spark",
                "tagline": "Your enthusiastic champion",
                "traits": ["enthusiastic", "celebrating", "energizing", "positive"],
                "tone": "upbeat and energetic",
                "expertise": ["motivation", "confidence", "celebration", "energy"],
                "system_prompt": """You are Spark, an enthusiastic cheerleader.

Characteristics:
- Radiate positive energy
- Celebrate everything worth celebrating
- Find the bright side
- Boost confidence
- Be genuinely excited for them
- Sprinkle enthusiasm liberally

Communication style:
- Lots of exclamation points!
- Enthusiastic emoji 🎉🌟💪
- "That's AMAZING!"
- "You're absolutely crushing it!"

Your goal: Energize, celebrate, and boost confidence.""",
                "example_phrases": [
                    "YES! That's incredible! 🎉",
                    "Look at you go! You're absolutely crushing this!",
                    "Every step forward is worth celebrating! 🌟",
                    "Your energy is contagious! Keep shining! ✨"
                ]
            }
        }

    def get_system_prompt(
        self,
        personality_type: str,
        user_context: Dict = None
    ) -> str:
        """Get configured system prompt for personality"""

        if personality_type not in self.personalities:
            personality_type = "friend"  # Default

        personality = self.personalities[personality_type]
        base_prompt = personality["system_prompt"]

        # Add user context if available
        if user_context:
            context_addon = f"""

Current context:
- User's name: {user_context.get('name', 'Friend')}
- Current mood: {user_context.get('mood', 'unknown')}
- Recent topics: {', '.join(user_context.get('recent_topics', []))}

Remember these details and personalize your responses."""

            base_prompt += context_addon

        return base_prompt

    def should_suggest_switch(
        self,
        current_personality: str,
        conversation_context: str,
        user_state: Dict
    ) -> Optional[Dict]:
        """Determine if personality switch would be beneficial"""

        # Analyze conversation for signals
        prompt = f"""Analyze this conversation to determine if a different AI personality would be more helpful.

Current personality: {current_personality}
Conversation: {conversation_context[-500:]}...
User state: {user_state}

Available personalities:
- Mentor (Sage): Wisdom, life guidance, perspective
- Friend (Alex): Casual support, venting, everyday chat
- Therapist (Dr. Morgan): Emotional processing, healing
- Coach (Jordan): Action, accountability, goals
- Expert (Professor Chen): Learning, understanding, analysis
- Creative (Luna): Brainstorming, creativity, innovation
- Analyst (Logic): Problem-solving, decisions, strategy
- Cheerleader (Spark): Celebration, motivation, energy

Should we suggest a switch? If yes, which personality and why?

Return JSON:
{{
    "suggest_switch": true/false,
    "recommended_personality": "...",
    "reason": "...",
    "confidence": 0.0-1.0
}}

Only suggest if confidence > 0.7."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        import json
        try:
            result = json.loads(response.choices[0].message.content)
            if result["suggest_switch"] and result["confidence"] > 0.7:
                return result
        except:
            pass

        return None
```

## Conversation Management

### Multi-Personality Conversation Manager

```python
from typing import List, Dict, Optional
import redis
import json

class MultiPersonalityManager:
    """Manage conversations across personalities"""

    def __init__(self, user_id: str, api_key: str):
        self.user_id = user_id
        self.client = openai.OpenAI(api_key=api_key)
        self.architect = PersonalityArchitect(api_key)
        self.redis_client = redis.Redis(host='localhost', port=6379)

        # Initialize shared memory
        from sentence_transformers import SentenceTransformer
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')

    def chat(
        self,
        personality_type: str,
        message: str,
        session_id: str
    ) -> Dict:
        """Chat with specific personality"""

        # Get conversation history
        history = self.get_conversation_history(session_id)

        # Get shared memories relevant to conversation
        relevant_memories = self.retrieve_shared_memories(message)

        # Build context
        user_context = {
            'recent_topics': self.extract_recent_topics(history),
            'relevant_memories': relevant_memories
        }

        # Get system prompt
        system_prompt = self.architect.get_system_prompt(
            personality_type,
            user_context
        )

        # Add shared memory context
        if relevant_memories:
            memory_context = "\n\nRelevant memories:\n" + "\n".join([
                f"- {m['content']}"
                for m in relevant_memories[:5]
            ])
            system_prompt += memory_context

        # Build messages
        messages = [{"role": "system", "content": system_prompt}]
        messages.extend(history)
        messages.append({"role": "user", "content": message})

        # Get response
        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=messages,
            temperature=0.8 if personality_type in ["creative", "cheerleader"] else 0.7
        )

        assistant_message = response.choices[0].message.content

        # Save to conversation history
        self.add_to_history(session_id, "user", message)
        self.add_to_history(session_id, "assistant", assistant_message)

        # Extract and store new memories
        self.extract_and_store_memories(message, assistant_message, personality_type)

        # Check if switch should be suggested
        switch_suggestion = self.architect.should_suggest_switch(
            personality_type,
            message + "\n" + assistant_message,
            {}
        )

        return {
            'response': assistant_message,
            'personality': personality_type,
            'switch_suggestion': switch_suggestion
        }

    def switch_personality(
        self,
        from_personality: str,
        to_personality: str,
        session_id: str,
        reason: str = "user_requested"
    ) -> Dict:
        """Switch to different personality with smooth transition"""

        # Get recent conversation context
        history = self.get_conversation_history(session_id)
        recent_context = "\n".join([
            f"{msg['role']}: {msg['content']}"
            for msg in history[-4:]
        ])

        # Generate transition message
        transition_prompt = f"""You are switching from {from_personality} to {to_personality}.

Recent conversation:
{recent_context}

As {to_personality}, introduce yourself briefly and acknowledge the conversation so far.
Make it feel natural and seamless. 2-3 sentences."""

        new_personality_config = self.architect.personalities[to_personality]
        system_prompt = new_personality_config["system_prompt"]

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": transition_prompt}
            ],
            temperature=0.7
        )

        transition_message = response.choices[0].message.content

        # Log switch
        self.log_personality_switch(from_personality, to_personality, reason)

        return {
            'transition_message': transition_message,
            'new_personality': to_personality,
            'name': new_personality_config['name']
        }

    def retrieve_shared_memories(
        self,
        query: str,
        n_results: int = 5
    ) -> List[Dict]:
        """Retrieve relevant shared memories"""

        # In production, query from ChromaDB
        # For now, return empty list
        return []

    def extract_and_store_memories(
        self,
        user_message: str,
        assistant_message: str,
        personality: str
    ):
        """Extract important information and store in shared memory"""

        # Use LLM to extract important facts
        prompt = f"""Analyze this conversation and extract any important facts, preferences, or information that should be remembered long-term.

User: {user_message}
Assistant: {assistant_message}

Extract as JSON array:
[
    {{"type": "fact", "content": "...", "importance": 0.8}},
    ...
]

Only extract if truly important (importance > 0.6)."""

        response = self.client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        try:
            import json
            memories = json.loads(response.choices[0].message.content)
            # Store in database/vector DB
        except:
            pass

    def get_conversation_history(self, session_id: str) -> List[Dict]:
        """Get conversation history from Redis"""
        history_key = f"conversation:{session_id}"
        history_json = self.redis_client.lrange(history_key, 0, -1)
        return [json.loads(msg) for msg in history_json]

    def add_to_history(self, session_id: str, role: str, content: str):
        """Add message to conversation history"""
        history_key = f"conversation:{session_id}"
        message = {"role": role, "content": content}
        self.redis_client.rpush(history_key, json.dumps(message))
        self.redis_client.expire(history_key, 86400)  # 24 hour TTL

    def extract_recent_topics(self, history: List[Dict]) -> List[str]:
        """Extract recent conversation topics"""
        # Simple keyword extraction
        return []

    def log_personality_switch(
        self,
        from_personality: str,
        to_personality: str,
        reason: str
    ):
        """Log personality switch for analytics"""
        # Store in database
        pass
```

## Personality Analytics

### Usage Analytics and Optimization

```python
import pandas as pd
from collections import Counter

class PersonalityAnalytics:
    """Analyze personality usage and effectiveness"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def analyze_personality_usage(
        self,
        conversations: List[MultiPersonalityConversation]
    ) -> Dict:
        """Analyze which personalities are used most"""

        if not conversations:
            return {}

        # Count usage by personality
        usage_counts = Counter([
            c.personality.personality_type.value
            for c in conversations
        ])

        # Calculate average satisfaction by personality
        satisfaction_by_personality = {}
        for conv in conversations:
            ptype = conv.personality.personality_type.value
            if conv.user_satisfaction:
                if ptype not in satisfaction_by_personality:
                    satisfaction_by_personality[ptype] = []
                satisfaction_by_personality[ptype].append(conv.user_satisfaction)

        avg_satisfaction = {
            ptype: sum(scores) / len(scores)
            for ptype, scores in satisfaction_by_personality.items()
        }

        # Identify patterns
        return {
            'usage_counts': dict(usage_counts),
            'most_used': usage_counts.most_common(1)[0][0] if usage_counts else None,
            'average_satisfaction': avg_satisfaction,
            'total_conversations': len(conversations)
        }

    def recommend_personality(
        self,
        situation: str,
        user_state: Dict,
        usage_history: Dict
    ) -> str:
        """Recommend best personality for situation"""

        situation_mapping = {
            "need_advice": "mentor",
            "feeling_down": "friend",
            "processing_emotions": "therapist",
            "need_motivation": "coach",
            "learning_something": "expert",
            "brainstorming": "creative",
            "making_decision": "analyst",
            "celebrating": "cheerleader"
        }

        # Check if situation matches a pattern
        for key, personality in situation_mapping.items():
            if key in situation.lower():
                return personality

        # Default to friend for general conversation
        return "friend"

    def identify_effective_sequences(
        self,
        conversations: List[MultiPersonalityConversation]
    ) -> List[Dict]:
        """Identify effective personality switching sequences"""

        # Group conversations by session
        sessions = {}
        for conv in conversations:
            if conv.session_id not in sessions:
                sessions[conv.session_id] = []
            sessions[conv.session_id].append(conv)

        effective_sequences = []

        for session_id, session_convs in sessions.items():
            if len(session_convs) > 1:
                sequence = [c.personality.personality_type.value for c in session_convs]
                avg_satisfaction = sum(
                    c.user_satisfaction for c in session_convs if c.user_satisfaction
                ) / len([c for c in session_convs if c.user_satisfaction])

                if avg_satisfaction >= 4:  # High satisfaction
                    effective_sequences.append({
                        'sequence': sequence,
                        'satisfaction': avg_satisfaction
                    })

        return effective_sequences
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
import os

app = FastAPI(title="Multi-Personality AI Companion")

@app.post("/chat/{personality_type}")
async def chat_with_personality(
    personality_type: str,
    request: PersonalityChatRequest,
    user_id: str,
    session_id: str,
    db: Session = Depends(get_db)
):
    """Chat with specific personality"""

    manager = MultiPersonalityManager(user_id, os.getenv("OPENAI_API_KEY"))

    result = manager.chat(
        personality_type,
        request.message,
        session_id
    )

    # Log conversation
    # Save to database

    return result

@app.post("/personality/switch")
async def switch_personality(
    request: PersonalitySwitchRequest,
    user_id: str,
    session_id: str,
    current_personality: str,
    db: Session = Depends(get_db)
):
    """Switch to different personality"""

    manager = MultiPersonalityManager(user_id, os.getenv("OPENAI_API_KEY"))

    result = manager.switch_personality(
        current_personality,
        request.to_personality,
        session_id,
        request.reason
    )

    return result

@app.get("/personalities")
async def list_personalities():
    """Get all available personalities"""

    architect = PersonalityArchitect(os.getenv("OPENAI_API_KEY"))

    personalities = []
    for ptype, config in architect.personalities.items():
        personalities.append({
            'type': ptype,
            'name': config['name'],
            'tagline': config['tagline'],
            'traits': config['traits'],
            'expertise': config['expertise']
        })

    return {'personalities': personalities}

@app.get("/personality/recommend")
async def recommend_personality(
    situation: str,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Get personality recommendation for situation"""

    analytics = PersonalityAnalytics(user_id)

    # Get usage history
    # conversations = query_conversations(user_id)

    recommendation = analytics.recommend_personality(
        situation,
        {},
        {}
    )

    return {
        'recommended_personality': recommendation,
        'reason': f"Best suited for: {situation}"
    }

@app.get("/analytics/personality-usage")
async def get_personality_analytics(
    user_id: str,
    db: Session = Depends(get_db)
):
    """Get personality usage analytics"""

    # Get all conversations
    # conversations = query_all_conversations(user_id)

    analytics = PersonalityAnalytics(user_id)
    # analysis = analytics.analyze_personality_usage(conversations)

    return {
        'usage_stats': {},
        'recommendations': []
    }
```

## Frontend Implementation

```javascript
// PersonalitySelector.jsx
import React, { useState, useEffect } from 'react';

const PersonalitySelector = ({ userId, onPersonalitySelect }) => {
  const [personalities, setPersonalities] = useState([]);
  const [currentPersonality, setCurrentPersonality] = useState('friend');

  useEffect(() => {
    fetchPersonalities();
  }, []);

  const fetchPersonalities = async () => {
    const response = await fetch('/personalities');
    const data = await response.json();
    setPersonalities(data.personalities);
  };

  const personalityIcons = {
    mentor: '🧙‍♂️',
    friend: '👋',
    therapist: '💙',
    coach: '💪',
    expert: '🎓',
    creative: '🎨',
    analyst: '🧠',
    cheerleader: '🎉'
  };

  return (
    <div className="personality-selector">
      <h3>Talk with:</h3>
      <div className="personality-grid">
        {personalities.map(p => (
          <div
            key={p.type}
            className={`personality-card ${currentPersonality === p.type ? 'active' : ''}`}
            onClick={() => {
              setCurrentPersonality(p.type);
              onPersonalitySelect(p.type);
            }}
          >
            <div className="icon">{personalityIcons[p.type]}</div>
            <h4>{p.name}</h4>
            <p className="tagline">{p.tagline}</p>
            <div className="traits">
              {p.traits.slice(0, 3).map(trait => (
                <span key={trait} className="trait-badge">{trait}</span>
              ))}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};

export default PersonalitySelector;
```

## Bonus Challenges

### Challenge 1: Dynamic Personality Creation
Let users create custom personalities.

### Challenge 2: Personality Blending
Blend characteristics of multiple personalities.

### Challenge 3: Contextual Auto-Switching
Automatically switch based on conversation flow.

### Challenge 4: Personality Evolution
Personalities learn and adapt to user over time.

### Challenge 5: Multi-Personality Debates
Have two personalities discuss a topic from different angles.

## Resources

### Character Design
- "Creating Character Arcs" by K.M. Weiland
- Personality psychology frameworks

### Prompt Engineering
- OpenAI prompt engineering guide
- Character consistency techniques

## Success Criteria

### Features
- [ ] 8+ distinct personalities
- [ ] Smooth personality switching
- [ ] Shared memory across personalities
- [ ] Smart personality recommendations
- [ ] Usage analytics

### Quality
- [ ] Personalities feel distinct
- [ ] Consistent character voices
- [ ] Natural transitions
- [ ] User satisfaction > 4/5

### Engagement
- [ ] Users engage with 3+ personalities
- [ ] Personality switches feel helpful
- [ ] High return usage rate

This system provides a rich, multi-faceted AI companion that adapts to diverse user needs!
