# Project 07: Journaling AI with Reflective Questions

## Overview
Build an AI companion that facilitates deep reflection and self-discovery through intelligent journaling prompts, Socratic questioning, and thoughtful analysis of entries. The system helps users develop self-awareness, process emotions, and track personal growth through structured and free-form journaling.

## Learning Objectives
- Design effective journaling prompts and frameworks
- Implement Socratic questioning techniques
- Build semantic analysis for journal entries
- Create theme and pattern detection across entries
- Develop progress tracking for personal growth
- Generate personalized insights from journal data

## Difficulty Level
**Intermediate** - Requires NLP, conversation design, and therapeutic questioning knowledge

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL + Vector DB (Pinecone/Chroma)
- **NLP**: spaCy, transformers for analysis
- **Frontend**: React with rich text editor
- **Storage**: S3 or local storage for media
- **Search**: Elasticsearch for journal search

### Libraries
```python
# requirements.txt
openai==1.12.0
anthropic==0.18.1
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
chromadb==0.4.22
sentence-transformers==2.3.1
spacy==3.7.2
transformers==4.37.2
pydantic==2.6.1
python-dateutil==2.8.2
pandas==2.1.4
numpy==1.26.3
```

## Data Model and Architecture

### Journaling Schema

```python
from sqlalchemy import Column, String, DateTime, Integer, JSON, Text, Boolean, Float, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class JournalType(enum.Enum):
    FREE_FORM = "free_form"
    PROMPTED = "prompted"
    GRATITUDE = "gratitude"
    MORNING_PAGES = "morning_pages"
    EVENING_REFLECTION = "evening_reflection"
    GOAL_REFLECTION = "goal_reflection"
    EMOTIONAL_PROCESSING = "emotional_processing"

class JournalEntry(Base):
    """Core journal entry"""
    __tablename__ = "journal_entries"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)

    # Content
    title = Column(String, nullable=True)
    content = Column(Text, nullable=False)
    journal_type = Column(String)  # JournalType enum

    # Metadata
    word_count = Column(Integer)
    writing_duration = Column(Integer)  # seconds
    mood = Column(String, nullable=True)
    tags = Column(JSON)  # ["growth", "work", "relationships"]

    # AI interaction
    prompt_used = Column(String, nullable=True)
    ai_questions = Column(JSON, nullable=True)  # Questions asked during writing
    ai_insights = Column(Text, nullable=True)  # AI-generated insights

    # Analysis
    themes = Column(JSON)  # Detected themes
    sentiment_score = Column(Float)
    key_entities = Column(JSON)  # People, places, concepts mentioned
    related_entries = Column(JSON)  # IDs of related entries

    # Privacy
    is_private = Column(Boolean, default=True)
    encrypted = Column(Boolean, default=False)

class JournalPrompt(Base):
    """Journaling prompts"""
    __tablename__ = "journal_prompts"

    id = Column(Integer, primary_key=True)
    prompt_text = Column(Text, nullable=False)
    prompt_type = Column(String)  # "reflective", "gratitude", "creative", "therapeutic"
    category = Column(String)  # "emotions", "relationships", "growth", "values"
    difficulty_level = Column(Integer)  # 1-5
    follow_up_questions = Column(JSON)
    created_at = Column(DateTime, default=datetime.utcnow)
    times_used = Column(Integer, default=0)
    avg_rating = Column(Float, default=0.0)

class JournalTheme(Base):
    """Recurring themes across entries"""
    __tablename__ = "journal_themes"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    theme_name = Column(String)
    description = Column(Text)
    first_appeared = Column(DateTime)
    last_appeared = Column(DateTime)
    frequency = Column(Integer)
    related_entries = Column(JSON)  # Entry IDs

class ReflectionSession(Base):
    """Guided reflection sessions"""
    __tablename__ = "reflection_sessions"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    session_date = Column(DateTime, default=datetime.utcnow)
    session_type = Column(String)  # "weekly", "monthly", "quarterly"

    questions_asked = Column(JSON)
    responses = Column(JSON)
    insights_generated = Column(Text)
    action_items = Column(JSON)

    completed = Column(Boolean, default=False)

# Pydantic models
class JournalEntryCreate(BaseModel):
    content: str
    title: Optional[str] = None
    journal_type: str = "free_form"
    mood: Optional[str] = None
    tags: Optional[List[str]] = None
    prompt_id: Optional[int] = None

class JournalAnalysis(BaseModel):
    themes: List[str]
    sentiment: float
    entities: Dict[str, List[str]]
    word_count: int
    insights: str
```

## Prompt Generation System

### Intelligent Prompt Generator

```python
import openai
from datetime import datetime, timedelta
from typing import List, Dict

class JournalPromptGenerator:
    """Generate personalized journaling prompts"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def generate_daily_prompt(
        self,
        user_context: Dict = None,
        recent_entries: List[JournalEntry] = None
    ) -> Dict:
        """Generate personalized daily prompt"""

        # Analyze recent entries for context
        if recent_entries:
            themes = self.extract_recent_themes(recent_entries)
            sentiment_trend = self.analyze_sentiment_trend(recent_entries)
        else:
            themes = []
            sentiment_trend = "neutral"

        # Generate contextual prompt
        prompt_request = f"""Generate a thoughtful journaling prompt for today.

Recent themes: {', '.join(themes) if themes else 'None yet'}
Recent sentiment: {sentiment_trend}
Day of week: {datetime.now().strftime('%A')}

Create a prompt that:
1. Encourages self-reflection
2. Is open-ended
3. Relates to recent themes (if any)
4. Is thought-provoking but not overwhelming

Include 2-3 follow-up questions to deepen exploration.

Format as JSON:
{{
    "main_prompt": "...",
    "follow_up_questions": ["...", "...", "..."],
    "why_this_prompt": "...",
    "suggested_duration": "10-15 minutes"
}}"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt_request}],
            temperature=0.8
        )

        import json
        return json.loads(response.choices[0].message.content)

    def generate_themed_prompts(self, theme: str) -> List[str]:
        """Generate prompts around a specific theme"""

        theme_prompts = {
            "gratitude": [
                "What are three things you're grateful for today, and why?",
                "Who in your life deserves more appreciation? What would you like to tell them?",
                "What's a challenge you're grateful for because of what it taught you?"
            ],
            "growth": [
                "What's one area where you've grown in the past month?",
                "What's a skill or quality you want to develop? Why is it important to you?",
                "Describe a recent mistake and what you learned from it."
            ],
            "values": [
                "What are your top 3 values? Are you living in alignment with them?",
                "Describe a moment when you felt most like yourself. What were you doing?",
                "What would you want to be remembered for?"
            ],
            "relationships": [
                "How do you want to show up in your relationships?",
                "Who energizes you? Who drains you? Why?",
                "What boundary do you need to set in your relationships?"
            ],
            "emotions": [
                "What emotion have you been avoiding? What might it be trying to tell you?",
                "Describe a recent emotional experience in detail.",
                "What do you need to forgive yourself for?"
            ]
        }

        return theme_prompts.get(theme.lower(), [
            "What's on your mind today?",
            "How are you really feeling?",
            "What do you need right now?"
        ])

    def extract_recent_themes(self, entries: List[JournalEntry]) -> List[str]:
        """Extract themes from recent entries"""
        all_themes = []
        for entry in entries[-5:]:  # Last 5 entries
            if entry.themes:
                all_themes.extend(entry.themes)

        # Count frequency
        from collections import Counter
        theme_counts = Counter(all_themes)
        return [theme for theme, count in theme_counts.most_common(3)]

    def analyze_sentiment_trend(self, entries: List[JournalEntry]) -> str:
        """Analyze sentiment trend"""
        if not entries:
            return "neutral"

        recent_sentiments = [e.sentiment_score for e in entries[-5:] if e.sentiment_score]

        if not recent_sentiments:
            return "neutral"

        avg = sum(recent_sentiments) / len(recent_sentiments)

        if avg > 0.3:
            return "positive"
        elif avg < -0.3:
            return "negative"
        else:
            return "neutral"

    def generate_socratic_questions(
        self,
        entry_content: str,
        context: str = ""
    ) -> List[str]:
        """Generate Socratic questions based on entry"""

        prompt = f"""Based on this journal entry, generate 3-5 Socratic questions that encourage deeper reflection:

Entry:
{entry_content[:500]}...

Generate questions that:
1. Challenge assumptions
2. Explore implications
3. Clarify values and beliefs
4. Examine different perspectives
5. Deepen understanding

Make them thought-provoking but supportive. Return as JSON array:
["Question 1?", "Question 2?", ...]"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        import json
        return json.loads(response.choices[0].message.content)
```

## Journal Analysis Engine

### Semantic Journal Analyzer

```python
from sentence_transformers import SentenceTransformer
import chromadb
from typing import List, Dict
import spacy

class JournalAnalyzer:
    """Analyze journal entries for patterns and insights"""

    def __init__(self, user_id: str):
        self.user_id = user_id
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')
        self.nlp = spacy.load("en_core_web_sm")

        # Vector database for semantic search
        self.chroma_client = chromadb.Client()
        self.collection = self.chroma_client.get_or_create_collection(
            name=f"journal_{user_id}"
        )

    def analyze_entry(self, entry: JournalEntry) -> JournalAnalysis:
        """Comprehensive analysis of journal entry"""

        # Extract themes
        themes = self.extract_themes(entry.content)

        # Sentiment analysis
        sentiment = self.analyze_sentiment(entry.content)

        # Extract entities
        entities = self.extract_entities(entry.content)

        # Generate insights
        insights = self.generate_insights(entry.content, themes, sentiment)

        # Find related entries
        related = self.find_similar_entries(entry.content)

        return JournalAnalysis(
            themes=themes,
            sentiment=sentiment,
            entities=entities,
            word_count=len(entry.content.split()),
            insights=insights
        )

    def extract_themes(self, text: str) -> List[str]:
        """Extract themes from text"""
        doc = self.nlp(text.lower())

        # Extract noun phrases as potential themes
        themes = []
        for chunk in doc.noun_chunks:
            if len(chunk.text.split()) <= 3:  # Keep it concise
                themes.append(chunk.text)

        # Keywords that indicate themes
        theme_keywords = {
            "work": ["work", "job", "career", "project", "deadline"],
            "relationships": ["friend", "family", "partner", "relationship", "love"],
            "health": ["health", "exercise", "sleep", "energy", "body"],
            "emotions": ["feel", "emotion", "happy", "sad", "anxious", "excited"],
            "growth": ["learn", "grow", "develop", "improve", "change"],
            "stress": ["stress", "pressure", "overwhelm", "anxious", "worried"]
        }

        detected_themes = []
        text_lower = text.lower()

        for theme, keywords in theme_keywords.items():
            if any(keyword in text_lower for keyword in keywords):
                detected_themes.append(theme)

        return list(set(detected_themes + themes[:3]))

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

    def extract_entities(self, text: str) -> Dict[str, List[str]]:
        """Extract named entities"""
        doc = self.nlp(text)

        entities = {}
        for ent in doc.ents:
            if ent.label_ not in entities:
                entities[ent.label_] = []
            if ent.text not in entities[ent.label_]:
                entities[ent.label_].append(ent.text)

        return entities

    def generate_insights(
        self,
        text: str,
        themes: List[str],
        sentiment: float
    ) -> str:
        """Generate AI insights about entry"""

        prompt = f"""Analyze this journal entry and provide 2-3 brief, supportive insights:

Entry (excerpt): {text[:300]}...

Detected themes: {', '.join(themes)}
Overall sentiment: {sentiment:.2f}

Provide insights that:
1. Acknowledge key themes or emotions
2. Offer gentle observations or patterns
3. Ask a thought-provoking follow-up question

Keep it warm, non-judgmental, and concise (3-4 sentences)."""

        import openai
        client = openai.OpenAI()

        response = client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def find_similar_entries(
        self,
        text: str,
        n_results: int = 5
    ) -> List[Dict]:
        """Find semantically similar past entries"""

        # Query vector database
        results = self.collection.query(
            query_texts=[text],
            n_results=n_results
        )

        similar_entries = []
        if results['documents']:
            for i, doc in enumerate(results['documents'][0]):
                similar_entries.append({
                    'content': doc[:200],  # Preview
                    'similarity': 1 - results['distances'][0][i] if 'distances' in results else None
                })

        return similar_entries

    def track_theme_evolution(
        self,
        theme: str,
        entries: List[JournalEntry]
    ) -> Dict:
        """Track how a theme evolves over time"""

        theme_entries = [
            e for e in entries
            if e.themes and theme in e.themes
        ]

        if not theme_entries:
            return {"message": f"No entries found for theme: {theme}"}

        # Analyze sentiment over time
        sentiments = [e.sentiment_score for e in theme_entries if e.sentiment_score]

        # Calculate trend
        if len(sentiments) >= 2:
            trend = "improving" if sentiments[-1] > sentiments[0] else \
                   "declining" if sentiments[-1] < sentiments[0] else "stable"
        else:
            trend = "insufficient_data"

        return {
            'theme': theme,
            'total_entries': len(theme_entries),
            'first_mention': theme_entries[0].timestamp,
            'latest_mention': theme_entries[-1].timestamp,
            'sentiment_trend': trend,
            'avg_sentiment': sum(sentiments) / len(sentiments) if sentiments else None
        }
```

## Reflective Conversation Engine

### Socratic Journaling Coach

```python
class SocraticJournalingCoach:
    """Guide users through reflective journaling"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def start_reflection_session(
        self,
        session_type: str = "general"
    ) -> Dict:
        """Start a guided reflection session"""

        session_structures = {
            "general": {
                "intro": "Let's take some time to reflect. What's been on your mind lately?",
                "questions": [
                    "What's been going well?",
                    "What challenges are you facing?",
                    "What patterns do you notice?",
                    "What do you want to change or improve?"
                ]
            },
            "weekly": {
                "intro": "Time for your weekly reflection. Let's look back at the past week.",
                "questions": [
                    "What were your biggest wins this week?",
                    "What didn't go as planned?",
                    "What did you learn about yourself?",
                    "What will you do differently next week?"
                ]
            },
            "emotional": {
                "intro": "Let's explore what you're feeling right now.",
                "questions": [
                    "What emotion are you experiencing most strongly?",
                    "Where do you feel this in your body?",
                    "What might have triggered this feeling?",
                    "What do you need right now?"
                ]
            }
        }

        return session_structures.get(session_type, session_structures["general"])

    def respond_to_journal_entry(
        self,
        entry_text: str,
        conversation_history: List[Dict] = None
    ) -> str:
        """Respond to journal entry with thoughtful questions"""

        system_prompt = """You are a thoughtful journaling companion. Your role is to:

1. Listen deeply and reflect back what you hear
2. Ask open-ended questions that deepen exploration
3. Help identify patterns and insights
4. Be curious, not prescriptive
5. Create a safe space for honest reflection

Use Socratic questioning techniques:
- "What do you mean by...?"
- "Can you give me an example?"
- "How does that make you feel?"
- "What assumptions might you be making?"
- "What would happen if...?"

Keep responses concise (2-3 sentences) and always end with a question."""

        messages = [{"role": "system", "content": system_prompt}]

        if conversation_history:
            messages.extend(conversation_history)

        messages.append({"role": "user", "content": entry_text})

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=messages,
            temperature=0.7,
            max_tokens=200
        )

        return response.choices[0].message.content

    def generate_weekly_synthesis(
        self,
        week_entries: List[JournalEntry]
    ) -> str:
        """Generate synthesis of week's journaling"""

        # Compile week's content
        entries_text = "\n\n---\n\n".join([
            f"Day {i+1}:\n{entry.content[:300]}..."
            for i, entry in enumerate(week_entries)
        ])

        prompt = f"""Review this week's journal entries and create a thoughtful synthesis.

{entries_text}

Provide:
1. Key themes that emerged
2. Patterns or shifts noticed
3. Growth moments
4. Questions to carry forward

Write in second person ("you"), be warm and insightful, 4-5 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def suggest_action_items(
        self,
        entry: JournalEntry
    ) -> List[str]:
        """Extract potential action items from entry"""

        prompt = f"""Read this journal entry and suggest 1-3 concrete, actionable next steps:

{entry.content}

Make suggestions:
- Specific and achievable
- Aligned with the writer's values
- Supportive of their growth

Return as JSON array: ["Action 1", "Action 2", ...]"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.6
        )

        import json
        try:
            return json.loads(response.choices[0].message.content)
        except:
            return []
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
import os

app = FastAPI(title="Reflective Journaling AI")

prompt_generator = JournalPromptGenerator(api_key=os.getenv("OPENAI_API_KEY"))
socratic_coach = SocraticJournalingCoach(api_key=os.getenv("OPENAI_API_KEY"))

@app.post("/journal/entry")
async def create_journal_entry(
    entry: JournalEntryCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Create new journal entry with AI analysis"""

    # Create entry
    journal_entry = JournalEntry(
        user_id=user_id,
        content=entry.content,
        title=entry.title,
        journal_type=entry.journal_type,
        mood=entry.mood,
        tags=entry.tags,
        word_count=len(entry.content.split())
    )

    # Analyze entry
    analyzer = JournalAnalyzer(user_id)
    analysis = analyzer.analyze_entry(journal_entry)

    # Update entry with analysis
    journal_entry.themes = analysis.themes
    journal_entry.sentiment_score = analysis.sentiment
    journal_entry.key_entities = analysis.entities
    journal_entry.ai_insights = analysis.insights

    # Save to database
    db.add(journal_entry)
    db.commit()

    # Generate follow-up questions
    follow_up_questions = prompt_generator.generate_socratic_questions(entry.content)

    return {
        'entry': journal_entry,
        'analysis': analysis,
        'follow_up_questions': follow_up_questions
    }

@app.get("/journal/prompt")
async def get_daily_prompt(
    user_id: str,
    db: Session = Depends(get_db)
):
    """Get personalized daily prompt"""

    # Get recent entries
    recent_entries = db.query(JournalEntry).filter(
        JournalEntry.user_id == user_id
    ).order_by(JournalEntry.timestamp.desc()).limit(10).all()

    # Generate prompt
    prompt = prompt_generator.generate_daily_prompt(
        recent_entries=recent_entries
    )

    return prompt

@app.post("/journal/reflect")
async def start_reflection(
    user_id: str,
    session_type: str = "general",
    db: Session = Depends(get_db)
):
    """Start guided reflection session"""

    session = socratic_coach.start_reflection_session(session_type)

    # Create reflection session record
    reflection = ReflectionSession(
        user_id=user_id,
        session_type=session_type,
        questions_asked=session['questions']
    )

    db.add(reflection)
    db.commit()

    return {
        'session': reflection,
        'intro': session['intro'],
        'first_question': session['questions'][0]
    }

@app.get("/journal/insights/{days}")
async def get_insights(
    user_id: str,
    days: int = 7,
    db: Session = Depends(get_db)
):
    """Get insights from recent journal entries"""

    # Get entries from past N days
    cutoff = datetime.utcnow() - timedelta(days=days)
    entries = db.query(JournalEntry).filter(
        JournalEntry.user_id == user_id,
        JournalEntry.timestamp >= cutoff
    ).all()

    if not entries:
        return {'message': 'No entries in this period'}

    # Generate synthesis
    synthesis = socratic_coach.generate_weekly_synthesis(entries)

    # Analyze themes
    analyzer = JournalAnalyzer(user_id)
    all_themes = []
    for entry in entries:
        if entry.themes:
            all_themes.extend(entry.themes)

    from collections import Counter
    theme_freq = Counter(all_themes)

    return {
        'synthesis': synthesis,
        'top_themes': dict(theme_freq.most_common(5)),
        'entry_count': len(entries),
        'avg_sentiment': sum(e.sentiment_score for e in entries if e.sentiment_score) / len(entries)
    }
```

## Bonus Challenges

### Challenge 1: Voice Journaling
Transcribe and analyze voice journal entries.

### Challenge 2: Mood-Based Prompts
Generate prompts based on detected mood.

### Challenge 3: Journal Visualization
Create visual timelines of themes and emotions.

### Challenge 4: Collaborative Journaling
Share entries with therapist or accountability partner.

### Challenge 5: Dream Journal
Specialized analysis for dream journaling.

## Resources

### Books
- "The Artist's Way" by Julia Cameron
- "Writing Down Your Soul" by Janet Conner
- "Socratic Questioning" research

### Papers
- "Expressive Writing and Health" (Pennebaker, 1997)
- "Benefits of Journaling" meta-analysis

## Success Criteria

### Features
- [ ] Intelligent prompt generation
- [ ] Socratic questioning
- [ ] Theme detection
- [ ] Sentiment analysis
- [ ] Related entry finding
- [ ] Weekly synthesis

### Quality
- [ ] Prompts feel personalized
- [ ] Questions deepen reflection
- [ ] Insights are meaningful
- [ ] Privacy is maintained

This system helps users develop self-awareness through AI-guided reflective journaling!
