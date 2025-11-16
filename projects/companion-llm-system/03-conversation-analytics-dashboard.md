# Project 03: Conversation Analytics Dashboard for User Pattern Tracking

## Overview
Build a comprehensive analytics dashboard that tracks, analyzes, and visualizes conversation patterns, emotional trends, topics of interest, and communication habits. This system provides insights into user behavior, helping both the user understand themselves better and the AI companion to provide more personalized support.

## Learning Objectives
- Design and implement conversation analytics pipeline
- Build time-series analysis for behavioral patterns
- Create visualizations for complex conversational data
- Implement NLP techniques for topic modeling and sentiment analysis
- Develop real-time streaming analytics
- Design privacy-conscious analytics systems

## Difficulty Level
**Intermediate** - Requires knowledge of data analysis, NLP, and web development

## Technical Stack

### Core Technologies
- **Backend**: Python with FastAPI
- **Analytics**: pandas, numpy, scikit-learn
- **NLP**: spaCy, NLTK, transformers
- **Database**: PostgreSQL (time-series data), ClickHouse (analytics)
- **Caching**: Redis for real-time metrics
- **Frontend**: React with Recharts/D3.js
- **ML**: Topic modeling (LDA, BERTopic), sentiment analysis

### Libraries
```python
# requirements.txt
fastapi==0.109.2
uvicorn==0.27.1
pandas==2.1.4
numpy==1.26.3
scikit-learn==1.4.0
spacy==3.7.2
nltk==3.8.1
transformers==4.37.2
bertopic==0.16.0
sqlalchemy==2.0.25
asyncpg==0.29.0
redis==5.0.1
plotly==5.18.0
streamlit==1.31.0  # Alternative for quick dashboard
psycopg2-binary==2.9.9
pydantic==2.6.1
python-dateutil==2.8.2
```

## Data Model and Architecture

### Analytics Data Schema

```python
from sqlalchemy import Column, String, DateTime, Float, Integer, JSON, Text
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
from typing import List, Dict, Optional
from pydantic import BaseModel

Base = declarative_base()

class ConversationLog(Base):
    """Store individual conversation messages"""
    __tablename__ = "conversation_logs"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    message_id = Column(String, unique=True)
    role = Column(String)  # user or assistant
    content = Column(Text)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)
    session_id = Column(String, index=True)
    metadata = Column(JSON)

class ConversationMetrics(Base):
    """Aggregated conversation metrics"""
    __tablename__ = "conversation_metrics"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    date = Column(DateTime, index=True)
    message_count = Column(Integer)
    avg_message_length = Column(Float)
    avg_response_time = Column(Float)  # seconds
    sentiment_score = Column(Float)  # -1 to 1
    topics = Column(JSON)  # List of topics discussed
    emotion_distribution = Column(JSON)  # Distribution of emotions

class TopicAnalysis(Base):
    """Track topics over time"""
    __tablename__ = "topic_analysis"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    topic_id = Column(String)
    topic_name = Column(String)
    keywords = Column(JSON)
    frequency = Column(Integer)
    timestamp = Column(DateTime, index=True)
    sentiment = Column(Float)

class UserInsights(Base):
    """Store generated insights about user"""
    __tablename__ = "user_insights"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    insight_type = Column(String)  # pattern, trend, anomaly
    insight_text = Column(Text)
    confidence_score = Column(Float)
    generated_at = Column(DateTime, default=datetime.utcnow)
    metadata = Column(JSON)
```

### Analytics Pipeline Architecture

```python
from typing import List, Dict
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

class AnalyticsPipeline:
    """Process conversation data through analytics pipeline"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def process_message(self, message: Dict) -> Dict:
        """Process a single message and extract analytics"""
        analytics = {
            'user_id': self.user_id,
            'message_id': message['message_id'],
            'timestamp': message['timestamp'],
            'word_count': len(message['content'].split()),
            'char_count': len(message['content']),
            'sentiment': self.analyze_sentiment(message['content']),
            'emotions': self.detect_emotions(message['content']),
            'topics': self.extract_topics(message['content']),
            'entities': self.extract_entities(message['content']),
            'complexity_score': self.calculate_complexity(message['content'])
        }

        return analytics

    def analyze_sentiment(self, text: str) -> float:
        """Analyze sentiment of text"""
        from transformers import pipeline

        sentiment_analyzer = pipeline(
            "sentiment-analysis",
            model="cardiffnlp/twitter-roberta-base-sentiment"
        )

        result = sentiment_analyzer(text[:512])[0]

        # Convert to -1 to 1 scale
        if result['label'] == 'NEGATIVE':
            return -result['score']
        elif result['label'] == 'POSITIVE':
            return result['score']
        else:
            return 0.0

    def detect_emotions(self, text: str) -> Dict[str, float]:
        """Detect emotions in text"""
        from transformers import pipeline

        emotion_classifier = pipeline(
            "text-classification",
            model="j-hartmann/emotion-english-distilroberta-base",
            top_k=None
        )

        emotions = emotion_classifier(text[:512])[0]

        return {e['label']: e['score'] for e in emotions}

    def extract_topics(self, text: str) -> List[str]:
        """Extract topics using NLP"""
        import spacy

        nlp = spacy.load("en_core_web_sm")
        doc = nlp(text)

        # Extract noun phrases as topic indicators
        topics = [chunk.text.lower() for chunk in doc.noun_chunks]

        return list(set(topics))[:5]  # Top 5 unique topics

    def extract_entities(self, text: str) -> Dict[str, List[str]]:
        """Extract named entities"""
        import spacy

        nlp = spacy.load("en_core_web_sm")
        doc = nlp(text)

        entities = {}
        for ent in doc.ents:
            if ent.label_ not in entities:
                entities[ent.label_] = []
            entities[ent.label_].append(ent.text)

        return entities

    def calculate_complexity(self, text: str) -> float:
        """Calculate text complexity score"""
        words = text.split()
        sentences = text.split('.')

        if not words or not sentences:
            return 0.0

        # Simple complexity metrics
        avg_word_length = sum(len(w) for w in words) / len(words)
        avg_sentence_length = len(words) / len(sentences)

        # Normalize to 0-1 scale
        complexity = (avg_word_length / 10 + avg_sentence_length / 20) / 2

        return min(complexity, 1.0)
```

## Pattern Detection and Analysis

### Conversation Pattern Detector

```python
from sklearn.cluster import DBSCAN
from collections import Counter
import pandas as pd

class PatternDetector:
    """Detect patterns in conversation data"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def detect_temporal_patterns(self, conversations: pd.DataFrame) -> Dict:
        """Detect when user typically converses"""
        conversations['hour'] = pd.to_datetime(conversations['timestamp']).dt.hour
        conversations['day_of_week'] = pd.to_datetime(conversations['timestamp']).dt.dayofweek

        # Find peak hours
        hour_distribution = conversations['hour'].value_counts()
        peak_hours = hour_distribution.nlargest(3).index.tolist()

        # Find peak days
        day_distribution = conversations['day_of_week'].value_counts()
        peak_days = day_distribution.nlargest(3).index.tolist()

        day_names = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
        peak_day_names = [day_names[d] for d in peak_days]

        return {
            'peak_hours': peak_hours,
            'peak_days': peak_day_names,
            'hour_distribution': hour_distribution.to_dict(),
            'day_distribution': day_distribution.to_dict()
        }

    def detect_topic_trends(self, conversations: pd.DataFrame) -> Dict:
        """Detect trending topics over time"""
        # Extract topics from each conversation
        from bertopic import BERTopic

        texts = conversations['content'].tolist()
        timestamps = conversations['timestamp'].tolist()

        # Create topic model
        topic_model = BERTopic(min_topic_size=5)
        topics, probs = topic_model.fit_transform(texts)

        # Track topics over time
        topics_over_time = topic_model.topics_over_time(
            texts,
            timestamps,
            nr_bins=20
        )

        return {
            'topics': topic_model.get_topic_info(),
            'topics_over_time': topics_over_time,
            'representative_docs': topic_model.get_representative_docs()
        }

    def detect_conversation_clusters(self, conversations: pd.DataFrame) -> Dict:
        """Cluster similar conversations"""
        from sentence_transformers import SentenceTransformer

        # Encode conversations
        model = SentenceTransformer('all-MiniLM-L6-v2')
        embeddings = model.encode(conversations['content'].tolist())

        # Cluster using DBSCAN
        clustering = DBSCAN(eps=0.5, min_samples=2).fit(embeddings)

        conversations['cluster'] = clustering.labels_

        # Analyze clusters
        cluster_summary = {}
        for cluster_id in set(clustering.labels_):
            if cluster_id == -1:  # Noise
                continue

            cluster_convs = conversations[conversations['cluster'] == cluster_id]
            cluster_summary[f"cluster_{cluster_id}"] = {
                'size': len(cluster_convs),
                'sample_conversations': cluster_convs['content'].head(3).tolist(),
                'avg_sentiment': cluster_convs['sentiment'].mean() if 'sentiment' in cluster_convs else None
            }

        return cluster_summary

    def detect_behavioral_anomalies(self, metrics: pd.DataFrame) -> List[Dict]:
        """Detect unusual conversation patterns"""
        anomalies = []

        # Check for unusual message frequency
        daily_counts = metrics.groupby(metrics['date'].dt.date)['message_count'].sum()
        mean_count = daily_counts.mean()
        std_count = daily_counts.std()

        unusual_days = daily_counts[
            (daily_counts > mean_count + 2 * std_count) |
            (daily_counts < mean_count - 2 * std_count)
        ]

        for date, count in unusual_days.items():
            anomalies.append({
                'type': 'message_frequency',
                'date': str(date),
                'value': count,
                'expected': mean_count,
                'severity': abs(count - mean_count) / std_count
            })

        # Check for sentiment anomalies
        if 'sentiment_score' in metrics.columns:
            unusual_sentiment = metrics[
                abs(metrics['sentiment_score']) > 0.8
            ]

            for _, row in unusual_sentiment.iterrows():
                anomalies.append({
                    'type': 'unusual_sentiment',
                    'date': str(row['date']),
                    'value': row['sentiment_score'],
                    'severity': abs(row['sentiment_score'])
                })

        return anomalies
```

### Insight Generator

```python
from typing import List, Dict
import openai

class InsightGenerator:
    """Generate human-readable insights from analytics"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def generate_weekly_summary(
        self,
        user_id: str,
        metrics: Dict
    ) -> str:
        """Generate weekly conversation summary"""

        prompt = f"""Analyze this week's conversation data and provide insights:

Message Count: {metrics['message_count']}
Average Message Length: {metrics['avg_message_length']} words
Top Topics: {', '.join(metrics['top_topics'][:5])}
Sentiment Distribution: {metrics['sentiment_distribution']}
Peak Hours: {metrics['peak_hours']}

Generate a friendly, insightful summary highlighting:
1. Communication patterns
2. Emotional trends
3. Main topics of interest
4. Any notable changes or patterns

Keep it conversational and actionable."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[
                {"role": "system", "content": "You are an insightful data analyst who explains patterns in a friendly, accessible way."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7
        )

        return response.choices[0].message.content

    def generate_trend_insights(self, trends: pd.DataFrame) -> List[str]:
        """Generate insights from trend data"""
        insights = []

        # Check for increasing/decreasing trends
        if len(trends) >= 7:
            recent_avg = trends.tail(3)['message_count'].mean()
            older_avg = trends.head(3)['message_count'].mean()

            if recent_avg > older_avg * 1.2:
                insights.append(
                    f"Your conversation frequency increased by {((recent_avg/older_avg - 1) * 100):.1f}% recently"
                )
            elif recent_avg < older_avg * 0.8:
                insights.append(
                    f"Your conversation frequency decreased by {((1 - recent_avg/older_avg) * 100):.1f}% recently"
                )

        return insights

    def generate_personalized_recommendations(
        self,
        user_data: Dict
    ) -> List[str]:
        """Generate personalized recommendations"""
        recommendations = []

        # Based on conversation patterns
        if user_data.get('avg_response_time', 0) > 60:
            recommendations.append(
                "Consider shorter, more focused questions to get quicker responses"
            )

        # Based on topics
        top_topics = user_data.get('top_topics', [])
        if 'stress' in top_topics or 'anxiety' in top_topics:
            recommendations.append(
                "I notice you've been discussing stress. Would you like to explore some relaxation techniques?"
            )

        # Based on time patterns
        peak_hours = user_data.get('peak_hours', [])
        if any(h >= 23 or h <= 5 for h in peak_hours):
            recommendations.append(
                "You often chat late at night. Remember to maintain good sleep habits!"
            )

        return recommendations
```

## Dashboard Implementation

### FastAPI Backend

```python
from fastapi import FastAPI, Depends, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware
from datetime import datetime, timedelta
from typing import Optional, List
import pandas as pd

app = FastAPI(title="Conversation Analytics API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

class AnalyticsAPI:
    """API endpoints for analytics"""

    def __init__(self):
        self.pipeline = AnalyticsPipeline("user_id")
        self.pattern_detector = PatternDetector("user_id")

@app.get("/analytics/overview")
async def get_overview(
    user_id: str,
    days: int = Query(7, ge=1, le=90)
):
    """Get analytics overview for specified time period"""
    end_date = datetime.utcnow()
    start_date = end_date - timedelta(days=days)

    # Query database
    # conversations = query_conversations(user_id, start_date, end_date)

    # For demo, create sample data
    conversations = pd.DataFrame({
        'timestamp': pd.date_range(start_date, end_date, periods=100),
        'content': ['Sample message'] * 100,
        'role': ['user'] * 50 + ['assistant'] * 50,
        'sentiment': np.random.uniform(-0.5, 0.8, 100)
    })

    # Calculate metrics
    total_messages = len(conversations)
    user_messages = len(conversations[conversations['role'] == 'user'])
    avg_sentiment = conversations['sentiment'].mean()

    # Detect patterns
    patterns = PatternDetector(user_id).detect_temporal_patterns(conversations)

    return {
        'period': f'{days} days',
        'total_messages': total_messages,
        'user_messages': user_messages,
        'avg_sentiment': round(avg_sentiment, 2),
        'patterns': patterns,
        'start_date': start_date.isoformat(),
        'end_date': end_date.isoformat()
    }

@app.get("/analytics/sentiment-trend")
async def get_sentiment_trend(
    user_id: str,
    days: int = Query(30, ge=1, le=365)
):
    """Get sentiment trend over time"""
    # Query and process data
    # For demo:
    dates = pd.date_range(
        datetime.utcnow() - timedelta(days=days),
        datetime.utcnow(),
        freq='D'
    )

    sentiment_data = [
        {
            'date': date.isoformat(),
            'sentiment': round(np.random.uniform(-0.3, 0.7), 2),
            'message_count': np.random.randint(5, 30)
        }
        for date in dates
    ]

    return {
        'data': sentiment_data,
        'summary': {
            'avg_sentiment': round(np.mean([d['sentiment'] for d in sentiment_data]), 2),
            'trend': 'positive'  # Calculate actual trend
        }
    }

@app.get("/analytics/topics")
async def get_top_topics(
    user_id: str,
    days: int = Query(30, ge=1, le=365),
    limit: int = Query(10, ge=1, le=50)
):
    """Get top discussion topics"""
    # In production, query from TopicAnalysis table

    topics = [
        {'topic': 'AI and Technology', 'count': 45, 'sentiment': 0.7},
        {'topic': 'Personal Goals', 'count': 38, 'sentiment': 0.5},
        {'topic': 'Health and Wellness', 'count': 32, 'sentiment': 0.3},
        {'topic': 'Work Projects', 'count': 28, 'sentiment': 0.2},
        {'topic': 'Learning and Education', 'count': 25, 'sentiment': 0.8},
    ]

    return {
        'topics': topics[:limit],
        'period': f'{days} days'
    }

@app.get("/analytics/insights")
async def get_insights(user_id: str):
    """Get AI-generated insights"""
    # Generate insights based on recent data
    insights = [
        {
            'type': 'pattern',
            'title': 'Increased Evening Conversations',
            'description': 'You chat 40% more between 8-10 PM this week',
            'confidence': 0.85
        },
        {
            'type': 'trend',
            'title': 'Growing Interest in AI',
            'description': 'AI-related topics increased by 60% this month',
            'confidence': 0.92
        },
        {
            'type': 'recommendation',
            'title': 'Balance Discussion Topics',
            'description': 'Consider exploring health and wellness topics more',
            'confidence': 0.70
        }
    ]

    return {'insights': insights}

@app.get("/analytics/conversation-stats")
async def get_conversation_stats(
    user_id: str,
    days: int = Query(7, ge=1, le=90)
):
    """Get detailed conversation statistics"""
    stats = {
        'total_conversations': 156,
        'avg_conversation_length': 12.5,  # messages
        'avg_message_length': 45.3,  # words
        'response_time_avg': 3.2,  # seconds
        'top_emotions': [
            {'emotion': 'joy', 'percentage': 35},
            {'emotion': 'neutral', 'percentage': 30},
            {'emotion': 'curiosity', 'percentage': 20},
            {'emotion': 'concern', 'percentage': 10},
            {'emotion': 'frustration', 'percentage': 5},
        ],
        'conversation_complexity': 0.68,  # 0-1 scale
        'unique_topics_discussed': 47
    }

    return stats

@app.get("/analytics/export")
async def export_analytics(
    user_id: str,
    format: str = Query("json", regex="^(json|csv)$"),
    days: int = Query(30, ge=1, le=365)
):
    """Export analytics data"""
    # Generate export file
    # Return download link or file

    return {
        'export_url': f'/downloads/{user_id}_analytics_{days}days.{format}',
        'generated_at': datetime.utcnow().isoformat()
    }
```

### React Dashboard Frontend

```javascript
// Dashboard.jsx
import React, { useState, useEffect } from 'react';
import {
  LineChart, Line, BarChart, Bar, PieChart, Pie,
  XAxis, YAxis, CartesianGrid, Tooltip, Legend,
  ResponsiveContainer, Cell
} from 'recharts';

const Dashboard = ({ userId }) => {
  const [overview, setOverview] = useState(null);
  const [sentimentTrend, setSentimentTrend] = useState([]);
  const [topics, setTopics] = useState([]);
  const [insights, setInsights] = useState([]);
  const [timeRange, setTimeRange] = useState(7);

  useEffect(() => {
    fetchAnalytics();
  }, [userId, timeRange]);

  const fetchAnalytics = async () => {
    // Fetch overview
    const overviewRes = await fetch(
      `http://localhost:8000/analytics/overview?user_id=${userId}&days=${timeRange}`
    );
    setOverview(await overviewRes.json());

    // Fetch sentiment trend
    const sentimentRes = await fetch(
      `http://localhost:8000/analytics/sentiment-trend?user_id=${userId}&days=${timeRange}`
    );
    const sentimentData = await sentimentRes.json();
    setSentimentTrend(sentimentData.data);

    // Fetch topics
    const topicsRes = await fetch(
      `http://localhost:8000/analytics/topics?user_id=${userId}&days=${timeRange}`
    );
    const topicsData = await topicsRes.json();
    setTopics(topicsData.topics);

    // Fetch insights
    const insightsRes = await fetch(
      `http://localhost:8000/analytics/insights?user_id=${userId}`
    );
    const insightsData = await insightsRes.json();
    setInsights(insightsData.insights);
  };

  return (
    <div className="dashboard">
      <h1>Conversation Analytics</h1>

      {/* Time Range Selector */}
      <div className="time-range-selector">
        <button onClick={() => setTimeRange(7)}>7 Days</button>
        <button onClick={() => setTimeRange(30)}>30 Days</button>
        <button onClick={() => setTimeRange(90)}>90 Days</button>
      </div>

      {/* Overview Cards */}
      {overview && (
        <div className="overview-cards">
          <div className="card">
            <h3>Total Messages</h3>
            <p className="metric">{overview.total_messages}</p>
          </div>
          <div className="card">
            <h3>Your Messages</h3>
            <p className="metric">{overview.user_messages}</p>
          </div>
          <div className="card">
            <h3>Average Sentiment</h3>
            <p className="metric">
              {overview.avg_sentiment > 0 ? '😊' : '😐'}
              {overview.avg_sentiment.toFixed(2)}
            </p>
          </div>
        </div>
      )}

      {/* Sentiment Trend Chart */}
      <div className="chart-container">
        <h2>Sentiment Over Time</h2>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={sentimentTrend}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis
              dataKey="date"
              tickFormatter={(date) => new Date(date).toLocaleDateString()}
            />
            <YAxis domain={[-1, 1]} />
            <Tooltip />
            <Legend />
            <Line
              type="monotone"
              dataKey="sentiment"
              stroke="#8884d8"
              name="Sentiment Score"
            />
          </LineChart>
        </ResponsiveContainer>
      </div>

      {/* Top Topics Chart */}
      <div className="chart-container">
        <h2>Top Discussion Topics</h2>
        <ResponsiveContainer width="100%" height={300}>
          <BarChart data={topics}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="topic" />
            <YAxis />
            <Tooltip />
            <Legend />
            <Bar dataKey="count" fill="#82ca9d" />
          </BarChart>
        </ResponsiveContainer>
      </div>

      {/* Insights Section */}
      <div className="insights-section">
        <h2>AI-Generated Insights</h2>
        {insights.map((insight, idx) => (
          <div key={idx} className={`insight-card ${insight.type}`}>
            <h3>{insight.title}</h3>
            <p>{insight.description}</p>
            <span className="confidence">
              Confidence: {(insight.confidence * 100).toFixed(0)}%
            </span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default Dashboard;
```

```css
/* Dashboard.css */
.dashboard {
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
}

.overview-cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin: 30px 0;
}

.card {
  background: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  text-align: center;
}

.card h3 {
  margin: 0;
  font-size: 14px;
  color: #666;
  text-transform: uppercase;
}

.card .metric {
  font-size: 32px;
  font-weight: bold;
  margin: 10px 0 0 0;
  color: #333;
}

.chart-container {
  background: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  margin: 20px 0;
}

.insights-section {
  margin-top: 30px;
}

.insight-card {
  background: white;
  padding: 20px;
  border-radius: 10px;
  margin: 15px 0;
  border-left: 4px solid #4CAF50;
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

.insight-card.pattern {
  border-left-color: #2196F3;
}

.insight-card.trend {
  border-left-color: #FF9800;
}

.insight-card.recommendation {
  border-left-color: #9C27B0;
}

.insight-card h3 {
  margin: 0 0 10px 0;
}

.insight-card .confidence {
  font-size: 12px;
  color: #888;
}

.time-range-selector {
  display: flex;
  gap: 10px;
  margin: 20px 0;
}

.time-range-selector button {
  padding: 10px 20px;
  border: 2px solid #4CAF50;
  background: white;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.3s;
}

.time-range-selector button:hover {
  background: #4CAF50;
  color: white;
}
```

## Real-Time Analytics

```python
import asyncio
from datetime import datetime
import redis
import json

class RealTimeAnalytics:
    """Real-time conversation analytics"""

    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=1)

    async def track_message(self, user_id: str, message: Dict):
        """Track message in real-time"""
        # Update counters
        today = datetime.utcnow().date().isoformat()
        hour = datetime.utcnow().hour

        # Increment message count
        self.redis_client.incr(f"user:{user_id}:messages:{today}")
        self.redis_client.incr(f"user:{user_id}:messages:hour:{hour}")

        # Update sentiment moving average
        sentiment = message.get('sentiment', 0)
        self.redis_client.zadd(
            f"user:{user_id}:sentiment:recent",
            {json.dumps(message): datetime.utcnow().timestamp()}
        )

        # Keep only recent 100 messages
        self.redis_client.zremrangebyrank(
            f"user:{user_id}:sentiment:recent",
            0, -101
        )

        # Publish to subscribers
        await self.publish_update(user_id, {
            'type': 'new_message',
            'timestamp': datetime.utcnow().isoformat(),
            'metrics': await self.get_current_metrics(user_id)
        })

    async def get_current_metrics(self, user_id: str) -> Dict:
        """Get current real-time metrics"""
        today = datetime.utcnow().date().isoformat()

        return {
            'messages_today': int(self.redis_client.get(f"user:{user_id}:messages:{today}") or 0),
            'current_hour_messages': int(self.redis_client.get(
                f"user:{user_id}:messages:hour:{datetime.utcnow().hour}"
            ) or 0),
            'recent_avg_sentiment': await self.get_recent_sentiment(user_id)
        }

    async def get_recent_sentiment(self, user_id: str) -> float:
        """Calculate recent sentiment average"""
        recent = self.redis_client.zrange(
            f"user:{user_id}:sentiment:recent",
            0, -1
        )

        if not recent:
            return 0.0

        sentiments = [json.loads(msg.decode())['sentiment'] for msg in recent]
        return sum(sentiments) / len(sentiments)

    async def publish_update(self, user_id: str, update: Dict):
        """Publish update to subscribers"""
        self.redis_client.publish(
            f"analytics:{user_id}",
            json.dumps(update)
        )
```

## Privacy and Data Handling

```python
class PrivacyControls:
    """Handle privacy for analytics"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def anonymize_data(self, data: pd.DataFrame) -> pd.DataFrame:
        """Anonymize sensitive data"""
        # Remove PII
        import re

        def remove_pii(text):
            # Remove emails
            text = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL]', text)
            # Remove phone numbers
            text = re.sub(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE]', text)
            # Remove names (use NER in production)
            return text

        data['content'] = data['content'].apply(remove_pii)
        return data

    def get_retention_policy(self) -> Dict:
        """Get data retention policy"""
        return {
            'raw_messages': '90 days',
            'aggregated_analytics': '2 years',
            'anonymized_patterns': 'indefinite'
        }

    def export_user_analytics(self) -> Dict:
        """Export all analytics data"""
        # GDPR-compliant data export
        return {
            'user_id': self.user_id,
            'export_date': datetime.utcnow().isoformat(),
            'conversations': [],  # All conversations
            'analytics': [],  # All analytics
            'insights': []  # All insights
        }
```

## Bonus Challenges

### Challenge 1: Predictive Analytics
Predict future conversation patterns and needs.

```python
from sklearn.ensemble import RandomForestRegressor

class PredictiveAnalytics:
    def predict_next_conversation_time(self, history: pd.DataFrame):
        """Predict when user will likely chat next"""
        # Train model on historical patterns
        pass
```

### Challenge 2: Comparative Analytics
Compare with aggregate anonymous patterns.

### Challenge 3: Voice Analytics
Add voice conversation analytics (tone, pitch, speed).

### Challenge 4: Multi-User Analytics
Analytics for group conversations.

### Challenge 5: Mobile Dashboard
Build native mobile analytics app.

## Resources

### Documentation
- [Recharts Documentation](https://recharts.org/)
- [BERTopic Guide](https://maartengr.github.io/BERTopic/)
- [spaCy NLP](https://spacy.io/)

### Papers
- "Topic Modeling in Embedding Spaces" (Angelov, 2020)
- "Sentiment Analysis in Social Media" (Liu, 2012)

## Success Criteria

### Technical Metrics
- [ ] Real-time updates within 1 second
- [ ] Dashboard loads in < 3 seconds
- [ ] Analytics accuracy > 85%
- [ ] Support 1000+ messages analysis

### Features
- [ ] Sentiment trend visualization
- [ ] Topic detection and tracking
- [ ] Pattern recognition
- [ ] AI-generated insights
- [ ] Export functionality
- [ ] Privacy controls

### User Experience
- [ ] Intuitive visualizations
- [ ] Actionable insights
- [ ] Mobile-responsive design
- [ ] Customizable time ranges
- [ ] Real-time updates

This dashboard provides comprehensive insights into conversation patterns and helps users understand their interaction patterns better!
