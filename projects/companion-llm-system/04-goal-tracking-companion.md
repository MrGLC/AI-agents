# Project 04: Goal-Tracking Companion with Progress Monitoring

## Overview
Build an AI companion that helps users set, track, and achieve their goals through intelligent conversation, progress monitoring, accountability check-ins, and adaptive coaching. The system combines goal management with conversational AI to provide personalized support and motivation.

## Learning Objectives
- Design goal-setting frameworks (SMART goals, OKRs)
- Implement progress tracking systems
- Build reminder and notification engines
- Create motivational conversation flows
- Develop accountability mechanisms
- Implement visualization for goal progress

## Difficulty Level
**Intermediate** - Requires backend development, NLP, and scheduling systems

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL for goal data
- **Task Queue**: Celery with Redis
- **Notifications**: Twilio (SMS), SendGrid (Email), Push notifications
- **Frontend**: React or Vue.js
- **Charts**: Chart.js or Recharts
- **Scheduling**: APScheduler or Celery Beat

### Libraries
```python
# requirements.txt
openai==1.12.0
anthropic==0.18.1
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
asyncpg==0.29.0
celery==5.3.6
redis==5.0.1
apscheduler==3.10.4
pydantic==2.6.1
python-dateutil==2.8.2
twilio==8.11.1
sendgrid==6.11.0
pytz==2023.3
```

## Data Model and Architecture

### Goal Management Schema

```python
from sqlalchemy import Column, String, DateTime, Float, Integer, JSON, Boolean, Enum, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class GoalStatus(enum.Enum):
    NOT_STARTED = "not_started"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    ABANDONED = "abandoned"
    ON_HOLD = "on_hold"

class GoalCategory(enum.Enum):
    HEALTH = "health"
    CAREER = "career"
    EDUCATION = "education"
    FINANCE = "finance"
    RELATIONSHIPS = "relationships"
    PERSONAL_GROWTH = "personal_growth"
    CREATIVE = "creative"
    OTHER = "other"

class Goal(Base):
    """Main goal entity"""
    __tablename__ = "goals"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    title = Column(String, nullable=False)
    description = Column(String)
    category = Column(Enum(GoalCategory))
    status = Column(Enum(GoalStatus), default=GoalStatus.NOT_STARTED)

    # SMART goal criteria
    is_specific = Column(Boolean, default=False)
    is_measurable = Column(Boolean, default=False)
    is_achievable = Column(Boolean, default=False)
    is_relevant = Column(Boolean, default=False)
    is_timebound = Column(Boolean, default=False)

    # Metrics
    target_value = Column(Float)
    current_value = Column(Float, default=0.0)
    unit = Column(String)  # e.g., "hours", "pounds", "dollars"

    # Timeline
    start_date = Column(DateTime, default=datetime.utcnow)
    target_date = Column(DateTime)
    completed_date = Column(DateTime, nullable=True)

    # Metadata
    priority = Column(Integer, default=3)  # 1-5, 5 being highest
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    milestones = relationship("Milestone", back_populates="goal")
    progress_logs = relationship("ProgressLog", back_populates="goal")
    check_ins = relationship("CheckIn", back_populates="goal")

class Milestone(Base):
    """Sub-goals or milestones"""
    __tablename__ = "milestones"

    id = Column(Integer, primary_key=True)
    goal_id = Column(Integer, ForeignKey("goals.id"))
    title = Column(String, nullable=False)
    description = Column(String)
    target_value = Column(Float)
    completed = Column(Boolean, default=False)
    target_date = Column(DateTime)
    completed_date = Column(DateTime, nullable=True)

    goal = relationship("Goal", back_populates="milestones")

class ProgressLog(Base):
    """Track progress updates"""
    __tablename__ = "progress_logs"

    id = Column(Integer, primary_key=True)
    goal_id = Column(Integer, ForeignKey("goals.id"))
    timestamp = Column(DateTime, default=datetime.utcnow)
    value = Column(Float)
    notes = Column(String)
    mood = Column(String)  # How user felt during update
    metadata = Column(JSON)

    goal = relationship("Goal", back_populates="progress_logs")

class CheckIn(Base):
    """Scheduled check-ins and accountability"""
    __tablename__ = "check_ins"

    id = Column(Integer, primary_key=True)
    goal_id = Column(Integer, ForeignKey("goals.id"))
    scheduled_time = Column(DateTime)
    completed_time = Column(DateTime, nullable=True)
    response = Column(String)
    sentiment = Column(Float)
    was_successful = Column(Boolean)

    goal = relationship("Goal", back_populates="check_ins")

# Pydantic models for API
class GoalCreate(BaseModel):
    title: str
    description: Optional[str] = None
    category: str
    target_value: Optional[float] = None
    unit: Optional[str] = None
    target_date: Optional[datetime] = None
    priority: int = 3

class GoalUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    status: Optional[str] = None
    current_value: Optional[float] = None
    target_value: Optional[float] = None
    target_date: Optional[datetime] = None

class ProgressUpdate(BaseModel):
    value: float
    notes: Optional[str] = None
    mood: Optional[str] = None
```

## Goal Setting Assistant

### SMART Goal Validator

```python
from typing import Dict, List
import openai
import re
from datetime import datetime, timedelta

class SMARTGoalAssistant:
    """Help users create SMART goals through conversation"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def validate_smart_criteria(self, goal_description: str) -> Dict:
        """Analyze if goal meets SMART criteria"""

        prompt = f"""Analyze this goal and determine if it meets SMART criteria:

Goal: "{goal_description}"

Evaluate each criterion:
1. Specific - Is it clear and well-defined?
2. Measurable - Can progress be tracked with numbers/metrics?
3. Achievable - Is it realistic and attainable?
4. Relevant - Does it align with broader objectives?
5. Time-bound - Does it have a deadline or timeframe?

Respond in JSON:
{{
    "specific": {{"met": true/false, "reason": "..."}},
    "measurable": {{"met": true/false, "reason": "...", "suggested_metric": "..."}},
    "achievable": {{"met": true/false, "reason": "..."}},
    "relevant": {{"met": true/false, "reason": "..."}},
    "timebound": {{"met": true/false, "reason": "...", "suggested_deadline": "..."}}
}}"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        import json
        return json.loads(response.choices[0].message.content)

    def improve_goal(self, goal_description: str) -> Dict:
        """Suggest improvements to make goal SMART"""

        smart_analysis = self.validate_smart_criteria(goal_description)

        # Generate improvement suggestions
        improvements = []
        improved_goal = goal_description

        for criterion, data in smart_analysis.items():
            if not data.get('met', False):
                improvements.append({
                    'criterion': criterion,
                    'reason': data.get('reason', ''),
                    'suggestion': data.get('suggested_metric') or data.get('suggested_deadline')
                })

        # Generate improved version
        if improvements:
            improved_goal = self.generate_improved_goal(goal_description, improvements)

        return {
            'original_goal': goal_description,
            'improved_goal': improved_goal,
            'improvements': improvements,
            'smart_score': sum(1 for c in smart_analysis.values() if c.get('met', False)) / 5
        }

    def generate_improved_goal(self, original: str, improvements: List[Dict]) -> str:
        """Generate improved SMART goal"""

        prompt = f"""Rewrite this goal to address these improvements:

Original: {original}

Improvements needed:
{chr(10).join([f"- {imp['criterion']}: {imp['reason']}" for imp in improvements])}

Generate a SMART version of this goal in one clear sentence."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def suggest_milestones(self, goal: Goal) -> List[Dict]:
        """Suggest milestones for a goal"""

        prompt = f"""Break down this goal into 3-5 achievable milestones:

Goal: {goal.title}
Description: {goal.description}
Target: {goal.target_value} {goal.unit}
Deadline: {goal.target_date}

Generate milestones that are evenly spaced and build toward the final goal.
Return as JSON array:
[
    {{
        "title": "Milestone 1",
        "description": "...",
        "target_value": 25,
        "target_date": "YYYY-MM-DD"
    }},
    ...
]"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        import json
        return json.loads(response.choices[0].message.content)

    def conversational_goal_setup(self, user_input: str, conversation_history: List[Dict]) -> str:
        """Interactive goal setting conversation"""

        system_prompt = """You are a supportive goal-setting coach. Help the user define a SMART goal through conversation.

Guidelines:
- Ask clarifying questions to make goals specific
- Suggest measurable metrics
- Ensure goals are achievable but challenging
- Help identify relevant motivations
- Establish clear timelines
- Be encouraging and positive
- Guide them step-by-step"""

        messages = [{"role": "system", "content": system_prompt}]
        messages.extend(conversation_history)
        messages.append({"role": "user", "content": user_input})

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=messages,
            temperature=0.7
        )

        return response.choices[0].message.content
```

## Progress Tracking and Monitoring

### Progress Tracker

```python
from datetime import datetime, timedelta
import pandas as pd
import numpy as np

class ProgressTracker:
    """Track and analyze goal progress"""

    def __init__(self, goal: Goal):
        self.goal = goal

    def calculate_progress_percentage(self) -> float:
        """Calculate current progress as percentage"""
        if not self.goal.target_value or self.goal.target_value == 0:
            return 0.0

        progress = (self.goal.current_value / self.goal.target_value) * 100
        return min(progress, 100.0)

    def calculate_time_progress(self) -> float:
        """Calculate how much time has passed"""
        if not self.goal.target_date:
            return 0.0

        total_time = (self.goal.target_date - self.goal.start_date).total_seconds()
        elapsed_time = (datetime.utcnow() - self.goal.start_date).total_seconds()

        return min((elapsed_time / total_time) * 100, 100.0)

    def is_on_track(self) -> Dict:
        """Determine if goal is on track"""
        progress_pct = self.calculate_progress_percentage()
        time_pct = self.calculate_time_progress()

        # Should have at least as much progress as time passed
        on_track = progress_pct >= time_pct - 10  # 10% buffer

        status = "on_track" if on_track else "behind"
        if progress_pct > time_pct + 10:
            status = "ahead"

        return {
            'status': status,
            'progress_percentage': round(progress_pct, 1),
            'time_percentage': round(time_pct, 1),
            'difference': round(progress_pct - time_pct, 1)
        }

    def predict_completion_date(self, progress_logs: List[ProgressLog]) -> datetime:
        """Predict when goal will be completed based on current rate"""
        if len(progress_logs) < 2:
            return self.goal.target_date

        # Calculate average rate of progress
        df = pd.DataFrame([
            {'date': log.timestamp, 'value': log.value}
            for log in progress_logs
        ])
        df = df.sort_values('date')

        # Linear regression to predict completion
        from sklearn.linear_model import LinearRegression

        X = np.array([(d - df['date'].min()).total_seconds() for d in df['date']]).reshape(-1, 1)
        y = df['value'].values

        model = LinearRegression()
        model.fit(X, y)

        # Predict when we'll reach target
        remaining = self.goal.target_value - self.goal.current_value
        if model.coef_[0] <= 0:
            return self.goal.target_date

        seconds_to_target = remaining / model.coef_[0]
        predicted_date = datetime.utcnow() + timedelta(seconds=seconds_to_target)

        return predicted_date

    def generate_progress_report(self, progress_logs: List[ProgressLog]) -> str:
        """Generate natural language progress report"""
        tracking = self.is_on_track()
        predicted_date = self.predict_completion_date(progress_logs)

        report = f"""Progress Report for: {self.goal.title}

Current Status: {tracking['status'].replace('_', ' ').title()}
Progress: {tracking['progress_percentage']}% complete
Time Elapsed: {tracking['time_percentage']}%

Current Value: {self.goal.current_value} {self.goal.unit}
Target Value: {self.goal.target_value} {self.goal.unit}

Target Date: {self.goal.target_date.strftime('%Y-%m-%d')}
Predicted Completion: {predicted_date.strftime('%Y-%m-%d')}
"""

        if tracking['status'] == 'behind':
            days_behind = (predicted_date - self.goal.target_date).days
            report += f"\nYou're {days_behind} days behind schedule. Let's discuss how to get back on track!"
        elif tracking['status'] == 'ahead':
            days_ahead = (self.goal.target_date - predicted_date).days
            report += f"\nYou're {days_ahead} days ahead of schedule! Great work!"
        else:
            report += "\nYou're right on track! Keep up the great work!"

        return report

    def get_next_milestone(self, milestones: List[Milestone]) -> Optional[Milestone]:
        """Get the next uncompleted milestone"""
        incomplete = [m for m in milestones if not m.completed]
        if not incomplete:
            return None

        # Return closest by target date
        return min(incomplete, key=lambda m: m.target_date or datetime.max)
```

## Accountability and Check-ins

### Check-in Scheduler

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
import asyncio

class CheckInScheduler:
    """Schedule and manage accountability check-ins"""

    def __init__(self):
        self.scheduler = AsyncIOScheduler()
        self.scheduler.start()

    def schedule_daily_check_in(
        self,
        user_id: str,
        goal_id: int,
        time_of_day: str = "18:00"  # 6 PM default
    ):
        """Schedule daily check-in"""
        hour, minute = map(int, time_of_day.split(':'))

        self.scheduler.add_job(
            self.send_check_in,
            CronTrigger(hour=hour, minute=minute),
            args=[user_id, goal_id],
            id=f"daily_checkin_{user_id}_{goal_id}",
            replace_existing=True
        )

    def schedule_weekly_review(
        self,
        user_id: str,
        goal_id: int,
        day_of_week: int = 6,  # Sunday
        time: str = "10:00"
    ):
        """Schedule weekly progress review"""
        hour, minute = map(int, time.split(':'))

        self.scheduler.add_job(
            self.send_weekly_review,
            CronTrigger(day_of_week=day_of_week, hour=hour, minute=minute),
            args=[user_id, goal_id],
            id=f"weekly_review_{user_id}_{goal_id}",
            replace_existing=True
        )

    async def send_check_in(self, user_id: str, goal_id: int):
        """Send check-in message"""
        # Get goal details
        # goal = get_goal(goal_id)

        # Generate personalized check-in message
        message = await self.generate_check_in_message(user_id, goal_id)

        # Send via preferred channel (SMS, email, push)
        await self.send_notification(user_id, message)

        # Log check-in
        # create_check_in_log(goal_id, scheduled_time=datetime.utcnow())

    async def generate_check_in_message(self, user_id: str, goal_id: int) -> str:
        """Generate personalized check-in message"""
        # This would use LLM to generate encouraging, personalized messages

        messages = [
            f"Hey! How did you do with your goal today?",
            f"Time for a quick check-in! Any progress to report?",
            f"Let's see how you're doing! Any wins today?",
        ]

        import random
        return random.choice(messages)

    async def send_weekly_review(self, user_id: str, goal_id: int):
        """Send weekly progress review"""
        # Generate comprehensive weekly report
        # Send via email with charts
        pass

    async def send_notification(self, user_id: str, message: str, channel: str = "sms"):
        """Send notification via specified channel"""
        if channel == "sms":
            await self.send_sms(user_id, message)
        elif channel == "email":
            await self.send_email(user_id, message)
        elif channel == "push":
            await self.send_push(user_id, message)

    async def send_sms(self, user_id: str, message: str):
        """Send SMS via Twilio"""
        from twilio.rest import Client
        # Implement Twilio SMS
        pass

    async def send_email(self, user_id: str, message: str):
        """Send email via SendGrid"""
        from sendgrid import SendGridAPIClient
        from sendgrid.helpers.mail import Mail
        # Implement SendGrid email
        pass

    async def send_push(self, user_id: str, message: str):
        """Send push notification"""
        # Implement push notification
        pass
```

### Motivational Coach

```python
class MotivationalCoach:
    """Provide motivation and encouragement"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def generate_motivation(
        self,
        goal: Goal,
        recent_progress: List[ProgressLog],
        context: str = ""
    ) -> str:
        """Generate personalized motivational message"""

        # Analyze recent progress
        if recent_progress:
            latest = recent_progress[0]
            trend = "improving" if len(recent_progress) > 1 and latest.value > recent_progress[1].value else "steady"
        else:
            trend = "starting"

        prompt = f"""Generate a brief, encouraging message for someone working on this goal:

Goal: {goal.title}
Progress: {goal.current_value}/{goal.target_value} {goal.unit}
Trend: {trend}
Context: {context}

Make it:
- Personal and warm
- Specific to their progress
- Encouraging but realistic
- 2-3 sentences max"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.8
        )

        return response.choices[0].message.content

    def handle_setback(self, goal: Goal, setback_description: str) -> str:
        """Provide support after a setback"""

        prompt = f"""The user experienced a setback with their goal:

Goal: {goal.title}
Setback: {setback_description}

Provide:
1. Empathy and validation
2. Perspective on setbacks being normal
3. Concrete next step
4. Encouragement

Be warm, understanding, and actionable. 3-4 sentences."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def celebrate_milestone(self, milestone: Milestone) -> str:
        """Celebrate milestone achievement"""

        celebrations = [
            f"Incredible! You completed '{milestone.title}'! That's real progress! 🎉",
            f"Milestone achieved: {milestone.title}! You're crushing it! 💪",
            f"Yes! {milestone.title} is done! You're getting closer every day! ⭐",
        ]

        import random
        return random.choice(celebrations)
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

app = FastAPI(title="Goal Tracking Companion")

# Database dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/goals/create")
async def create_goal(
    goal_data: GoalCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Create a new goal with AI assistance"""

    # Validate and improve goal
    assistant = SMARTGoalAssistant(api_key=os.getenv("OPENAI_API_KEY"))
    improved = assistant.improve_goal(f"{goal_data.title}. {goal_data.description}")

    # Create goal
    goal = Goal(
        user_id=user_id,
        title=goal_data.title,
        description=goal_data.description,
        category=GoalCategory[goal_data.category.upper()],
        target_value=goal_data.target_value,
        unit=goal_data.unit,
        target_date=goal_data.target_date,
        priority=goal_data.priority
    )

    db.add(goal)
    db.commit()
    db.refresh(goal)

    # Generate milestones
    suggested_milestones = assistant.suggest_milestones(goal)

    # Schedule check-ins
    scheduler = CheckInScheduler()
    scheduler.schedule_daily_check_in(user_id, goal.id)
    scheduler.schedule_weekly_review(user_id, goal.id)

    return {
        'goal': goal,
        'smart_analysis': improved,
        'suggested_milestones': suggested_milestones
    }

@app.post("/goals/{goal_id}/progress")
async def log_progress(
    goal_id: int,
    progress: ProgressUpdate,
    db: Session = Depends(get_db)
):
    """Log progress update"""

    goal = db.query(Goal).filter(Goal.id == goal_id).first()
    if not goal:
        raise HTTPException(status_code=404, detail="Goal not found")

    # Update current value
    goal.current_value = progress.value
    goal.updated_at = datetime.utcnow()

    # Create progress log
    log = ProgressLog(
        goal_id=goal_id,
        value=progress.value,
        notes=progress.notes,
        mood=progress.mood
    )

    db.add(log)
    db.commit()

    # Analyze progress
    tracker = ProgressTracker(goal)
    analysis = tracker.is_on_track()

    # Generate motivation
    coach = MotivationalCoach(api_key=os.getenv("OPENAI_API_KEY"))
    motivation = coach.generate_motivation(goal, [log])

    # Check if milestone completed
    milestones = db.query(Milestone).filter(
        Milestone.goal_id == goal_id,
        Milestone.completed == False
    ).all()

    completed_milestones = []
    for milestone in milestones:
        if goal.current_value >= milestone.target_value:
            milestone.completed = True
            milestone.completed_date = datetime.utcnow()
            db.commit()
            completed_milestones.append(milestone)

    return {
        'progress': analysis,
        'motivation': motivation,
        'completed_milestones': completed_milestones
    }

@app.get("/goals/{goal_id}/report")
async def get_progress_report(
    goal_id: int,
    db: Session = Depends(get_db)
):
    """Get comprehensive progress report"""

    goal = db.query(Goal).filter(Goal.id == goal_id).first()
    if not goal:
        raise HTTPException(status_code=404, detail="Goal not found")

    # Get progress logs
    logs = db.query(ProgressLog).filter(
        ProgressLog.goal_id == goal_id
    ).order_by(ProgressLog.timestamp.desc()).all()

    # Generate report
    tracker = ProgressTracker(goal)
    report = tracker.generate_progress_report(logs)

    return {
        'report': report,
        'tracking': tracker.is_on_track(),
        'recent_logs': logs[:10]
    }

@app.get("/goals/active")
async def get_active_goals(
    user_id: str,
    db: Session = Depends(get_db)
):
    """Get all active goals"""

    goals = db.query(Goal).filter(
        Goal.user_id == user_id,
        Goal.status.in_([GoalStatus.IN_PROGRESS, GoalStatus.NOT_STARTED])
    ).all()

    goals_with_progress = []
    for goal in goals:
        tracker = ProgressTracker(goal)
        goals_with_progress.append({
            'goal': goal,
            'progress': tracker.calculate_progress_percentage(),
            'status': tracker.is_on_track()
        })

    return goals_with_progress
```

## Visualization Dashboard

```javascript
// GoalDashboard.jsx
import React, { useState, useEffect } from 'react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';
import { CircularProgressbar, buildStyles } from 'react-circular-progressbar';
import 'react-circular-progressbar/dist/styles.css';

const GoalDashboard = ({ userId }) => {
  const [goals, setGoals] = useState([]);
  const [selectedGoal, setSelectedGoal] = useState(null);
  const [progressData, setProgressData] = useState([]);

  useEffect(() => {
    fetchGoals();
  }, [userId]);

  const fetchGoals = async () => {
    const response = await fetch(`/goals/active?user_id=${userId}`);
    const data = await response.json();
    setGoals(data);
    if (data.length > 0) {
      setSelectedGoal(data[0]);
    }
  };

  return (
    <div className="goal-dashboard">
      <h1>Your Goals</h1>

      {/* Goal Cards */}
      <div className="goal-grid">
        {goals.map(({ goal, progress, status }) => (
          <div
            key={goal.id}
            className={`goal-card ${status.status}`}
            onClick={() => setSelectedGoal({ goal, progress, status })}
          >
            <h3>{goal.title}</h3>
            <div className="progress-circle">
              <CircularProgressbar
                value={progress}
                text={`${Math.round(progress)}%`}
                styles={buildStyles({
                  textColor: '#333',
                  pathColor: status.status === 'ahead' ? '#4caf50' : '#2196f3',
                  trailColor: '#d6d6d6'
                })}
              />
            </div>
            <p className="goal-metric">
              {goal.current_value} / {goal.target_value} {goal.unit}
            </p>
            <span className={`status-badge ${status.status}`}>
              {status.status.replace('_', ' ')}
            </span>
          </div>
        ))}
      </div>

      {/* Detailed View */}
      {selectedGoal && (
        <div className="goal-detail">
          <h2>{selectedGoal.goal.title}</h2>
          <p>{selectedGoal.goal.description}</p>

          {/* Progress Chart */}
          <div className="chart-container">
            <h3>Progress Over Time</h3>
            <LineChart width={600} height={300} data={progressData}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis />
              <Tooltip />
              <Legend />
              <Line type="monotone" dataKey="value" stroke="#8884d8" />
              <Line type="monotone" dataKey="target" stroke="#82ca9d" strokeDasharray="5 5" />
            </LineChart>
          </div>

          {/* Quick Actions */}
          <div className="actions">
            <button onClick={() => showProgressForm(selectedGoal.goal.id)}>
              Log Progress
            </button>
            <button onClick={() => viewReport(selectedGoal.goal.id)}>
              View Report
            </button>
          </div>
        </div>
      )}
    </div>
  );
};

export default GoalDashboard;
```

## Bonus Challenges

### Challenge 1: Habit Stacking
Integrate with habit tracking for compound goals.

### Challenge 2: Social Accountability
Add accountability partners who can see progress.

### Challenge 3: Gamification
Add points, badges, streaks for motivation.

### Challenge 4: AI Coach Sessions
Scheduled 1-on-1 coaching sessions with AI.

### Challenge 5: Voice Check-ins
Voice-based progress logging and check-ins.

## Resources

### Papers
- "Goal Setting Theory" (Locke & Latham, 1990)
- "The Science of Goal Setting" (Morisano et al., 2010)

### Documentation
- [APScheduler](https://apscheduler.readthedocs.io/)
- [Celery](https://docs.celeryq.dev/)
- [Twilio API](https://www.twilio.com/docs)

## Success Criteria

### Features
- [ ] SMART goal creation with AI assistance
- [ ] Progress tracking and visualization
- [ ] Automated check-ins and reminders
- [ ] Milestone tracking
- [ ] Progress predictions
- [ ] Motivational messaging

### Technical
- [ ] Scheduled tasks run reliably
- [ ] Real-time progress updates
- [ ] Mobile-responsive dashboard
- [ ] Multi-channel notifications

### User Experience
- [ ] Easy goal creation
- [ ] Clear progress visualization
- [ ] Helpful, not annoying reminders
- [ ] Motivating interactions
- [ ] Actionable insights

This system provides comprehensive goal tracking with AI-powered coaching and accountability!
