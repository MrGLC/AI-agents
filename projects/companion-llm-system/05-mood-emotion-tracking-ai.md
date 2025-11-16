# Project 05: Mood and Emotion Tracking Conversational AI

## Overview
Build an empathetic AI companion that tracks emotional states over time, helps users understand their emotional patterns, provides support during difficult times, and celebrates positive moments. The system combines emotion detection, mood journaling, and therapeutic conversation techniques.

## Learning Objectives
- Implement emotion detection from text and voice
- Design mood tracking and visualization systems
- Apply conversational AI for emotional support
- Build pattern recognition for emotional triggers
- Create personalized coping strategies
- Implement crisis detection and intervention

## Difficulty Level
**Intermediate to Advanced** - Requires NLP, emotional intelligence, and ethical considerations

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude (with careful prompting)
- **Emotion AI**: Hume AI, Google Cloud NLP, or transformer models
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL + TimescaleDB for time-series data
- **Frontend**: React with emotion-focused UI components
- **Analytics**: pandas, scikit-learn for pattern analysis
- **Visualization**: D3.js, Recharts for emotion timelines

### Libraries
```python
# requirements.txt
openai==1.12.0
anthropic==0.18.1
transformers==4.37.2
torch==2.1.2
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
psycopg2-binary==2.9.9
pandas==2.1.4
numpy==1.26.3
scikit-learn==1.4.0
pydantic==2.6.1
python-dateutil==2.8.2
plotly==5.18.0

# Emotion detection
emotion==0.2.0
vaderSentiment==3.3.2

# Optional: Speech emotion recognition
librosa==0.10.1
soundfile==0.12.1
```

## Data Model and Architecture

### Emotion Tracking Schema

```python
from sqlalchemy import Column, String, DateTime, Float, Integer, JSON, Text, Enum
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class EmotionType(enum.Enum):
    JOY = "joy"
    SADNESS = "sadness"
    ANGER = "anger"
    FEAR = "fear"
    SURPRISE = "surprise"
    DISGUST = "disgust"
    NEUTRAL = "neutral"
    LOVE = "love"
    ANXIETY = "anxiety"
    EXCITEMENT = "excitement"

class MoodLevel(enum.Enum):
    VERY_LOW = 1
    LOW = 2
    MODERATE = 3
    GOOD = 4
    GREAT = 5

class MoodEntry(Base):
    """Track individual mood entries"""
    __tablename__ = "mood_entries"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)

    # Primary emotion
    primary_emotion = Column(Enum(EmotionType))
    emotion_intensity = Column(Float)  # 0.0 to 1.0

    # Additional emotions (can have multiple)
    emotions_detected = Column(JSON)  # {"joy": 0.7, "excitement": 0.4}

    # Mood rating
    mood_level = Column(Enum(MoodLevel))

    # Context
    text_entry = Column(Text, nullable=True)
    activities = Column(JSON)  # ["work", "exercise", "social"]
    location = Column(String, nullable=True)
    weather = Column(String, nullable=True)

    # Analysis
    sentiment_score = Column(Float)  # -1 to 1
    triggers = Column(JSON)  # Identified triggers
    coping_strategies_used = Column(JSON)

    # Metadata
    entry_method = Column(String)  # "chat", "voice", "quick_log"
    ai_response = Column(Text, nullable=True)

class EmotionalPattern(Base):
    """Store identified emotional patterns"""
    __tablename__ = "emotional_patterns"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    pattern_type = Column(String)  # "time_of_day", "activity", "trigger"
    pattern_description = Column(Text)
    confidence = Column(Float)
    detected_at = Column(DateTime, default=datetime.utcnow)
    occurrences = Column(Integer)
    metadata = Column(JSON)

class CopingStrategy(Base):
    """Track coping strategies and their effectiveness"""
    __tablename__ = "coping_strategies"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    strategy_name = Column(String)
    description = Column(Text)
    category = Column(String)  # "breathing", "physical", "social", "cognitive"

    # Effectiveness tracking
    times_used = Column(Integer, default=0)
    times_helpful = Column(Integer, default=0)
    avg_effectiveness = Column(Float, default=0.0)  # 0-1

    created_at = Column(DateTime, default=datetime.utcnow)

class SupportConversation(Base):
    """Log supportive conversations"""
    __tablename__ = "support_conversations"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    mood_entry_id = Column(Integer, nullable=True)
    timestamp = Column(DateTime, default=datetime.utcnow)

    initial_emotion = Column(Enum(EmotionType))
    final_emotion = Column(Enum(EmotionType))

    conversation_log = Column(JSON)  # Full conversation
    techniques_used = Column(JSON)  # ["validation", "reframing", "grounding"]

    was_helpful = Column(Integer, nullable=True)  # User feedback: 1-5
    crisis_detected = Column(Integer, default=False)

# Pydantic models
class MoodEntryCreate(BaseModel):
    mood_level: int  # 1-5
    text_entry: Optional[str] = None
    activities: Optional[List[str]] = None
    location: Optional[str] = None

class EmotionDetectionResult(BaseModel):
    primary_emotion: str
    intensity: float
    all_emotions: Dict[str, float]
    sentiment: float
```

## Emotion Detection System

### Multi-Modal Emotion Detector

```python
from transformers import pipeline
from typing import Dict, List
import numpy as np

class EmotionDetector:
    """Detect emotions from text, voice, and context"""

    def __init__(self):
        # Load emotion classification model
        self.emotion_classifier = pipeline(
            "text-classification",
            model="j-hartmann/emotion-english-distilroberta-base",
            top_k=None
        )

        # Load sentiment analyzer
        from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
        self.sentiment_analyzer = SentimentIntensityAnalyzer()

    def detect_from_text(self, text: str) -> EmotionDetectionResult:
        """Detect emotions from text input"""

        # Get emotion probabilities
        emotions = self.emotion_classifier(text[:512])[0]
        emotion_dict = {e['label']: e['score'] for e in emotions}

        # Find primary emotion
        primary = max(emotions, key=lambda x: x['score'])

        # Get sentiment
        sentiment_scores = self.sentiment_analyzer.polarity_scores(text)
        sentiment = sentiment_scores['compound']  # -1 to 1

        return EmotionDetectionResult(
            primary_emotion=primary['label'],
            intensity=primary['score'],
            all_emotions=emotion_dict,
            sentiment=sentiment
        )

    def detect_mixed_emotions(self, text: str, threshold: float = 0.3) -> List[Dict]:
        """Detect when multiple emotions are present"""
        result = self.detect_from_text(text)

        mixed_emotions = [
            {'emotion': emotion, 'score': score}
            for emotion, score in result.all_emotions.items()
            if score >= threshold
        ]

        return sorted(mixed_emotions, key=lambda x: x['score'], reverse=True)

    def analyze_emotional_shift(
        self,
        previous_entries: List[MoodEntry],
        current_text: str
    ) -> Dict:
        """Detect shifts in emotional state"""

        current_emotion = self.detect_from_text(current_text)

        if not previous_entries:
            return {
                'shift_detected': False,
                'current_emotion': current_emotion.primary_emotion
            }

        recent = previous_entries[0]

        # Check for significant shift
        shift_threshold = 0.4
        is_significant_shift = (
            recent.primary_emotion.value != current_emotion.primary_emotion
            or abs(recent.emotion_intensity - current_emotion.intensity) > shift_threshold
        )

        return {
            'shift_detected': is_significant_shift,
            'previous_emotion': recent.primary_emotion.value,
            'current_emotion': current_emotion.primary_emotion,
            'previous_intensity': recent.emotion_intensity,
            'current_intensity': current_emotion.intensity,
            'direction': 'improved' if current_emotion.sentiment > recent.sentiment_score else 'declined'
        }

    def detect_crisis_indicators(self, text: str) -> Dict:
        """Detect potential crisis situations"""

        crisis_keywords = [
            'suicide', 'suicidal', 'kill myself', 'end it all',
            'no reason to live', 'better off dead', 'self-harm',
            'hurt myself', 'want to die'
        ]

        text_lower = text.lower()

        crisis_detected = any(keyword in text_lower for keyword in crisis_keywords)

        severity = 'high' if crisis_detected else 'none'

        # Additional context analysis
        if crisis_detected:
            # Use LLM to assess severity more accurately
            pass

        return {
            'crisis_detected': crisis_detected,
            'severity': severity,
            'immediate_action_required': crisis_detected
        }
```

## Empathetic Conversation System

### Therapeutic Conversation Handler

```python
import openai
from typing import List, Dict

class EmpathyEngine:
    """Provide empathetic, supportive conversations"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)
        self.conversation_techniques = [
            'active_listening',
            'validation',
            'reframing',
            'grounding',
            'perspective_taking'
        ]

    def respond_to_emotion(
        self,
        user_message: str,
        detected_emotion: EmotionDetectionResult,
        conversation_history: List[Dict] = None
    ) -> str:
        """Generate empathetic response based on emotion"""

        # Build context-aware system prompt
        system_prompt = self.build_empathetic_prompt(detected_emotion)

        messages = [{"role": "system", "content": system_prompt}]

        if conversation_history:
            messages.extend(conversation_history)

        messages.append({"role": "user", "content": user_message})

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=messages,
            temperature=0.7,
            max_tokens=300
        )

        return response.choices[0].message.content

    def build_empathetic_prompt(self, emotion: EmotionDetectionResult) -> str:
        """Build emotion-specific system prompt"""

        emotion_guidelines = {
            'sadness': """You are a compassionate companion. The user is feeling sad.
- Validate their feelings
- Show empathy without minimizing
- Offer gentle support
- Ask if they want to talk about it
- Don't rush to fix or give advice unless asked""",

            'anxiety': """You are a calming presence. The user is anxious.
- Acknowledge their anxiety
- Offer grounding techniques
- Speak in a calm, reassuring tone
- Help them focus on what they can control
- Suggest breathing exercises if appropriate""",

            'anger': """You are a patient listener. The user is angry.
- Validate that anger is a normal emotion
- Create space for them to express safely
- Don't argue or get defensive
- Help them identify what triggered it
- Suggest healthy outlets if appropriate""",

            'joy': """You are a celebratory companion. The user is happy!
- Share in their joy authentically
- Encourage them to savor the moment
- Ask them to elaborate on positive aspects
- Help them remember this feeling
- Be genuinely enthusiastic""",

            'fear': """You are a reassuring presence. The user is afraid.
- Acknowledge their fear without judgment
- Help them assess the situation realistically
- Offer perspective when appropriate
- Remind them of their strengths
- Be a steady, calm presence"""
        }

        base_prompt = emotion_guidelines.get(
            emotion.primary_emotion,
            "You are an empathetic companion. Listen actively and respond with care."
        )

        return f"""{base_prompt}

Guidelines:
- Be warm, genuine, and non-judgmental
- Use reflective listening
- Ask open-ended questions
- Match their energy level appropriately
- Never diagnose or replace professional help
- Encourage self-care and professional support when needed

Keep responses conversational, supportive, and 2-4 sentences."""

    def validate_emotion(self, emotion: str, context: str = "") -> str:
        """Generate validation statement"""

        validations = {
            'sadness': [
                "It's completely understandable to feel sad about this.",
                "Your sadness is valid. It's okay to feel this way.",
                "I hear you. These feelings are real and important."
            ],
            'anxiety': [
                "It makes sense that you're feeling anxious about this.",
                "Anxiety can be overwhelming. Your feelings are valid.",
                "It's natural to feel anxious in situations like this."
            ],
            'anger': [
                "Your anger is understandable. It's a valid response.",
                "It's okay to feel angry about this.",
                "I can see why this would make you angry."
            ],
            'joy': [
                "I'm so happy for you!",
                "That's wonderful! You deserve to feel this joy.",
                "What a beautiful moment to celebrate!"
            ]
        }

        import random
        return random.choice(validations.get(emotion, ["I hear you."]))

    def suggest_coping_strategy(
        self,
        emotion: EmotionType,
        user_preferences: Dict = None
    ) -> Dict:
        """Suggest appropriate coping strategy"""

        strategies = {
            EmotionType.ANXIETY: [
                {
                    'name': '4-7-8 Breathing',
                    'description': 'Breathe in for 4, hold for 7, exhale for 8',
                    'category': 'breathing',
                    'duration': '5 minutes'
                },
                {
                    'name': 'Grounding Exercise',
                    'description': 'Name 5 things you see, 4 you hear, 3 you can touch, 2 you smell, 1 you taste',
                    'category': 'grounding',
                    'duration': '5-10 minutes'
                }
            ],
            EmotionType.SADNESS: [
                {
                    'name': 'Gentle Movement',
                    'description': 'Take a short walk or do light stretching',
                    'category': 'physical',
                    'duration': '10-15 minutes'
                },
                {
                    'name': 'Reach Out',
                    'description': 'Connect with a friend or loved one',
                    'category': 'social',
                    'duration': 'as needed'
                }
            ],
            EmotionType.ANGER: [
                {
                    'name': 'Physical Release',
                    'description': 'Exercise, punch a pillow, or do vigorous activity',
                    'category': 'physical',
                    'duration': '15-20 minutes'
                },
                {
                    'name': 'Journaling',
                    'description': 'Write out your feelings without filter',
                    'category': 'cognitive',
                    'duration': '10-15 minutes'
                }
            ]
        }

        emotion_strategies = strategies.get(emotion, [])

        if not emotion_strategies:
            return None

        import random
        return random.choice(emotion_strategies)

    def provide_crisis_support(self, user_message: str) -> Dict:
        """Provide immediate crisis support and resources"""

        response = {
            'message': """I'm really concerned about what you're sharing. Your safety is the most important thing right now.

Please reach out to someone who can help:
- National Suicide Prevention Lifeline: 988 (US)
- Crisis Text Line: Text HOME to 741741
- International Association for Suicide Prevention: https://www.iasp.info/resources/Crisis_Centres/

I'm here to listen, but I'm not equipped to handle crisis situations. Please connect with a trained professional who can provide the support you need.

Would you like me to help you make a plan to reach out for help?""",
            'crisis_detected': True,
            'resources': [
                {'name': '988 Suicide & Crisis Lifeline', 'contact': '988'},
                {'name': 'Crisis Text Line', 'contact': 'Text HOME to 741741'},
                {'name': 'SAMHSA Helpline', 'contact': '1-800-662-4357'}
            ],
            'immediate_action': 'escalate_to_human'
        }

        return response
```

## Pattern Recognition and Insights

### Emotional Pattern Analyzer

```python
import pandas as pd
import numpy as np
from sklearn.cluster import KMeans
from datetime import datetime, timedelta

class EmotionalPatternAnalyzer:
    """Analyze patterns in emotional data"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def analyze_temporal_patterns(self, mood_entries: List[MoodEntry]) -> Dict:
        """Analyze when certain emotions occur"""

        df = pd.DataFrame([
            {
                'timestamp': entry.timestamp,
                'hour': entry.timestamp.hour,
                'day_of_week': entry.timestamp.weekday(),
                'emotion': entry.primary_emotion.value,
                'mood_level': entry.mood_level.value
            }
            for entry in mood_entries
        ])

        patterns = {}

        # Time of day patterns
        hour_mood = df.groupby('hour')['mood_level'].mean()
        patterns['best_hours'] = hour_mood.nlargest(3).index.tolist()
        patterns['worst_hours'] = hour_mood.nsmallest(3).index.tolist()

        # Day of week patterns
        day_mood = df.groupby('day_of_week')['mood_level'].mean()
        day_names = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
        patterns['best_days'] = [day_names[i] for i in day_mood.nlargest(2).index]
        patterns['worst_days'] = [day_names[i] for i in day_mood.nsmallest(2).index]

        # Emotion frequency
        emotion_freq = df['emotion'].value_counts()
        patterns['most_common_emotions'] = emotion_freq.head(3).to_dict()

        return patterns

    def detect_triggers(self, mood_entries: List[MoodEntry]) -> List[Dict]:
        """Identify emotional triggers"""

        triggers = []

        # Analyze activities associated with negative emotions
        negative_entries = [
            e for e in mood_entries
            if e.mood_level.value <= 2
        ]

        if negative_entries:
            activity_counts = {}
            for entry in negative_entries:
                if entry.activities:
                    for activity in entry.activities:
                        activity_counts[activity] = activity_counts.get(activity, 0) + 1

            # Find frequently occurring activities during low mood
            for activity, count in activity_counts.items():
                if count >= 3:  # Threshold
                    triggers.append({
                        'trigger': activity,
                        'type': 'activity',
                        'occurrences': count,
                        'confidence': min(count / len(negative_entries), 1.0)
                    })

        return triggers

    def analyze_mood_trends(self, mood_entries: List[MoodEntry], days: int = 30) -> Dict:
        """Analyze mood trends over time"""

        df = pd.DataFrame([
            {
                'date': entry.timestamp.date(),
                'mood_level': entry.mood_level.value,
                'sentiment': entry.sentiment_score
            }
            for entry in mood_entries
        ])

        # Calculate daily averages
        daily_mood = df.groupby('date').agg({
            'mood_level': 'mean',
            'sentiment': 'mean'
        }).reset_index()

        # Calculate trend
        if len(daily_mood) >= 7:
            recent_avg = daily_mood.tail(7)['mood_level'].mean()
            older_avg = daily_mood.head(min(7, len(daily_mood) - 7))['mood_level'].mean()

            trend = 'improving' if recent_avg > older_avg + 0.3 else \
                   'declining' if recent_avg < older_avg - 0.3 else 'stable'
        else:
            trend = 'insufficient_data'

        return {
            'trend': trend,
            'current_avg': daily_mood.tail(7)['mood_level'].mean() if len(daily_mood) >= 7 else None,
            'overall_avg': daily_mood['mood_level'].mean(),
            'best_day': str(daily_mood.loc[daily_mood['mood_level'].idxmax(), 'date']) if len(daily_mood) > 0 else None,
            'worst_day': str(daily_mood.loc[daily_mood['mood_level'].idxmin(), 'date']) if len(daily_mood) > 0 else None
        }

    def generate_insights(
        self,
        patterns: Dict,
        trends: Dict,
        triggers: List[Dict]
    ) -> List[str]:
        """Generate actionable insights"""

        insights = []

        # Temporal insights
        if patterns.get('best_hours'):
            best_hours = ', '.join([f"{h}:00" for h in patterns['best_hours']])
            insights.append(
                f"You tend to feel best around {best_hours}. Consider scheduling important tasks during these hours."
            )

        if patterns.get('worst_hours'):
            worst_hours = ', '.join([f"{h}:00" for h in patterns['worst_hours']])
            insights.append(
                f"You often feel lower around {worst_hours}. Plan self-care or breaks during these times."
            )

        # Trend insights
        if trends.get('trend') == 'improving':
            insights.append(
                "Your mood has been improving over the past week. Keep up whatever you've been doing!"
            )
        elif trends.get('trend') == 'declining':
            insights.append(
                "I've noticed your mood declining recently. Would you like to talk about what's been happening?"
            )

        # Trigger insights
        if triggers:
            top_trigger = triggers[0]
            insights.append(
                f"I've noticed '{top_trigger['trigger']}' often coincides with lower moods. Let's explore healthier alternatives."
            )

        return insights
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
import os

app = FastAPI(title="Mood & Emotion Tracking AI")

emotion_detector = EmotionDetector()
empathy_engine = EmpathyEngine(api_key=os.getenv("OPENAI_API_KEY"))

@app.post("/mood/log")
async def log_mood(
    entry: MoodEntryCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Log a mood entry with emotion detection"""

    # Detect emotions from text
    if entry.text_entry:
        emotion_result = emotion_detector.detect_from_text(entry.text_entry)

        # Check for crisis
        crisis_check = emotion_detector.detect_crisis_indicators(entry.text_entry)

        if crisis_check['crisis_detected']:
            crisis_response = empathy_engine.provide_crisis_support(entry.text_entry)
            return crisis_response

        # Create mood entry
        mood_entry = MoodEntry(
            user_id=user_id,
            primary_emotion=EmotionType[emotion_result.primary_emotion.upper()],
            emotion_intensity=emotion_result.intensity,
            emotions_detected=emotion_result.all_emotions,
            mood_level=MoodLevel(entry.mood_level),
            text_entry=entry.text_entry,
            activities=entry.activities,
            location=entry.location,
            sentiment_score=emotion_result.sentiment
        )

        # Generate empathetic response
        ai_response = empathy_engine.respond_to_emotion(
            entry.text_entry,
            emotion_result
        )

        mood_entry.ai_response = ai_response

        db.add(mood_entry)
        db.commit()

        # Suggest coping strategy if needed
        coping_strategy = None
        if mood_entry.mood_level.value <= 2:
            coping_strategy = empathy_engine.suggest_coping_strategy(
                mood_entry.primary_emotion
            )

        return {
            'mood_entry': mood_entry,
            'ai_response': ai_response,
            'emotion_detected': emotion_result.primary_emotion,
            'coping_strategy': coping_strategy
        }

@app.get("/mood/patterns")
async def get_emotional_patterns(
    user_id: str,
    days: int = 30,
    db: Session = Depends(get_db)
):
    """Get emotional patterns and insights"""

    # Get mood entries
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    mood_entries = db.query(MoodEntry).filter(
        MoodEntry.user_id == user_id,
        MoodEntry.timestamp >= cutoff_date
    ).all()

    if not mood_entries:
        return {'message': 'No data available yet. Keep logging your moods!'}

    # Analyze patterns
    analyzer = EmotionalPatternAnalyzer(user_id)
    patterns = analyzer.analyze_temporal_patterns(mood_entries)
    trends = analyzer.analyze_mood_trends(mood_entries, days)
    triggers = analyzer.detect_triggers(mood_entries)

    # Generate insights
    insights = analyzer.generate_insights(patterns, trends, triggers)

    return {
        'patterns': patterns,
        'trends': trends,
        'triggers': triggers,
        'insights': insights
    }

@app.post("/mood/chat")
async def emotional_support_chat(
    message: str,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Chat with empathetic AI"""

    # Detect emotion
    emotion_result = emotion_detector.detect_from_text(message)

    # Get recent mood context
    recent_moods = db.query(MoodEntry).filter(
        MoodEntry.user_id == user_id
    ).order_by(MoodEntry.timestamp.desc()).limit(5).all()

    # Check for emotional shift
    shift = emotion_detector.analyze_emotional_shift(recent_moods, message)

    # Generate response
    response = empathy_engine.respond_to_emotion(
        message,
        emotion_result
    )

    return {
        'response': response,
        'emotion_detected': emotion_result.primary_emotion,
        'emotional_shift': shift if shift['shift_detected'] else None
    }
```

## Visualization Dashboard

```javascript
// MoodDashboard.jsx
import React, { useState, useEffect } from 'react';
import { LineChart, Line, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts';

const MoodDashboard = ({ userId }) => {
  const [moodData, setMoodData] = useState([]);
  const [patterns, setPatterns] = useState(null);
  const [insights, setInsights] = useState([]);

  const moodEmojis = {
    1: '😢',
    2: '😔',
    3: '😐',
    4: '🙂',
    5: '😄'
  };

  const logMood = async (level, text) => {
    const response = await fetch('/mood/log', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        mood_level: level,
        text_entry: text,
        user_id: userId
      })
    });

    const data = await response.json();
    // Show AI response
    alert(data.ai_response);
  };

  return (
    <div className="mood-dashboard">
      <h1>How are you feeling?</h1>

      {/* Quick Mood Log */}
      <div className="mood-selector">
        {[1, 2, 3, 4, 5].map(level => (
          <button
            key={level}
            className="mood-button"
            onClick={() => setSelectedMood(level)}
          >
            <span className="emoji">{moodEmojis[level]}</span>
          </button>
        ))}
      </div>

      {/* Mood Timeline */}
      <div className="mood-timeline">
        <h2>Your Emotional Journey</h2>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={moodData}>
            <XAxis dataKey="date" />
            <YAxis domain={[1, 5]} />
            <Tooltip />
            <Line type="monotone" dataKey="mood" stroke="#8884d8" />
          </LineChart>
        </ResponsiveContainer>
      </div>

      {/* Insights */}
      <div className="insights">
        <h2>Insights</h2>
        {insights.map((insight, idx) => (
          <div key={idx} className="insight-card">
            {insight}
          </div>
        ))}
      </div>
    </div>
  );
};

export default MoodDashboard;
```

## Bonus Challenges

### Challenge 1: Voice Emotion Detection
Detect emotions from voice tone and prosody.

### Challenge 2: Predictive Mood Forecasting
Predict likely future mood based on patterns.

### Challenge 3: Integration with Wearables
Track physiological data (heart rate, sleep) for better insights.

### Challenge 4: Group Therapy Sessions
Facilitate group support conversations.

### Challenge 5: Therapist Dashboard
Provide anonymized insights to therapists.

## Resources

### Papers
- "Emotion Recognition in Conversations" (Poria et al., 2019)
- "Computational Empathy" (Rashkin et al., 2018)

### Documentation
- [Transformers Emotion Models](https://huggingface.co/models?pipeline_tag=text-classification&sort=downloads&search=emotion)
- [Crisis Resources](https://988lifeline.org/)

### Ethics
- Never replace professional mental health support
- Clear disclaimers about limitations
- Crisis detection and escalation protocols
- Data privacy and security

## Success Criteria

### Features
- [ ] Accurate emotion detection (>80%)
- [ ] Empathetic, contextual responses
- [ ] Pattern recognition and insights
- [ ] Crisis detection and resources
- [ ] Coping strategy suggestions
- [ ] Progress visualization

### Safety
- [ ] Crisis detection system
- [ ] Resource referrals
- [ ] Clear limitations stated
- [ ] Privacy protections
- [ ] Ethical guidelines followed

This system provides emotional support while maintaining appropriate boundaries and safety measures!
