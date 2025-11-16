# Project 09: Life Coach AI with Accountability Features

## Overview
Build a comprehensive AI life coach that combines goal setting, weekly planning, daily accountability, progress reviews, and motivational coaching. The system acts as a personal coach providing structure, accountability, and support for personal development across all life areas.

## Learning Objectives
- Design coaching conversation frameworks
- Implement accountability systems
- Build weekly planning and review workflows
- Create motivational and challenging conversations
- Develop life balance tracking
- Apply coaching methodologies (GROW model, CBT techniques)

## Difficulty Level
**Advanced** - Requires coaching knowledge, behavioral psychology, and complex conversation flows

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude (with coaching system prompts)
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL for all coaching data
- **Scheduler**: Celery for scheduled check-ins
- **Notifications**: Multi-channel (SMS, Email, Push)
- **Analytics**: pandas, plotly for visualizations
- **Frontend**: React with coaching dashboard

### Libraries
```python
# requirements.txt
openai==1.12.0
anthropic==0.18.1
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
celery==5.3.6
redis==5.0.1
pydantic==2.6.1
pandas==2.1.4
plotly==5.18.0
python-dateutil==2.8.2
apscheduler==3.10.4
```

## Data Model and Architecture

### Life Coaching Schema

```python
from sqlalchemy import Column, String, DateTime, Integer, JSON, Float, Boolean, Text, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class LifeArea(enum.Enum):
    CAREER = "career"
    HEALTH = "health"
    RELATIONSHIPS = "relationships"
    FINANCE = "finance"
    PERSONAL_GROWTH = "personal_growth"
    SPIRITUALITY = "spirituality"
    RECREATION = "recreation"
    CONTRIBUTION = "contribution"

class CoachingGoal(Base):
    """Long-term coaching goals"""
    __tablename__ = "coaching_goals"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    life_area = Column(String)  # LifeArea enum
    goal_statement = Column(Text, nullable=False)
    why_important = Column(Text)  # Personal motivation

    # SMART criteria
    specific_description = Column(Text)
    measurable_criteria = Column(JSON)
    achievable_plan = Column(Text)
    relevant_reasons = Column(Text)
    timebound_deadline = Column(DateTime)

    # Status
    created_at = Column(DateTime, default=datetime.utcnow)
    status = Column(String, default="active")
    progress_percentage = Column(Float, default=0.0)
    last_reviewed = Column(DateTime)

    # Relationships
    weekly_commitments = relationship("WeeklyCommitment", back_populates="goal")
    accountability_checks = relationship("AccountabilityCheck", back_populates="goal")

class WeeklyCommitment(Base):
    """Weekly commitments and intentions"""
    __tablename__ = "weekly_commitments"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    goal_id = Column(Integer, ForeignKey("coaching_goals.id"), nullable=True)

    week_start = Column(DateTime, index=True)
    week_end = Column(DateTime)

    # Commitments
    commitment_text = Column(Text, nullable=False)
    specific_actions = Column(JSON)  # List of specific actions
    success_criteria = Column(String)

    # Tracking
    completed = Column(Boolean, default=False)
    completion_percentage = Column(Float, default=0.0)
    obstacles_encountered = Column(JSON)

    goal = relationship("CoachingGoal", back_populates="weekly_commitments")
    daily_check_ins = relationship("DailyCheckIn", back_populates="weekly_commitment")

class DailyCheckIn(Base):
    """Daily accountability check-ins"""
    __tablename__ = "daily_check_ins"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    weekly_commitment_id = Column(Integer, ForeignKey("weekly_commitments.id"), nullable=True)
    check_in_date = Column(DateTime, default=datetime.utcnow, index=True)

    # Morning intention
    morning_intention = Column(Text, nullable=True)
    top_priorities = Column(JSON)  # Top 3 priorities for the day
    energy_level = Column(Integer, nullable=True)  # 1-5

    # Evening reflection
    evening_reflection = Column(Text, nullable=True)
    accomplishments = Column(JSON)
    challenges = Column(Text, nullable=True)
    gratitude = Column(JSON)  # 3 things grateful for
    tomorrow_prep = Column(Text, nullable=True)

    # Scoring
    day_rating = Column(Integer, nullable=True)  # 1-10
    alignment_with_goals = Column(Integer, nullable=True)  # 1-10

    weekly_commitment = relationship("WeeklyCommitment", back_populates="daily_check_ins")

class AccountabilityCheck(Base):
    """Scheduled accountability conversations"""
    __tablename__ = "accountability_checks"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    goal_id = Column(Integer, ForeignKey("coaching_goals.id"))

    scheduled_time = Column(DateTime, index=True)
    completed_time = Column(DateTime, nullable=True)
    check_type = Column(String)  # "daily", "weekly", "monthly"

    # Conversation
    questions_asked = Column(JSON)
    responses = Column(JSON)
    coach_feedback = Column(Text)
    action_items = Column(JSON)

    # Outcomes
    progress_made = Column(Boolean, nullable=True)
    obstacles_identified = Column(JSON)
    adjustments_needed = Column(Boolean, default=False)

    goal = relationship("CoachingGoal", back_populates="accountability_checks")

class CoachingSession(Base):
    """Full coaching sessions"""
    __tablename__ = "coaching_sessions"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    session_date = Column(DateTime, default=datetime.utcnow)
    session_type = Column(String)  # "goal_setting", "problem_solving", "review"

    # Session content
    topic = Column(String)
    conversation_log = Column(JSON)
    framework_used = Column(String)  # "GROW", "Solution-Focused", etc.

    # Outcomes
    insights = Column(JSON)
    breakthroughs = Column(Text, nullable=True)
    commitments_made = Column(JSON)
    next_session_focus = Column(Text)

    duration = Column(Integer)  # minutes
    user_rating = Column(Integer, nullable=True)  # 1-5

class LifeBalanceAssessment(Base):
    """Periodic life balance assessments"""
    __tablename__ = "life_balance_assessments"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    assessment_date = Column(DateTime, default=datetime.utcnow)

    # Wheel of Life scores (1-10 for each area)
    career_score = Column(Integer)
    health_score = Column(Integer)
    relationships_score = Column(Integer)
    finance_score = Column(Integer)
    personal_growth_score = Column(Integer)
    spirituality_score = Column(Integer)
    recreation_score = Column(Integer)
    contribution_score = Column(Integer)

    # Analysis
    most_satisfied_area = Column(String)
    least_satisfied_area = Column(String)
    recommended_focus = Column(JSON)
    notes = Column(Text)

# Pydantic models
class GoalCreate(BaseModel):
    life_area: str
    goal_statement: str
    why_important: str
    timebound_deadline: Optional[datetime] = None

class DailyCheckInCreate(BaseModel):
    morning_intention: Optional[str] = None
    top_priorities: Optional[List[str]] = None
    energy_level: Optional[int] = None

class DailyReflectionCreate(BaseModel):
    evening_reflection: str
    accomplishments: List[str]
    challenges: Optional[str] = None
    gratitude: List[str]
    day_rating: int
```

## Coaching Frameworks Implementation

### GROW Model Coach

```python
import openai
from typing import List, Dict

class GROWModelCoach:
    """Implement GROW (Goal, Reality, Options, Way Forward) coaching model"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def conduct_grow_session(
        self,
        topic: str,
        conversation_history: List[Dict] = None
    ) -> Dict:
        """Conduct a GROW model coaching session"""

        if not conversation_history:
            # Start with Goal phase
            return self.goal_phase(topic)

        # Determine current phase
        current_phase = self.detect_phase(conversation_history)

        if current_phase == "goal":
            return self.reality_phase(topic, conversation_history)
        elif current_phase == "reality":
            return self.options_phase(topic, conversation_history)
        elif current_phase == "options":
            return self.way_forward_phase(topic, conversation_history)
        else:
            return self.wrap_up(conversation_history)

    def goal_phase(self, topic: str) -> Dict:
        """G - Goal: What do you want?"""

        prompt = f"""Acting as an executive coach, help the client define their goal around: {topic}

GROW Model - Goal Phase:
- What specifically do you want to achieve?
- What would success look like?
- How will you know when you've achieved it?

Ask 2-3 powerful questions to help them clarify their goal.
Be curious, not prescriptive. Keep response to 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return {
            'phase': 'goal',
            'question': response.choices[0].message.content,
            'next_phase': 'reality'
        }

    def reality_phase(self, topic: str, history: List[Dict]) -> Dict:
        """R - Reality: What's happening now?"""

        conversation = self.format_history(history)

        prompt = f"""Continue this GROW coaching conversation.

Topic: {topic}
Conversation so far:
{conversation}

GROW Model - Reality Phase:
- What's the current situation?
- What have you tried so far?
- What's working? What isn't?
- What obstacles are present?

Ask 2-3 questions to understand their current reality.
Reflect back what you hear. Keep response to 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return {
            'phase': 'reality',
            'question': response.choices[0].message.content,
            'next_phase': 'options'
        }

    def options_phase(self, topic: str, history: List[Dict]) -> Dict:
        """O - Options: What could you do?"""

        conversation = self.format_history(history)

        prompt = f"""Continue this GROW coaching conversation.

Topic: {topic}
Conversation so far:
{conversation}

GROW Model - Options Phase:
- What are all the possible options?
- What else could you try?
- If you had no constraints, what would you do?
- What would you advise a friend in this situation?

Help them brainstorm options. Don't evaluate yet.
Ask questions that generate creative possibilities. 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.8
        )

        return {
            'phase': 'options',
            'question': response.choices[0].message.content,
            'next_phase': 'way_forward'
        }

    def way_forward_phase(self, topic: str, history: List[Dict]) -> Dict:
        """W - Way Forward: What will you do?"""

        conversation = self.format_history(history)

        prompt = f"""Complete this GROW coaching conversation.

Topic: {topic}
Conversation so far:
{conversation}

GROW Model - Way Forward Phase:
- What will you actually do?
- When will you do it?
- What support do you need?
- How committed are you (1-10)?
- What might get in the way?

Help them commit to specific actions. Make it concrete.
Summarize their commitment. 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return {
            'phase': 'way_forward',
            'question': response.choices[0].message.content,
            'next_phase': 'complete'
        }

    def detect_phase(self, history: List[Dict]) -> str:
        """Detect current phase from conversation"""
        # Simple heuristic based on conversation length
        if len(history) <= 3:
            return "goal"
        elif len(history) <= 6:
            return "reality"
        elif len(history) <= 9:
            return "options"
        else:
            return "way_forward"

    def format_history(self, history: List[Dict]) -> str:
        """Format conversation history"""
        return "\n".join([
            f"{msg['role']}: {msg['content']}"
            for msg in history[-6:]  # Last 6 exchanges
        ])

    def wrap_up(self, history: List[Dict]) -> Dict:
        """Wrap up session with summary"""

        conversation = self.format_history(history)

        prompt = f"""Summarize this GROW coaching session:

{conversation}

Provide:
1. The goal they identified
2. Key insights from Reality phase
3. Options they generated
4. Specific commitments made

Make it concise (4-5 sentences) and actionable."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5
        )

        return {
            'phase': 'complete',
            'summary': response.choices[0].message.content
        }
```

## Accountability System

### Daily and Weekly Check-in Manager

```python
from datetime import datetime, timedelta
import asyncio

class AccountabilityManager:
    """Manage accountability check-ins"""

    def __init__(self, user_id: str, api_key: str):
        self.user_id = user_id
        self.client = openai.OpenAI(api_key=api_key)

    async def morning_check_in(self) -> str:
        """Conduct morning intention setting"""

        # Get active goals
        active_goals = self.get_active_goals()

        # Get yesterday's reflection (if exists)
        yesterday = self.get_check_in(datetime.utcnow() - timedelta(days=1))

        prompt = f"""Morning check-in as a life coach.

Active goals:
{chr(10).join([f"- {g.goal_statement}" for g in active_goals[:3]])}

{"Yesterday's reflection: " + yesterday.evening_reflection if yesterday else ""}

Guide the morning intention:
1. How are you feeling today?
2. What are your top 3 priorities?
3. What would make today a great day?
4. What's one small win you want to achieve?

Be energizing and focused. 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    async def evening_reflection(
        self,
        user_input: Dict
    ) -> str:
        """Conduct evening reflection"""

        prompt = f"""Evening reflection as a supportive coach.

Today's accomplishments:
{chr(10).join([f"- {a}" for a in user_input.get('accomplishments', [])])}

Challenges:
{user_input.get('challenges', 'None noted')}

Day rating: {user_input.get('day_rating', 'Not provided')}/10

Provide:
1. Acknowledgment of accomplishments
2. Perspective on challenges
3. One question for reflection
4. Encouragement for tomorrow

Be warm, insightful, and brief (3-4 sentences)."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    async def weekly_review(
        self,
        week_data: Dict
    ) -> str:
        """Conduct comprehensive weekly review"""

        prompt = f"""Weekly coaching review.

Week overview:
- Days checked in: {week_data['check_ins']}/7
- Average day rating: {week_data['avg_rating']}/10
- Goals worked on: {', '.join(week_data['goals'])}
- Wins: {chr(10).join(week_data['wins'])}
- Challenges: {chr(10).join(week_data['challenges'])}

Provide:
1. Celebrate wins
2. Patterns noticed
3. Areas for improvement
4. Question for next week's focus

Be insightful and motivating. 5-6 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def calculate_accountability_score(
        self,
        check_ins: List[DailyCheckIn],
        commitments: List[WeeklyCommitment]
    ) -> Dict:
        """Calculate accountability metrics"""

        # Check-in consistency
        days_with_check_ins = len([c for c in check_ins if c.morning_intention or c.evening_reflection])
        check_in_rate = days_with_check_ins / 7 if len(check_ins) <= 7 else days_with_check_ins / len(check_ins)

        # Commitment completion rate
        completed_commitments = len([c for c in commitments if c.completed])
        commitment_rate = completed_commitments / len(commitments) if commitments else 0

        # Average day rating
        ratings = [c.day_rating for c in check_ins if c.day_rating]
        avg_rating = sum(ratings) / len(ratings) if ratings else 0

        # Overall score
        overall_score = (check_in_rate * 0.3 + commitment_rate * 0.4 + (avg_rating / 10) * 0.3) * 100

        return {
            'check_in_rate': round(check_in_rate * 100, 1),
            'commitment_completion_rate': round(commitment_rate * 100, 1),
            'average_day_rating': round(avg_rating, 1),
            'overall_accountability_score': round(overall_score, 1)
        }

    def get_active_goals(self) -> List[CoachingGoal]:
        """Get user's active goals"""
        # Database query
        pass

    def get_check_in(self, date: datetime) -> Optional[DailyCheckIn]:
        """Get check-in for specific date"""
        # Database query
        pass
```

## Motivational Coaching Engine

### Adaptive Motivation System

```python
class MotivationalCoach:
    """Provide adaptive motivation and support"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def generate_motivation(
        self,
        context: str,
        user_state: Dict
    ) -> str:
        """Generate contextual motivation"""

        # Determine motivation style based on user state
        if user_state.get('energy_level', 3) <= 2:
            style = "gentle_encouragement"
        elif user_state.get('recent_success_rate', 0.5) < 0.3:
            style = "rebuild_confidence"
        elif user_state.get('recent_success_rate', 0.5) > 0.8:
            style = "challenge_growth"
        else:
            style = "balanced_support"

        styles = {
            "gentle_encouragement": "Be warm, understanding, and focus on small steps. Acknowledge difficulty.",
            "rebuild_confidence": "Remind of past successes. Reframe setbacks. Build belief.",
            "challenge_growth": "Push them to the next level. Celebrate wins and raise the bar.",
            "balanced_support": "Mix of acknowledgment and forward momentum. Stay encouraging."
        }

        prompt = f"""As a life coach, provide motivation.

Context: {context}
User state: Energy {user_state.get('energy_level', 3)}/5, Recent success rate {user_state.get('recent_success_rate', 0.5)}

Style: {styles[style]}

Generate motivational message (2-3 sentences).
Make it personal, specific, and actionable."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.8
        )

        return response.choices[0].message.content

    def handle_obstacle(
        self,
        obstacle_description: str,
        goal_context: str
    ) -> str:
        """Coach through obstacles"""

        prompt = f"""As a coach, help navigate this obstacle.

Goal: {goal_context}
Obstacle: {obstacle_description}

Use solution-focused approach:
1. Validate the challenge
2. Ask what's working despite the obstacle
3. Explore what they've tried
4. Generate new possibilities
5. Identify smallest next step

Be empowering. 4-5 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def celebrate_progress(
        self,
        achievement: str,
        goal: CoachingGoal
    ) -> str:
        """Celebrate achievements"""

        celebrations = [
            f"Yes! {achievement}! That's real progress toward {goal.goal_statement}! 🎉",
            f"Look at you! {achievement}! You're making {goal.goal_statement} happen! 💪",
            f"Fantastic! {achievement}! This is exactly what commitment looks like! ⭐"
        ]

        import random
        return random.choice(celebrations)

    def provide_tough_love(
        self,
        situation: str
    ) -> str:
        """Provide challenging accountability when needed"""

        prompt = f"""As a direct but caring coach, provide accountability.

Situation: {situation}

Be:
- Direct and honest
- Respectful but firm
- Challenging but supportive
- Focus on their potential

Call them forward without being harsh. 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content
```

## Life Balance Tracking

### Wheel of Life Assessment

```python
import plotly.graph_objects as go

class LifeBalanceTracker:
    """Track and visualize life balance"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def conduct_wheel_of_life(self) -> Dict:
        """Guide through Wheel of Life assessment"""

        areas = [
            "Career & Work",
            "Health & Fitness",
            "Relationships",
            "Finance",
            "Personal Growth",
            "Spirituality",
            "Recreation & Fun",
            "Contribution"
        ]

        questions = {
            "Career & Work": "On a scale of 1-10, how satisfied are you with your career and work?",
            "Health & Fitness": "How would you rate your physical health and fitness?",
            "Relationships": "How fulfilling are your relationships (family, friends, romantic)?",
            "Finance": "How satisfied are you with your financial situation?",
            "Personal Growth": "How much are you growing and developing as a person?",
            "Spirituality": "How connected do you feel to your spiritual or philosophical beliefs?",
            "Recreation & Fun": "How much joy and recreation do you have in your life?",
            "Contribution": "How much are you contributing to others and making a difference?"
        }

        return {
            'areas': areas,
            'questions': questions
        }

    def analyze_balance(
        self,
        scores: Dict[str, int]
    ) -> Dict:
        """Analyze life balance from scores"""

        # Find highest and lowest areas
        highest = max(scores.items(), key=lambda x: x[1])
        lowest = min(scores.items(), key=lambda x: x[1])

        # Calculate overall balance
        avg_score = sum(scores.values()) / len(scores)
        variance = sum((score - avg_score) ** 2 for score in scores.values()) / len(scores)
        balance_score = 100 - (variance * 2)  # Lower variance = better balance

        # Identify areas needing attention
        needs_attention = [
            area for area, score in scores.items()
            if score <= 5
        ]

        return {
            'overall_average': round(avg_score, 1),
            'balance_score': round(balance_score, 1),
            'strongest_area': highest[0],
            'weakest_area': lowest[0],
            'needs_attention': needs_attention,
            'recommendations': self.generate_recommendations(scores)
        }

    def generate_recommendations(
        self,
        scores: Dict[str, int]
    ) -> List[str]:
        """Generate recommendations based on scores"""

        recommendations = []

        # Find lowest 2-3 areas
        sorted_areas = sorted(scores.items(), key=lambda x: x[1])

        for area, score in sorted_areas[:2]:
            if score <= 5:
                recommendations.append(
                    f"Focus on {area}: Currently at {score}/10. Consider setting a goal in this area."
                )

        # Check for major imbalances
        highest_score = max(scores.values())
        lowest_score = min(scores.values())

        if highest_score - lowest_score >= 6:
            recommendations.append(
                "Your life shows significant imbalance. Consider redistributing energy across areas."
            )

        return recommendations

    def visualize_wheel(
        self,
        scores: Dict[str, int]
    ) -> go.Figure:
        """Create Wheel of Life visualization"""

        areas = list(scores.keys())
        values = list(scores.values())

        fig = go.Figure()

        fig.add_trace(go.Scatterpolar(
            r=values,
            theta=areas,
            fill='toself',
            name='Current State'
        ))

        fig.update_layout(
            polar=dict(
                radialaxis=dict(
                    visible=True,
                    range=[0, 10]
                )
            ),
            showlegend=False,
            title="Your Wheel of Life"
        )

        return fig
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends, BackgroundTasks
from sqlalchemy.orm import Session

app = FastAPI(title="Life Coach AI API")

grow_coach = GROWModelCoach(api_key=os.getenv("OPENAI_API_KEY"))
accountability_mgr = AccountabilityManager("user_id", api_key=os.getenv("OPENAI_API_KEY"))
motivational_coach = MotivationalCoach(api_key=os.getenv("OPENAI_API_KEY"))

@app.post("/coaching/goal")
async def create_coaching_goal(
    goal: GoalCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Create new coaching goal"""

    # Start GROW session to refine goal
    grow_session = grow_coach.conduct_grow_session(goal.goal_statement)

    # Create goal
    coaching_goal = CoachingGoal(
        user_id=user_id,
        life_area=goal.life_area,
        goal_statement=goal.goal_statement,
        why_important=goal.why_important,
        timebound_deadline=goal.timebound_deadline
    )

    db.add(coaching_goal)
    db.commit()

    return {
        'goal': coaching_goal,
        'coaching_question': grow_session['question']
    }

@app.post("/coaching/check-in/morning")
async def morning_check_in(
    user_id: str,
    db: Session = Depends(get_db)
):
    """Conduct morning check-in"""

    mgr = AccountabilityManager(user_id, os.getenv("OPENAI_API_KEY"))
    message = await mgr.morning_check_in()

    # Create check-in record
    check_in = DailyCheckIn(
        user_id=user_id,
        check_in_date=datetime.utcnow()
    )

    db.add(check_in)
    db.commit()

    return {
        'message': message,
        'check_in_id': check_in.id
    }

@app.post("/coaching/check-in/evening")
async def evening_reflection(
    reflection: DailyReflectionCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Submit evening reflection"""

    # Get today's check-in
    today = datetime.utcnow().date()
    check_in = db.query(DailyCheckIn).filter(
        DailyCheckIn.user_id == user_id,
        DailyCheckIn.check_in_date >= datetime.combine(today, datetime.min.time())
    ).first()

    if check_in:
        check_in.evening_reflection = reflection.evening_reflection
        check_in.accomplishments = reflection.accomplishments
        check_in.challenges = reflection.challenges
        check_in.gratitude = reflection.gratitude
        check_in.day_rating = reflection.day_rating

    db.commit()

    # Get AI response
    mgr = AccountabilityManager(user_id, os.getenv("OPENAI_API_KEY"))
    response = await mgr.evening_reflection(reflection.dict())

    return {
        'coach_response': response,
        'check_in': check_in
    }

@app.get("/coaching/accountability-score")
async def get_accountability_score(
    user_id: str,
    days: int = 7,
    db: Session = Depends(get_db)
):
    """Get accountability metrics"""

    # Get recent check-ins
    cutoff = datetime.utcnow() - timedelta(days=days)
    check_ins = db.query(DailyCheckIn).filter(
        DailyCheckIn.user_id == user_id,
        DailyCheckIn.check_in_date >= cutoff
    ).all()

    # Get commitments
    commitments = db.query(WeeklyCommitment).filter(
        WeeklyCommitment.user_id == user_id,
        WeeklyCommitment.week_start >= cutoff
    ).all()

    mgr = AccountabilityManager(user_id, os.getenv("OPENAI_API_KEY"))
    score = mgr.calculate_accountability_score(check_ins, commitments)

    return score
```

## Bonus Challenges

### Challenge 1: Video Coaching Sessions
Add video call integration for live sessions.

### Challenge 2: Group Coaching
Facilitate group coaching circles.

### Challenge 3: Goal Interdependencies
Track how goals affect each other.

### Challenge 4: Energy Management
Track and optimize energy across day/week.

### Challenge 5: Values Alignment
Track alignment between actions and values.

## Resources

### Coaching
- "Co-Active Coaching" by Whitworth et al.
- "The Coaching Habit" by Michael Bungay Stanier
- ICF Core Competencies

### Models
- GROW Model
- Solution-Focused Brief Therapy
- Wheel of Life

## Success Criteria

### Features
- [ ] Goal setting with GROW model
- [ ] Daily check-ins (morning & evening)
- [ ] Weekly reviews
- [ ] Life balance tracking
- [ ] Accountability scoring
- [ ] Adaptive motivation

### Impact
- [ ] Consistent check-in rate > 70%
- [ ] Goal progress > 50%
- [ ] User satisfaction > 4/5
- [ ] Reported life balance improvement

This system provides comprehensive life coaching with structure, accountability, and AI-powered support!
