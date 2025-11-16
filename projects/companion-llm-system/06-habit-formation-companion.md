# Project 06: Habit Formation Companion with Daily Check-ins

## Overview
Build an AI companion that helps users build and maintain positive habits through daily check-ins, streak tracking, behavioral psychology techniques, and adaptive encouragement. The system combines habit science with conversational AI to create sustainable behavior change.

## Learning Objectives
- Implement habit tracking and streak systems
- Apply behavioral psychology principles (triggers, rewards, tiny habits)
- Design effective reminder and notification systems
- Build habit stacking and routine creation
- Create accountability through conversation
- Implement adaptive difficulty and progression

## Difficulty Level
**Intermediate** - Requires scheduling systems, behavioral science knowledge, and engagement design

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL for habit data
- **Scheduler**: Celery with Redis for check-ins
- **Notifications**: Push notifications, SMS, Email
- **Frontend**: React Native (for mobile) or React
- **Analytics**: pandas for habit analytics

### Libraries
```python
# requirements.txt
openai==1.12.0
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
celery==5.3.6
redis==5.0.1
apscheduler==3.10.4
pydantic==2.6.1
python-dateutil==2.8.2
pytz==2023.3
numpy==1.26.3
pandas==2.1.4
```

## Data Model and Architecture

### Habit Tracking Schema

```python
from sqlalchemy import Column, String, DateTime, Integer, Boolean, JSON, Float, ForeignKey, Enum
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime, time
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class HabitFrequency(enum.Enum):
    DAILY = "daily"
    WEEKLY = "weekly"
    WEEKDAYS = "weekdays"
    CUSTOM = "custom"

class HabitCategory(enum.Enum):
    HEALTH = "health"
    FITNESS = "fitness"
    PRODUCTIVITY = "productivity"
    LEARNING = "learning"
    MINDFULNESS = "mindfulness"
    SOCIAL = "social"
    CREATIVE = "creative"

class Habit(Base):
    """Core habit entity"""
    __tablename__ = "habits"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    name = Column(String, nullable=False)
    description = Column(String)
    category = Column(Enum(HabitCategory))

    # Frequency and scheduling
    frequency = Column(Enum(HabitFrequency))
    target_days = Column(JSON)  # [0, 1, 2, 3, 4, 5, 6] for daily, or specific days
    reminder_time = Column(String)  # "08:00", "14:30"
    time_of_day = Column(String)  # "morning", "afternoon", "evening"

    # Habit design (Tiny Habits methodology)
    trigger = Column(String)  # "After I pour coffee..."
    tiny_version = Column(String)  # "I will do 2 push-ups"
    celebration = Column(String)  # How to celebrate completion

    # Tracking
    current_streak = Column(Integer, default=0)
    longest_streak = Column(Integer, default=0)
    total_completions = Column(Integer, default=0)
    completion_rate = Column(Float, default=0.0)  # 0-100

    # Status
    is_active = Column(Boolean, default=True)
    difficulty_level = Column(Integer, default=1)  # 1-5, auto-adjusted
    created_at = Column(DateTime, default=datetime.utcnow)
    started_at = Column(DateTime, nullable=True)

    # Relationships
    check_ins = relationship("HabitCheckIn", back_populates="habit")
    milestones = relationship("HabitMilestone", back_populates="habit")

class HabitCheckIn(Base):
    """Daily check-in records"""
    __tablename__ = "habit_check_ins"

    id = Column(Integer, primary_key=True)
    habit_id = Column(Integer, ForeignKey("habits.id"))
    user_id = Column(String, index=True)

    check_in_date = Column(DateTime, index=True)
    completed = Column(Boolean, default=False)
    completion_time = Column(DateTime, nullable=True)

    # Context
    difficulty_rating = Column(Integer, nullable=True)  # 1-5
    notes = Column(String, nullable=True)
    mood = Column(String, nullable=True)
    obstacles = Column(JSON, nullable=True)

    # AI interaction
    reminder_sent = Column(Boolean, default=False)
    reminder_sent_at = Column(DateTime, nullable=True)
    ai_encouragement = Column(String, nullable=True)
    conversation_log = Column(JSON, nullable=True)

    habit = relationship("Habit", back_populates="check_ins")

class HabitMilestone(Base):
    """Achievement milestones"""
    __tablename__ = "habit_milestones"

    id = Column(Integer, primary_key=True)
    habit_id = Column(Integer, ForeignKey("habits.id"))
    milestone_type = Column(String)  # "streak", "total_completions", "custom"
    threshold = Column(Integer)  # e.g., 7 for week streak
    achieved = Column(Boolean, default=False)
    achieved_at = Column(DateTime, nullable=True)
    reward_message = Column(String)

    habit = relationship("Habit", back_populates="milestones")

class HabitRoutine(Base):
    """Collection of habits as a routine"""
    __tablename__ = "habit_routines"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    name = Column(String)  # "Morning Routine", "Evening Wind-down"
    description = Column(String)
    time_of_day = Column(String)
    habit_ids = Column(JSON)  # Ordered list of habit IDs
    created_at = Column(DateTime, default=datetime.utcnow)

# Pydantic models
class HabitCreate(BaseModel):
    name: str
    description: Optional[str] = None
    category: str
    frequency: str = "daily"
    reminder_time: Optional[str] = None
    trigger: Optional[str] = None
    tiny_version: Optional[str] = None

class CheckInResponse(BaseModel):
    completed: bool
    difficulty_rating: Optional[int] = None
    notes: Optional[str] = None
    mood: Optional[str] = None
```

## Habit Design Assistant

### Tiny Habits Builder

```python
import openai
from typing import Dict, List

class TinyHabitsBuilder:
    """Help users design effective tiny habits"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def design_tiny_habit(self, desired_habit: str, user_context: Dict = None) -> Dict:
        """Convert big habit into tiny, actionable version"""

        prompt = f"""Help design a Tiny Habit using BJ Fogg's methodology.

Desired habit: {desired_habit}

Create a Tiny Habit following this formula:
"After I [EXISTING HABIT/ANCHOR], I will [TINY VERSION OF NEW HABIT]."

Guidelines:
- Make it SO tiny it's almost impossible to fail (e.g., 2 push-ups, not 20)
- Attach to an existing reliable anchor
- Make it take less than 30 seconds
- Ensure it's the first step toward the bigger habit

Respond in JSON:
{{
    "anchor": "After I pour my morning coffee",
    "tiny_behavior": "I will do 2 push-ups",
    "celebration": "I will say 'Yes!' and smile",
    "full_recipe": "After I pour my morning coffee, I will do 2 push-ups, then celebrate.",
    "why_it_works": "...",
    "expansion_path": ["Week 1: 2 push-ups", "Week 2: 5 push-ups", ...]
}}"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        import json
        return json.loads(response.choices[0].message.content)

    def suggest_anchors(self, user_routine: List[str]) -> List[Dict]:
        """Suggest reliable anchors from user's routine"""

        common_anchors = [
            "After I wake up",
            "After I brush my teeth",
            "After I pour coffee/tea",
            "After I sit down at my desk",
            "After I get home",
            "After I eat lunch",
            "After I close my laptop",
            "Before I go to bed"
        ]

        # In production, personalize based on user_routine
        return [
            {'anchor': anchor, 'reliability': 0.9}
            for anchor in common_anchors
        ]

    def create_habit_stack(self, habits: List[Habit]) -> Dict:
        """Create a habit stack (chain multiple habits)"""

        if not habits:
            return {}

        # Build stack: Habit 1 becomes anchor for Habit 2, etc.
        stack = []

        for i, habit in enumerate(habits):
            if i == 0:
                stack.append({
                    'step': i + 1,
                    'habit': habit.name,
                    'instruction': f"{habit.trigger}, {habit.tiny_version}"
                })
            else:
                stack.append({
                    'step': i + 1,
                    'habit': habit.name,
                    'instruction': f"After {habits[i-1].tiny_version}, {habit.tiny_version}"
                })

        return {
            'stack_name': f"{habits[0].time_of_day.title()} Stack",
            'steps': stack,
            'estimated_time': f"{len(habits) * 2} minutes",
            'completion_phrase': "Stack complete! Celebrate!"
        }

    def optimize_habit_difficulty(
        self,
        habit: Habit,
        recent_check_ins: List[HabitCheckIn]
    ) -> Dict:
        """Adjust habit difficulty based on performance"""

        if len(recent_check_ins) < 7:
            return {'action': 'wait', 'message': 'Keep going! Need more data to optimize.'}

        # Calculate recent success rate
        completed = sum(1 for ci in recent_check_ins[:7] if ci.completed)
        success_rate = completed / 7

        # Calculate average difficulty rating
        difficulty_ratings = [ci.difficulty_rating for ci in recent_check_ins[:7] if ci.difficulty_rating]
        avg_difficulty = sum(difficulty_ratings) / len(difficulty_ratings) if difficulty_ratings else 3

        recommendation = {}

        # Too easy (high success, low difficulty)
        if success_rate >= 0.9 and avg_difficulty <= 2:
            recommendation = {
                'action': 'increase_difficulty',
                'message': "You're crushing this! Ready to level up?",
                'suggested_change': self.suggest_progression(habit)
            }

        # Too hard (low success, high difficulty)
        elif success_rate <= 0.5 and avg_difficulty >= 4:
            recommendation = {
                'action': 'decrease_difficulty',
                'message': "Let's make this easier to maintain consistency.",
                'suggested_change': self.suggest_simplification(habit)
            }

        # Just right
        else:
            recommendation = {
                'action': 'maintain',
                'message': "Perfect! You're in the sweet spot. Keep it up!"
            }

        return recommendation

    def suggest_progression(self, habit: Habit) -> str:
        """Suggest next level for habit"""
        # Extract number from tiny_version if present
        import re
        numbers = re.findall(r'\d+', habit.tiny_version)

        if numbers:
            current = int(numbers[0])
            next_level = current + min(current, 5)  # Gradual increase
            return habit.tiny_version.replace(str(current), str(next_level))

        return f"{habit.tiny_version} (increase duration or intensity slightly)"

    def suggest_simplification(self, habit: Habit) -> str:
        """Suggest easier version"""
        return f"Try an even smaller version: {habit.tiny_version.replace('5', '2').replace('10', '5')}"
```

## Check-in System

### Daily Check-in Manager

```python
from datetime import datetime, timedelta
import asyncio
from typing import Optional

class CheckInManager:
    """Manage daily check-ins and reminders"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    async def send_daily_reminder(self, habit: Habit):
        """Send personalized daily reminder"""

        # Get recent context
        recent_check_ins = self.get_recent_check_ins(habit.id, days=7)

        # Calculate streak status
        streak_status = self.calculate_streak_status(recent_check_ins)

        # Generate personalized message
        message = await self.generate_reminder_message(
            habit,
            streak_status,
            recent_check_ins
        )

        # Send notification
        await self.send_notification(self.user_id, message, habit.id)

        # Log reminder sent
        self.log_reminder(habit.id)

    async def generate_reminder_message(
        self,
        habit: Habit,
        streak_status: Dict,
        recent_check_ins: List[HabitCheckIn]
    ) -> str:
        """Generate personalized reminder message"""

        templates = {
            'new_habit': [
                f"Time for {habit.name}! Remember: {habit.tiny_version} 🌱",
                f"Your daily {habit.name} moment is here! {habit.tiny_version}",
            ],
            'building_streak': [
                f"Day {streak_status['current_streak'] + 1} incoming! {habit.name}: {habit.tiny_version} 💪",
                f"Keep that {streak_status['current_streak']}-day streak alive! Time for {habit.name}",
            ],
            'streak_at_risk': [
                f"Don't break your {streak_status['current_streak']}-day streak! Quick {habit.name}? {habit.tiny_version}",
                f"You've got a {streak_status['current_streak']}-day streak going! {habit.name} time! ⚡",
            ],
            'comeback': [
                f"Fresh start! Ready for {habit.name}? {habit.tiny_version} 🌟",
                f"Every day is a new chance! Let's do {habit.name}: {habit.tiny_version}",
            ]
        }

        # Determine message type
        if streak_status['current_streak'] == 0 and recent_check_ins:
            message_type = 'comeback'
        elif streak_status['current_streak'] >= 7:
            message_type = 'building_streak'
        elif streak_status['current_streak'] >= 3:
            message_type = 'streak_at_risk'
        else:
            message_type = 'new_habit'

        import random
        return random.choice(templates[message_type])

    def calculate_streak_status(self, check_ins: List[HabitCheckIn]) -> Dict:
        """Calculate current streak and related stats"""

        if not check_ins:
            return {
                'current_streak': 0,
                'longest_streak': 0,
                'at_risk': False
            }

        # Sort by date descending
        sorted_check_ins = sorted(
            check_ins,
            key=lambda x: x.check_in_date,
            reverse=True
        )

        # Calculate current streak
        current_streak = 0
        yesterday = datetime.utcnow().date() - timedelta(days=1)

        for check_in in sorted_check_ins:
            check_date = check_in.check_in_date.date()

            if check_in.completed:
                if check_date == yesterday:
                    current_streak += 1
                    yesterday -= timedelta(days=1)
                else:
                    break
            else:
                break

        # Calculate longest streak
        longest_streak = 0
        temp_streak = 0
        prev_date = None

        for check_in in sorted(check_ins, key=lambda x: x.check_in_date):
            if check_in.completed:
                if prev_date is None or (check_in.check_in_date.date() - prev_date).days == 1:
                    temp_streak += 1
                    longest_streak = max(longest_streak, temp_streak)
                else:
                    temp_streak = 1

                prev_date = check_in.check_in_date.date()
            else:
                temp_streak = 0

        return {
            'current_streak': current_streak,
            'longest_streak': longest_streak,
            'at_risk': current_streak >= 3  # Streak worth protecting
        }

    async def process_check_in(
        self,
        habit_id: int,
        response: CheckInResponse,
        conversation_text: Optional[str] = None
    ) -> Dict:
        """Process user's check-in response"""

        # Create check-in record
        check_in = HabitCheckIn(
            habit_id=habit_id,
            user_id=self.user_id,
            check_in_date=datetime.utcnow(),
            completed=response.completed,
            completion_time=datetime.utcnow() if response.completed else None,
            difficulty_rating=response.difficulty_rating,
            notes=response.notes,
            mood=response.mood
        )

        # Update habit stats
        habit = self.get_habit(habit_id)

        if response.completed:
            habit.total_completions += 1
            habit.current_streak += 1
            habit.longest_streak = max(habit.longest_streak, habit.current_streak)

            # Generate celebration
            celebration = await self.generate_celebration(habit, check_in)
        else:
            habit.current_streak = 0
            celebration = await self.generate_encouragement(habit, check_in)

        # Update completion rate
        total_check_ins = len(self.get_recent_check_ins(habit_id, days=30))
        if total_check_ins > 0:
            habit.completion_rate = (habit.total_completions / total_check_ins) * 100

        # Check for milestones
        milestones_achieved = self.check_milestones(habit)

        return {
            'check_in': check_in,
            'celebration': celebration,
            'current_streak': habit.current_streak,
            'milestones_achieved': milestones_achieved,
            'habit_stats': {
                'total_completions': habit.total_completions,
                'completion_rate': habit.completion_rate,
                'longest_streak': habit.longest_streak
            }
        }

    async def generate_celebration(
        self,
        habit: Habit,
        check_in: HabitCheckIn
    ) -> str:
        """Generate enthusiastic celebration message"""

        celebrations = [
            f"Yes! {habit.celebration or 'You did it!'} 🎉",
            f"Boom! Day {habit.current_streak} in the books! {habit.celebration or '⭐'}",
            f"Fantastic! {habit.name} ✓ Keep that momentum!",
            f"You're on fire! 🔥 {habit.current_streak} days strong!"
        ]

        # Special celebrations for milestones
        if habit.current_streak == 7:
            return f"🎊 ONE WEEK STREAK! You've done {habit.name} for 7 days straight! Incredible!"
        elif habit.current_streak == 30:
            return f"🏆 30-DAY MILESTONE! You've officially made {habit.name} a habit!"
        elif habit.current_streak == 100:
            return f"💯 CENTURY CLUB! 100 days of {habit.name}! You're unstoppable!"

        import random
        return random.choice(celebrations)

    async def generate_encouragement(
        self,
        habit: Habit,
        check_in: HabitCheckIn
    ) -> str:
        """Generate supportive message for missed day"""

        encouragements = [
            "No worries! Tomorrow is a fresh start. What made today challenging?",
            "That's okay! Every setback is a setup for a comeback. You've got this! 💪",
            "Life happens! The important thing is you're still committed. Ready to try again tomorrow?",
            "Missing one day doesn't erase your progress. Let's talk about what got in the way."
        ]

        import random
        return random.choice(encouragements)

    def get_recent_check_ins(self, habit_id: int, days: int = 7) -> List[HabitCheckIn]:
        """Get recent check-ins for a habit"""
        # Database query
        pass

    def get_habit(self, habit_id: int) -> Habit:
        """Get habit by ID"""
        # Database query
        pass

    def check_milestones(self, habit: Habit) -> List[HabitMilestone]:
        """Check if any milestones were just achieved"""
        # Query and check milestones
        pass

    async def send_notification(self, user_id: str, message: str, habit_id: int):
        """Send notification via user's preferred channel"""
        # Implement notification sending
        pass

    def log_reminder(self, habit_id: int):
        """Log that reminder was sent"""
        # Update database
        pass
```

## Conversational Habit Coaching

### Habit Coach

```python
import openai

class HabitCoach:
    """Conversational AI for habit support"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def handle_obstacle(
        self,
        habit: Habit,
        obstacle_description: str,
        conversation_history: List[Dict] = None
    ) -> str:
        """Help user overcome obstacles"""

        system_prompt = f"""You are a supportive habit coach helping someone with: {habit.name}

Their tiny habit: {habit.tiny_version}
Current streak: {habit.current_streak} days

They're facing an obstacle. Your job:
1. Acknowledge the challenge
2. Ask clarifying questions
3. Suggest practical solutions
4. Remind them it's about progress, not perfection
5. Help them plan for tomorrow

Be warm, practical, and concise (2-3 sentences)."""

        messages = [{"role": "system", "content": system_prompt}]

        if conversation_history:
            messages.extend(conversation_history)

        messages.append({"role": "user", "content": obstacle_description})

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=messages,
            temperature=0.7
        )

        return response.choices[0].message.content

    def provide_implementation_intention(self, habit: Habit, context: str = "") -> str:
        """Help create if-then plans for habit success"""

        prompt = f"""Create an implementation intention (if-then plan) for this habit:

Habit: {habit.name}
Tiny version: {habit.tiny_version}
Trigger: {habit.trigger}
Context: {context}

Format: "If [SITUATION], then I will [SPECIFIC ACTION]."

Create 2-3 if-then plans that help ensure habit completion, including:
1. Primary plan for ideal conditions
2. Backup plan for obstacles
3. Recovery plan if missed

Make them specific and actionable."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def analyze_failure_patterns(
        self,
        check_ins: List[HabitCheckIn]
    ) -> Dict:
        """Identify patterns in failed attempts"""

        failed_check_ins = [ci for ci in check_ins if not ci.completed]

        if not failed_check_ins:
            return {'message': 'No failures to analyze! You're doing great!'}

        # Analyze patterns
        failure_patterns = {
            'days_of_week': {},
            'common_obstacles': [],
            'difficulty_trends': []
        }

        for check_in in failed_check_ins:
            # Day of week
            day = check_in.check_in_date.strftime('%A')
            failure_patterns['days_of_week'][day] = \
                failure_patterns['days_of_week'].get(day, 0) + 1

            # Obstacles
            if check_in.obstacles:
                failure_patterns['common_obstacles'].extend(check_in.obstacles)

        # Generate insights
        insights = []

        # Most common failure day
        if failure_patterns['days_of_week']:
            worst_day = max(
                failure_patterns['days_of_week'].items(),
                key=lambda x: x[1]
            )[0]
            insights.append(f"You often miss on {worst_day}s. Let's plan for that.")

        return {
            'patterns': failure_patterns,
            'insights': insights
        }
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends, BackgroundTasks
from sqlalchemy.orm import Session

app = FastAPI(title="Habit Formation Companion")

tiny_habits_builder = TinyHabitsBuilder(api_key=os.getenv("OPENAI_API_KEY"))
habit_coach = HabitCoach(api_key=os.getenv("OPENAI_API_KEY"))

@app.post("/habits/create")
async def create_habit(
    habit_data: HabitCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Create a new habit with AI assistance"""

    # Design tiny habit
    tiny_habit_design = tiny_habits_builder.design_tiny_habit(
        f"{habit_data.name}: {habit_data.description}"
    )

    # Create habit
    habit = Habit(
        user_id=user_id,
        name=habit_data.name,
        description=habit_data.description,
        category=HabitCategory[habit_data.category.upper()],
        frequency=HabitFrequency[habit_data.frequency.upper()],
        reminder_time=habit_data.reminder_time,
        trigger=tiny_habit_design.get('anchor') or habit_data.trigger,
        tiny_version=tiny_habit_design.get('tiny_behavior') or habit_data.tiny_version,
        celebration=tiny_habit_design.get('celebration')
    )

    db.add(habit)
    db.commit()

    # Create milestones
    milestones = [
        HabitMilestone(
            habit_id=habit.id,
            milestone_type="streak",
            threshold=7,
            reward_message="7-day streak! You're building momentum!"
        ),
        HabitMilestone(
            habit_id=habit.id,
            milestone_type="streak",
            threshold=30,
            reward_message="30-day streak! This is officially a habit!"
        ),
        HabitMilestone(
            habit_id=habit.id,
            milestone_type="total_completions",
            threshold=100,
            reward_message="100 completions! You're in the Century Club!"
        )
    ]

    for milestone in milestones:
        db.add(milestone)

    db.commit()

    return {
        'habit': habit,
        'tiny_habit_design': tiny_habit_design,
        'milestones': milestones
    }

@app.post("/habits/{habit_id}/check-in")
async def check_in(
    habit_id: int,
    response: CheckInResponse,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Process daily check-in"""

    manager = CheckInManager(user_id)
    result = await manager.process_check_in(habit_id, response)

    return result

@app.get("/habits/today")
async def get_todays_habits(
    user_id: str,
    db: Session = Depends(get_db)
):
    """Get all habits due today"""

    habits = db.query(Habit).filter(
        Habit.user_id == user_id,
        Habit.is_active == True
    ).all()

    # Check which ones are due today
    today_habits = []
    for habit in habits:
        # Check frequency logic
        # Add to today_habits if due today
        today_habits.append({
            'habit': habit,
            'completed_today': False,  # Check database
            'streak': habit.current_streak
        })

    return today_habits
```

## Bonus Challenges

### Challenge 1: Habit Buddy System
Match users with accountability partners.

### Challenge 2: Gamification
Add levels, XP, achievements.

### Challenge 3: Smart Scheduling
AI suggests optimal times based on calendar.

### Challenge 4: Wearable Integration
Track habits via Fitbit, Apple Watch.

### Challenge 5: Habit Journal
Auto-generate weekly reflection journals.

## Resources

### Books
- "Atomic Habits" by James Clear
- "Tiny Habits" by BJ Fogg
- "The Power of Habit" by Charles Duhigg

### Papers
- "Habit Formation" (Lally et al., 2010)
- "Implementation Intentions" (Gollwitzer, 1999)

## Success Criteria

### Features
- [ ] Tiny habit design assistance
- [ ] Daily check-in system
- [ ] Streak tracking
- [ ] Adaptive difficulty
- [ ] Milestone celebrations
- [ ] Obstacle coaching

### Engagement
- [ ] Check-in completion rate > 70%
- [ ] Average streak length > 7 days
- [ ] User retention > 30 days
- [ ] Positive user feedback

This system helps users build lasting habits through behavioral science and AI coaching!
