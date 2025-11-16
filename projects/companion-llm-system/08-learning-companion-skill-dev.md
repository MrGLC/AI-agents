# Project 08: Personalized Learning Companion for Skill Development

## Overview
Build an AI learning companion that creates personalized learning paths, adapts to individual learning styles, provides explanations at the right level, tracks progress, and uses spaced repetition to ensure long-term retention. The system acts as a personal tutor across any subject or skill.

## Learning Objectives
- Design adaptive learning systems
- Implement spaced repetition algorithms
- Build personalized curriculum generation
- Create interactive teaching conversations
- Develop learning analytics and progress tracking
- Apply learning science principles (Bloom's taxonomy, zone of proximal development)

## Difficulty Level
**Advanced** - Requires educational psychology knowledge, adaptive algorithms, and curriculum design

## Technical Stack

### Core Technologies
- **LLM**: OpenAI GPT-4 or Anthropic Claude
- **Backend**: Python with FastAPI
- **Database**: PostgreSQL for learning data
- **Spaced Repetition**: Custom SM-2 algorithm implementation
- **Knowledge Graphs**: Neo4j for concept relationships
- **Analytics**: pandas, scikit-learn
- **Frontend**: React with interactive exercises

### Libraries
```python
# requirements.txt
openai==1.12.0
anthropic==0.18.1
fastapi==0.109.2
uvicorn==0.27.1
sqlalchemy==2.0.25
neo4j==5.16.0
pydantic==2.6.1
pandas==2.1.4
numpy==1.26.3
scikit-learn==1.4.0
python-dateutil==2.8.2
```

## Data Model and Architecture

### Learning System Schema

```python
from sqlalchemy import Column, String, DateTime, Integer, JSON, Float, Boolean, ForeignKey, Enum
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime, timedelta
from typing import List, Dict, Optional
from pydantic import BaseModel
import enum

Base = declarative_base()

class LearningGoal(Base):
    """High-level learning goals"""
    __tablename__ = "learning_goals"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    skill_name = Column(String, nullable=False)  # "Python Programming", "Spanish"
    description = Column(String)
    target_proficiency = Column(String)  # "beginner", "intermediate", "advanced"

    # Timeline
    created_at = Column(DateTime, default=datetime.utcnow)
    target_date = Column(DateTime, nullable=True)
    estimated_hours = Column(Float)

    # Status
    current_level = Column(String, default="beginner")
    progress_percentage = Column(Float, default=0.0)
    is_active = Column(Boolean, default=True)

    # Relationships
    concepts = relationship("Concept", back_populates="learning_goal")
    learning_sessions = relationship("LearningSession", back_populates="learning_goal")

class Concept(Base):
    """Individual concepts to learn"""
    __tablename__ = "concepts"

    id = Column(Integer, primary_key=True)
    learning_goal_id = Column(Integer, ForeignKey("learning_goals.id"))
    name = Column(String, nullable=False)
    description = Column(String)

    # Learning structure
    difficulty_level = Column(Integer)  # 1-10
    prerequisites = Column(JSON)  # List of concept IDs
    estimated_time = Column(Integer)  # minutes

    # Bloom's taxonomy level
    cognitive_level = Column(String)  # remember, understand, apply, analyze, evaluate, create

    # Progress
    mastery_level = Column(Float, default=0.0)  # 0-1
    times_practiced = Column(Integer, default=0)
    last_reviewed = Column(DateTime, nullable=True)
    next_review = Column(DateTime, nullable=True)  # Spaced repetition

    # Content
    explanation = Column(String)
    examples = Column(JSON)
    exercises = Column(JSON)

    learning_goal = relationship("LearningGoal", back_populates="concepts")
    practice_attempts = relationship("PracticeAttempt", back_populates="concept")

class LearningSession(Base):
    """Individual learning sessions"""
    __tablename__ = "learning_sessions"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, index=True)
    learning_goal_id = Column(Integer, ForeignKey("learning_goals.id"))
    session_date = Column(DateTime, default=datetime.utcnow)

    # Session details
    duration = Column(Integer)  # seconds
    concepts_covered = Column(JSON)  # List of concept IDs
    activities = Column(JSON)  # ["explanation", "practice", "quiz"]

    # Outcomes
    concepts_mastered = Column(JSON)
    difficulty_rating = Column(Integer, nullable=True)  # 1-5
    engagement_score = Column(Float, nullable=True)
    notes = Column(String)

    learning_goal = relationship("LearningGoal", back_populates="learning_sessions")

class PracticeAttempt(Base):
    """Track practice attempts for spaced repetition"""
    __tablename__ = "practice_attempts"

    id = Column(Integer, primary_key=True)
    concept_id = Column(Integer, ForeignKey("concepts.id"))
    user_id = Column(String, index=True)
    attempt_date = Column(DateTime, default=datetime.utcnow)

    # Performance
    correct = Column(Boolean)
    confidence = Column(Integer)  # 1-5
    time_taken = Column(Integer)  # seconds
    hint_used = Column(Boolean, default=False)

    # Spaced repetition data
    ease_factor = Column(Float, default=2.5)  # SM-2 algorithm
    interval = Column(Integer, default=1)  # days until next review
    repetitions = Column(Integer, default=0)

    concept = relationship("Concept", back_populates="practice_attempts")

class LearningStyle(Base):
    """User's learning style profile"""
    __tablename__ = "learning_styles"

    id = Column(Integer, primary_key=True)
    user_id = Column(String, unique=True, index=True)

    # VARK model
    visual_score = Column(Float, default=0.5)
    auditory_score = Column(Float, default=0.5)
    reading_score = Column(Float, default=0.5)
    kinesthetic_score = Column(Float, default=0.5)

    # Preferences
    preferred_session_length = Column(Integer, default=30)  # minutes
    preferred_time_of_day = Column(String, nullable=True)
    pace_preference = Column(String, default="moderate")  # slow, moderate, fast

    # Adaptive data
    attention_span = Column(Integer, default=25)  # minutes
    optimal_difficulty = Column(Float, default=0.7)  # 0-1, sweet spot for challenge

# Pydantic models
class LearningGoalCreate(BaseModel):
    skill_name: str
    description: Optional[str] = None
    target_proficiency: str = "intermediate"
    target_date: Optional[datetime] = None

class PracticeResponse(BaseModel):
    correct: bool
    confidence: int  # 1-5
    time_taken: int
    answer: str
```

## Personalized Curriculum Generator

### Adaptive Learning Path Builder

```python
import openai
from typing import List, Dict, Set
import networkx as nx

class CurriculumGenerator:
    """Generate personalized learning curricula"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)
        self.concept_graph = nx.DiGraph()

    def generate_learning_path(
        self,
        goal: LearningGoal,
        user_profile: Dict
    ) -> List[Dict]:
        """Generate complete learning path for a goal"""

        prompt = f"""Create a comprehensive learning path for: {goal.skill_name}

Target level: {goal.target_proficiency}
User background: {user_profile.get('background', 'beginner')}
Available time: {goal.estimated_hours} hours

Create a structured learning path with:
1. Core concepts (ordered by prerequisite)
2. Estimated time for each
3. Difficulty level (1-10)
4. Learning activities
5. Milestone checkpoints

Return as JSON:
{{
    "path_name": "...",
    "total_concepts": 20,
    "estimated_weeks": 8,
    "concepts": [
        {{
            "name": "Variables and Data Types",
            "difficulty": 2,
            "estimated_hours": 3,
            "prerequisites": [],
            "key_topics": ["...", "..."],
            "practice_exercises": 5,
            "milestone": false
        }},
        ...
    ]
}}"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        import json
        return json.loads(response.choices[0].message.content)

    def adapt_difficulty(
        self,
        concept: Concept,
        recent_performance: List[PracticeAttempt]
    ) -> str:
        """Adapt explanation difficulty based on performance"""

        if not recent_performance:
            return "intermediate"

        # Calculate recent success rate
        recent_correct = sum(1 for attempt in recent_performance[-5:] if attempt.correct)
        success_rate = recent_correct / min(5, len(recent_performance))

        # Adjust difficulty
        if success_rate >= 0.8:
            return "advanced"  # Learner is ready for more challenge
        elif success_rate >= 0.5:
            return "intermediate"  # Just right
        else:
            return "beginner"  # Need more support

    def generate_explanation(
        self,
        concept: Concept,
        difficulty: str,
        learning_style: LearningStyle
    ) -> Dict:
        """Generate explanation adapted to user's level and style"""

        # Determine preferred modality
        modalities = {
            'visual': learning_style.visual_score,
            'auditory': learning_style.auditory_score,
            'reading': learning_style.reading_score,
            'kinesthetic': learning_style.kinesthetic_score
        }
        primary_modality = max(modalities, key=modalities.get)

        modality_instructions = {
            'visual': "Include visual descriptions, diagrams, and imagery",
            'auditory': "Use conversational tone, analogies, and verbal explanations",
            'reading': "Provide detailed written explanations with definitions",
            'kinesthetic': "Include hands-on examples and practical exercises"
        }

        prompt = f"""Explain this concept: {concept.name}

Description: {concept.description}
Difficulty level: {difficulty}
Learning style: {primary_modality} - {modality_instructions[primary_modality]}

Provide:
1. Core explanation (3-4 paragraphs)
2. Real-world analogy
3. 2-3 concrete examples
4. Common misconceptions to avoid
5. A simple practice exercise

Make it engaging, clear, and adapted to {difficulty} level."""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return {
            'explanation': response.choices[0].message.content,
            'difficulty': difficulty,
            'modality': primary_modality
        }

    def identify_knowledge_gaps(
        self,
        goal: LearningGoal,
        mastered_concepts: List[Concept]
    ) -> List[str]:
        """Identify knowledge gaps in learning path"""

        all_concepts = goal.concepts
        mastered_ids = {c.id for c in mastered_concepts if c.mastery_level >= 0.7}

        gaps = []
        for concept in all_concepts:
            # Check if prerequisites are met
            if concept.prerequisites:
                missing_prereqs = set(concept.prerequisites) - mastered_ids
                if missing_prereqs:
                    gaps.append(f"{concept.name}: Missing prerequisites")

            # Check if concept itself needs work
            if concept.id not in mastered_ids:
                gaps.append(f"{concept.name}: Not yet mastered")

        return gaps

    def suggest_next_concept(
        self,
        goal: LearningGoal,
        completed_concepts: Set[int],
        user_energy_level: str = "normal"
    ) -> Optional[Concept]:
        """Suggest next concept to learn"""

        # Get available concepts (prerequisites met)
        available = []
        for concept in goal.concepts:
            if concept.id in completed_concepts:
                continue

            # Check prerequisites
            if not concept.prerequisites or \
               all(prereq in completed_concepts for prereq in concept.prerequisites):
                available.append(concept)

        if not available:
            return None

        # Select based on difficulty and energy level
        if user_energy_level == "low":
            # Suggest easier concepts when energy is low
            return min(available, key=lambda c: c.difficulty_level)
        elif user_energy_level == "high":
            # Challenge with harder concepts when energized
            return max(available, key=lambda c: c.difficulty_level)
        else:
            # Normal: next in sequence
            return min(available, key=lambda c: c.difficulty_level)
```

## Spaced Repetition System

### SM-2 Algorithm Implementation

```python
from datetime import datetime, timedelta
import math

class SpacedRepetitionSystem:
    """Implement spaced repetition for long-term retention"""

    def __init__(self):
        pass

    def calculate_next_review(
        self,
        attempt: PracticeAttempt,
        performance_quality: int  # 0-5
    ) -> Dict:
        """Calculate next review date using SM-2 algorithm"""

        # SM-2 algorithm parameters
        if performance_quality < 3:
            # Failed recall - reset
            new_interval = 1
            new_repetitions = 0
            new_ease = max(1.3, attempt.ease_factor - 0.2)
        else:
            # Successful recall
            if attempt.repetitions == 0:
                new_interval = 1
            elif attempt.repetitions == 1:
                new_interval = 6
            else:
                new_interval = round(attempt.interval * attempt.ease_factor)

            new_repetitions = attempt.repetitions + 1

            # Adjust ease factor
            ease_adjustment = 0.1 - (5 - performance_quality) * (0.08 + (5 - performance_quality) * 0.02)
            new_ease = max(1.3, attempt.ease_factor + ease_adjustment)

        next_review_date = datetime.utcnow() + timedelta(days=new_interval)

        return {
            'next_review': next_review_date,
            'interval': new_interval,
            'repetitions': new_repetitions,
            'ease_factor': new_ease
        }

    def get_due_concepts(
        self,
        user_id: str,
        db_session
    ) -> List[Concept]:
        """Get concepts due for review"""

        now = datetime.utcnow()

        concepts = db_session.query(Concept).filter(
            Concept.next_review <= now,
            Concept.next_review != None
        ).all()

        # Sort by urgency (overdue first)
        concepts.sort(key=lambda c: c.next_review)

        return concepts

    def calculate_retention_rate(
        self,
        practice_attempts: List[PracticeAttempt]
    ) -> float:
        """Calculate overall retention rate"""

        if not practice_attempts:
            return 0.0

        correct_count = sum(1 for attempt in practice_attempts if attempt.correct)
        return correct_count / len(practice_attempts)

    def optimize_review_schedule(
        self,
        concept: Concept,
        performance_history: List[PracticeAttempt]
    ) -> int:
        """Optimize review schedule based on performance"""

        if not performance_history:
            return 1  # Default: review tomorrow

        # Calculate recent performance
        recent = performance_history[-5:]
        avg_confidence = sum(a.confidence for a in recent) / len(recent)
        success_rate = sum(1 for a in recent if a.correct) / len(recent)

        # Adjust interval based on performance
        base_interval = concept.interval or 1

        if success_rate >= 0.9 and avg_confidence >= 4:
            # Strong performance: increase interval
            return min(base_interval * 2, 90)  # Cap at 90 days
        elif success_rate >= 0.7:
            # Good performance: maintain interval
            return base_interval
        else:
            # Struggling: reduce interval
            return max(base_interval // 2, 1)
```

## Interactive Teaching System

### Socratic Tutor

```python
class SocraticTutor:
    """Interactive AI tutor using Socratic method"""

    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def teach_concept(
        self,
        concept: Concept,
        student_question: str = None,
        conversation_history: List[Dict] = None
    ) -> str:
        """Teach using Socratic questioning"""

        system_prompt = f"""You are a Socratic tutor teaching: {concept.name}

Teaching philosophy:
1. Guide students to discover answers themselves
2. Ask leading questions rather than giving direct answers
3. Build on what they already know
4. Use analogies and examples
5. Check understanding frequently
6. Adapt to their level

Concept details:
{concept.description}

Use the Socratic method: ask questions that help the student think through the concept."""

        messages = [{"role": "system", "content": system_prompt}]

        if conversation_history:
            messages.extend(conversation_history)

        if student_question:
            messages.append({"role": "user", "content": student_question})
        else:
            messages.append({
                "role": "user",
                "content": f"I want to learn about {concept.name}. Where should I start?"
            })

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=messages,
            temperature=0.7
        )

        return response.choices[0].message.content

    def provide_feedback(
        self,
        attempt: PracticeAttempt,
        student_answer: str,
        correct_answer: str
    ) -> str:
        """Provide constructive feedback on practice attempt"""

        if attempt.correct:
            feedback_type = "reinforcement"
            prompt = f"""The student correctly answered a practice question.

Their answer: {student_answer}

Provide:
1. Enthusiastic positive reinforcement
2. Highlight what they did well
3. Suggest a slightly harder challenge
4. 2-3 sentences, encouraging tone"""
        else:
            feedback_type = "correction"
            prompt = f"""The student incorrectly answered a practice question.

Their answer: {student_answer}
Correct answer: {correct_answer}

Provide:
1. Gentle acknowledgment without discouragement
2. Identify the misconception
3. Guide toward correct understanding with a question
4. Offer to explain the concept differently
5. 3-4 sentences, supportive tone"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )

        return response.choices[0].message.content

    def generate_practice_problem(
        self,
        concept: Concept,
        difficulty: int  # 1-5
    ) -> Dict:
        """Generate practice problem for concept"""

        difficulty_descriptions = {
            1: "very easy, basic recall",
            2: "easy, simple application",
            3: "moderate, requires thinking",
            4: "challenging, complex application",
            5: "very challenging, synthesis required"
        }

        prompt = f"""Generate a practice problem for: {concept.name}

Difficulty: {difficulty}/5 ({difficulty_descriptions[difficulty]})
Concept description: {concept.description}

Create a problem that:
1. Tests understanding at the {concept.cognitive_level} level (Bloom's taxonomy)
2. Is clearly stated
3. Has one correct answer (or clearly defined solution)
4. Takes 2-5 minutes to solve
5. Includes any necessary context

Return as JSON:
{{
    "problem_text": "...",
    "correct_answer": "...",
    "explanation": "Why this is the answer...",
    "common_mistakes": ["...", "..."],
    "hints": ["Hint 1", "Hint 2"]
}}"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.8
        )

        import json
        return json.loads(response.choices[0].message.content)

    def assess_understanding(
        self,
        student_explanation: str,
        concept: Concept
    ) -> Dict:
        """Assess student's understanding from their explanation"""

        prompt = f"""Assess this student's understanding of: {concept.name}

Student's explanation:
{student_explanation}

Evaluate:
1. Accuracy (0-100%)
2. Completeness (0-100%)
3. What they understand well
4. What's missing or incorrect
5. Suggested next steps

Return as JSON:
{{
    "accuracy": 75,
    "completeness": 60,
    "strengths": ["...", "..."],
    "gaps": ["...", "..."],
    "mastery_level": 0.7,
    "next_steps": "..."
}}"""

        response = self.client.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        import json
        return json.loads(response.choices[0].message.content)
```

## Learning Analytics

### Progress Tracker

```python
import pandas as pd
import numpy as np

class LearningAnalytics:
    """Track and analyze learning progress"""

    def __init__(self, user_id: str):
        self.user_id = user_id

    def calculate_learning_velocity(
        self,
        sessions: List[LearningSession]
    ) -> float:
        """Calculate concepts mastered per hour"""

        if not sessions:
            return 0.0

        total_concepts = sum(len(s.concepts_mastered or []) for s in sessions)
        total_hours = sum(s.duration for s in sessions) / 3600

        return total_concepts / total_hours if total_hours > 0 else 0.0

    def predict_completion_date(
        self,
        goal: LearningGoal,
        current_velocity: float
    ) -> datetime:
        """Predict when goal will be completed"""

        remaining_concepts = len([
            c for c in goal.concepts
            if c.mastery_level < 0.7
        ])

        if current_velocity == 0:
            return goal.target_date or datetime.utcnow() + timedelta(days=365)

        hours_needed = remaining_concepts / current_velocity
        days_needed = hours_needed / 2  # Assume 2 hours per day

        return datetime.utcnow() + timedelta(days=days_needed)

    def identify_struggling_areas(
        self,
        concepts: List[Concept],
        attempts: List[PracticeAttempt]
    ) -> List[Dict]:
        """Identify concepts where student is struggling"""

        struggling = []

        for concept in concepts:
            concept_attempts = [
                a for a in attempts
                if a.concept_id == concept.id
            ]

            if len(concept_attempts) >= 3:
                recent_success_rate = sum(
                    1 for a in concept_attempts[-5:]
                    if a.correct
                ) / min(5, len(concept_attempts))

                if recent_success_rate < 0.5:
                    struggling.append({
                        'concept': concept.name,
                        'success_rate': recent_success_rate,
                        'attempts': len(concept_attempts),
                        'recommendation': 'Review fundamentals or get additional explanation'
                    })

        return sorted(struggling, key=lambda x: x['success_rate'])

    def generate_progress_report(
        self,
        goal: LearningGoal,
        sessions: List[LearningSession]
    ) -> str:
        """Generate comprehensive progress report"""

        total_concepts = len(goal.concepts)
        mastered_concepts = len([c for c in goal.concepts if c.mastery_level >= 0.7])
        progress_pct = (mastered_concepts / total_concepts * 100) if total_concepts > 0 else 0

        total_time = sum(s.duration for s in sessions) / 3600  # hours

        report = f"""Learning Progress Report: {goal.skill_name}

Progress: {mastered_concepts}/{total_concepts} concepts ({progress_pct:.1f}%)
Current Level: {goal.current_level}
Total Time Invested: {total_time:.1f} hours

Recent Activity:
- Sessions this week: {len([s for s in sessions[-7:]])}
- Average session length: {np.mean([s.duration/60 for s in sessions[-7:]]) if sessions else 0:.1f} minutes

Next Steps:
1. Continue working on current concepts
2. Review scheduled items for spaced repetition
3. {"On track to complete by target date" if progress_pct >= 50 else "May need to increase study time"}
"""

        return report
```

## FastAPI Implementation

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session

app = FastAPI(title="Learning Companion API")

curriculum_gen = CurriculumGenerator(api_key=os.getenv("OPENAI_API_KEY"))
srs = SpacedRepetitionSystem()
tutor = SocraticTutor(api_key=os.getenv("OPENAI_API_KEY"))

@app.post("/learning/goal")
async def create_learning_goal(
    goal_data: LearningGoalCreate,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Create new learning goal with curriculum"""

    # Create goal
    goal = LearningGoal(
        user_id=user_id,
        skill_name=goal_data.skill_name,
        description=goal_data.description,
        target_proficiency=goal_data.target_proficiency,
        target_date=goal_data.target_date
    )

    db.add(goal)
    db.commit()

    # Generate curriculum
    curriculum = curriculum_gen.generate_learning_path(goal, {})

    # Create concepts
    for concept_data in curriculum['concepts']:
        concept = Concept(
            learning_goal_id=goal.id,
            name=concept_data['name'],
            difficulty_level=concept_data['difficulty'],
            estimated_time=concept_data['estimated_hours'] * 60,
            prerequisites=concept_data.get('prerequisites', [])
        )
        db.add(concept)

    db.commit()

    return {
        'goal': goal,
        'curriculum': curriculum
    }

@app.post("/learning/practice")
async def submit_practice(
    concept_id: int,
    response: PracticeResponse,
    user_id: str,
    db: Session = Depends(get_db)
):
    """Submit practice attempt"""

    # Create attempt
    attempt = PracticeAttempt(
        concept_id=concept_id,
        user_id=user_id,
        correct=response.correct,
        confidence=response.confidence,
        time_taken=response.time_taken
    )

    db.add(attempt)
    db.commit()

    # Update spaced repetition
    next_review_data = srs.calculate_next_review(attempt, response.confidence)

    # Update concept
    concept = db.query(Concept).filter(Concept.id == concept_id).first()
    concept.next_review = next_review_data['next_review']
    concept.times_practiced += 1

    # Update mastery
    recent_attempts = db.query(PracticeAttempt).filter(
        PracticeAttempt.concept_id == concept_id
    ).order_by(PracticeAttempt.attempt_date.desc()).limit(10).all()

    mastery = sum(1 for a in recent_attempts if a.correct) / len(recent_attempts)
    concept.mastery_level = mastery

    db.commit()

    # Generate feedback
    feedback = tutor.provide_feedback(attempt, response.answer, "")

    return {
        'attempt': attempt,
        'next_review': next_review_data['next_review'],
        'mastery_level': mastery,
        'feedback': feedback
    }

@app.get("/learning/next")
async def get_next_activity(
    user_id: str,
    goal_id: int,
    db: Session = Depends(get_db)
):
    """Get next learning activity"""

    # Check for due reviews
    due_concepts = srs.get_due_concepts(user_id, db)

    if due_concepts:
        return {
            'activity_type': 'review',
            'concept': due_concepts[0],
            'reason': 'Spaced repetition review due'
        }

    # Otherwise, suggest next new concept
    goal = db.query(LearningGoal).filter(LearningGoal.id == goal_id).first()
    mastered = {c.id for c in goal.concepts if c.mastery_level >= 0.7}

    next_concept = curriculum_gen.suggest_next_concept(goal, mastered)

    return {
        'activity_type': 'learn',
        'concept': next_concept,
        'reason': 'Next in learning path'
    }
```

## Bonus Challenges

### Challenge 1: Peer Learning
Match learners for study groups.

### Challenge 2: Multi-Modal Content
Generate videos, diagrams, interactive simulations.

### Challenge 3: Assessment Generation
Auto-generate quizzes and exams.

### Challenge 4: Learning Gamification
Add XP, levels, achievements.

### Challenge 5: Expert Matching
Connect with human tutors for advanced topics.

## Resources

### Learning Science
- "Make It Stick" by Brown, Roediger, McDaniel
- "How Learning Works" by Ambrose et al.
- Bloom's Taxonomy
- Zone of Proximal Development (Vygotsky)

### Algorithms
- SM-2 Spaced Repetition
- Item Response Theory (IRT)

## Success Criteria

### Features
- [ ] Personalized learning paths
- [ ] Spaced repetition system
- [ ] Adaptive difficulty
- [ ] Interactive teaching
- [ ] Progress tracking
- [ ] Multi-modal content

### Learning Outcomes
- [ ] Improved retention rates
- [ ] Consistent progress
- [ ] Mastery achievement
- [ ] Student satisfaction

This system provides comprehensive personalized learning powered by AI and learning science!
