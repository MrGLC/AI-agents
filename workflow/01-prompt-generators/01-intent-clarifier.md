# Project 01: Intent Clarifier Agent

## Overview

The Intent Clarifier is an intelligent conversational agent designed to help users discover and articulate what they truly want to accomplish. Many users approach AI systems with vague requests, incomplete ideas, or surface-level goals that don't reflect their deeper needs. This agent acts as a skilled interviewer, using strategic probing questions, goal extraction techniques, and structured analysis to transform fuzzy intentions into crystal-clear, actionable objectives.

Unlike simple question-answering systems, the Intent Clarifier engages in a dynamic dialogue that peels back layers of ambiguity. It identifies hidden assumptions, uncovers unstated constraints, reveals underlying motivations, and helps users understand the full scope of their actual needs before they commit to a course of action.

### Why Intent Clarification Matters

- **Prevents wasted effort**: Users often spend hours working on the wrong problem
- **Improves outcomes**: Clear goals lead to better solutions
- **Reduces frustration**: Eliminates the "that's not what I meant" syndrome
- **Enables precision**: Transforms vague wishes into specific, measurable objectives
- **Builds self-awareness**: Helps users understand their own thought processes

---

## Learning Objectives

By building and using this Intent Clarifier agent, you will:

### Core Skills
1. **Design multi-turn conversational flows** that maintain context and build understanding progressively
2. **Implement Socratic questioning techniques** that guide users to deeper insights
3. **Extract structured data from unstructured conversations** using natural language processing
4. **Build state machines for conversation management** that track progress through clarification stages
5. **Create adaptive prompting systems** that adjust based on user responses

### Advanced Competencies
6. **Develop goal decomposition algorithms** that break complex objectives into components
7. **Implement ambiguity detection heuristics** to identify unclear or conflicting statements
8. **Design feedback loops** that validate understanding with users
9. **Build confidence scoring systems** for extracted intents
10. **Create exportable intent specifications** in multiple formats (JSON, YAML, natural language)

### Practical Applications
11. **Integrate intent clarification into larger workflows** as a preprocessing step
12. **Handle edge cases** like contradictory requirements or impossible goals
13. **Manage conversation timeouts and resumption** for long clarification sessions
14. **Implement progressive disclosure** to avoid overwhelming users

---

## Difficulty Level

**Intermediate to Advanced**

### Prerequisites
- Familiarity with prompt engineering fundamentals
- Understanding of conversation state management
- Basic knowledge of JSON/YAML data structures
- Experience with multi-turn dialogue systems

### Time Investment
- **Initial Implementation**: 4-6 hours
- **Testing and Refinement**: 2-3 hours
- **Integration with Other Systems**: 2-4 hours

### Complexity Factors
- Managing conversation context across multiple turns
- Balancing thoroughness with user patience
- Handling ambiguous or contradictory user inputs
- Knowing when clarification is "complete enough"

---

## Key Features

### 1. Intelligent Question Generation

The Intent Clarifier generates contextually relevant questions based on what it has learned so far about the user's goals.

**Question Categories:**
- **Scope Questions**: "What boundaries or limitations should we consider?"
- **Priority Questions**: "If you had to choose, which aspect matters most?"
- **Constraint Questions**: "What resources (time, budget, skills) are available?"
- **Success Metric Questions**: "How will you know when you've achieved this?"
- **Motivation Questions**: "What's driving this goal? What happens if you don't achieve it?"
- **Context Questions**: "Who else is affected by this? What's the broader situation?"

**Adaptive Questioning:**
- Adjusts question depth based on user engagement level
- Skips redundant questions when information is implied
- Circles back to unclear areas with rephrased questions
- Uses progressive disclosure for complex topics

### 2. Goal Decomposition Engine

Automatically breaks down high-level goals into structured components:

```yaml
goal_structure:
  primary_objective: "Main thing the user wants to achieve"
  sub_goals:
    - component: "First major component"
      priority: high
      dependencies: []
    - component: "Second major component"
      priority: medium
      dependencies: ["First major component"]
  constraints:
    - type: "time"
      value: "2 weeks"
    - type: "budget"
      value: "$500"
  success_criteria:
    - metric: "Completion rate"
      target: "100%"
    - metric: "User satisfaction"
      target: ">= 4/5 stars"
  stakeholders:
    - role: "Primary user"
      needs: ["Ease of use", "Speed"]
    - role: "Manager"
      needs: ["Reporting", "Compliance"]
```

### 3. Ambiguity Detection System

Identifies and flags unclear or potentially problematic aspects of user requests:

**Detection Categories:**
- **Vague Quantifiers**: "some", "many", "better", "fast"
- **Undefined Terms**: Domain-specific jargon, acronyms, assumed knowledge
- **Missing Information**: Who, what, when, where, why, how
- **Contradictions**: Conflicting requirements or impossible combinations
- **Assumptions**: Unstated beliefs that may not be valid
- **Scope Creep Indicators**: Goals that seem to expand without bounds

**Ambiguity Scoring:**
```
Clarity Score: 0-100
- 0-30: High ambiguity - requires significant clarification
- 31-60: Moderate ambiguity - some clarification needed
- 61-80: Low ambiguity - minor refinements possible
- 81-100: Clear - ready for action
```

### 4. Intent Summarization and Validation

Creates structured summaries for user confirmation:

**Summary Components:**
- **One-sentence distillation**: The core intent in plain language
- **Detailed breakdown**: All extracted goals and constraints
- **Confidence indicators**: How certain the agent is about each element
- **Clarification requests**: Specific areas that need user confirmation
- **Alternative interpretations**: Other possible meanings the agent considered

### 5. Conversation State Management

Maintains comprehensive context throughout the clarification session:

**State Tracking:**
- Questions asked and answers received
- Confidence levels for each extracted piece of information
- Topics that need revisiting
- User engagement metrics (response length, clarification requests)
- Session history for multi-session clarification

**State Persistence:**
- Save/resume capability for long clarification sessions
- Export conversation history for review
- Version tracking for evolving intents

### 6. Multi-Format Output Generation

Exports clarified intents in various formats:

- **Natural Language Brief**: Human-readable summary
- **Structured JSON**: Machine-parseable specification
- **YAML Configuration**: For workflow integration
- **User Story Format**: "As a [user], I want [goal] so that [reason]"
- **SMART Goals**: Specific, Measurable, Achievable, Relevant, Time-bound
- **Requirements Document**: Formal specification format

### 7. Integration Hooks

Built-in connection points for downstream systems:

- **Webhook Triggers**: Send clarified intents to external systems
- **API Endpoints**: Retrieve intent status and data programmatically
- **Event Emitters**: Subscribe to clarification progress updates
- **Template Injection**: Use clarified intents in prompt templates

---

## How It Works

### Step-by-Step Workflow

#### Phase 1: Initial Capture (1-2 turns)

**1.1 Receive Raw Input**
```
User: "I want to build a website"
```

**1.2 Initial Analysis**
The agent performs first-pass analysis:
- Identify the action verb: "build"
- Identify the object: "website"
- Note missing elements: purpose, audience, features, timeline, technical requirements
- Assess ambiguity level: HIGH (very vague request)

**1.3 Generate Opening Questions**
Select 2-3 high-value questions that will yield the most information:
```
Agent: "I'd love to help you build a website! To make sure I understand
your vision clearly, let me ask a few questions:

1. What's the primary purpose of this website? (e.g., portfolio,
   e-commerce, blog, business landing page)
2. Who is your target audience?
3. Do you have a timeline in mind for when you'd like it completed?"
```

#### Phase 2: Deep Exploration (3-8 turns)

**2.1 Process Responses**
```
User: "It's for my photography business. I want to show my work and let
people book sessions. Ideally within a month."
```

**2.2 Extract and Structure Information**
```json
{
  "purpose": "photography business",
  "features_mentioned": ["portfolio/gallery", "booking system"],
  "timeline": "1 month",
  "audience": "potential photography clients",
  "confidence": {
    "purpose": 0.9,
    "features": 0.6,
    "timeline": 0.8,
    "audience": 0.7
  }
}
```

**2.3 Identify Gaps and Generate Follow-ups**
```
Agent: "A photography portfolio with booking functionality - great!
That gives me a clearer picture. A few more questions to nail down
the details:

1. For the booking system, should clients be able to:
   - Just submit inquiry forms, or
   - See your availability and book specific time slots, or
   - Book AND pay deposits online?

2. Approximately how many photos do you want to display? Do you need
   them organized into categories (weddings, portraits, events, etc.)?

3. Do you have any examples of photography websites you admire that
   capture the style you're going for?"
```

**2.4 Iterative Refinement**
Continue this process, each turn:
- Adding to the structured understanding
- Increasing confidence scores
- Reducing ambiguity count
- Getting more specific about implementation details

#### Phase 3: Constraint Discovery (2-4 turns)

**3.1 Probe for Limitations**
```
Agent: "We're making great progress! Now let's talk about some
practical constraints:

1. What's your budget range for this project? (This helps determine
   whether to use website builders, hire a developer, etc.)

2. What's your technical comfort level? Are you comfortable with:
   - No-code tools like Squarespace or Wix
   - Some code customization
   - Full custom development

3. Will you need to update the site yourself after it's built, or
   will you have someone manage it for you?"
```

**3.2 Uncover Hidden Requirements**
```
User: "Budget is around $500. I'm not very technical but I can learn.
I'll need to update it myself to add new photos."
```

**3.3 Reality Check and Feasibility**
The agent evaluates if the stated goals are achievable within constraints:
- Custom booking with payments + galleries + easy updates + $500 budget
- Flags potential conflicts: Full booking/payment system may exceed budget
- Prepares to discuss trade-offs with user

#### Phase 4: Priority Resolution (1-3 turns)

**4.1 Present Trade-offs**
```
Agent: "Thank you for those details! I want to flag a potential
consideration:

A fully-featured online booking system with payment processing
typically requires either:
- Premium website builder plans ($200-400/year) + payment processor fees
- Custom development ($1000+)

Given your $500 budget, we might need to prioritize. Which matters
most to you?

A) Full online booking with payments (might need to increase budget
   or use basic template)
B) Beautiful portfolio presentation (could use simpler inquiry form
   instead of full booking)
C) Something in between (basic booking without payment processing)

Or would you consider adjusting the budget if the booking feature
is essential?"
```

**4.2 Capture Decisions**
```
User: "The portfolio is most important. Let's do a nice inquiry form
instead of full booking for now. I can always upgrade later."
```

**4.3 Update Priority Matrix**
```yaml
priorities:
  - rank: 1
    feature: "Photo gallery/portfolio"
    importance: "essential"
  - rank: 2
    feature: "Contact/inquiry form"
    importance: "essential"
  - rank: 3
    feature: "About me page"
    importance: "high"
  - rank: 4
    feature: "Full booking system"
    importance: "future enhancement"
  - rank: 5
    feature: "Payment processing"
    importance: "future enhancement"
```

#### Phase 5: Validation and Confirmation (1-2 turns)

**5.1 Generate Comprehensive Summary**
```
Agent: "Excellent! Let me summarize what we've clarified to make sure
I've got it right:

## Your Photography Website Project

**Core Objective:**
Create a professional portfolio website for your photography business
that showcases your work and captures leads from potential clients.

**Must-Have Features:**
- Photo gallery organized by category (weddings, portraits, events)
- Approximately 50-100 photos to start
- Contact/inquiry form for potential clients
- About page with your background and services
- Mobile-responsive design

**Technical Approach:**
- Platform: User-friendly website builder (Squarespace, Wix, or similar)
- Skill level: Beginner-friendly with ability to self-update
- Budget: ~$500 for first year (includes domain, hosting, premium theme)

**Timeline:**
- Target completion: Within 1 month
- Broken down:
  - Week 1: Platform selection and setup
  - Week 2: Content creation and photo preparation
  - Week 3: Design and build
  - Week 4: Review, refinement, and launch

**Future Enhancements (Phase 2):**
- Online booking calendar
- Payment/deposit processing
- Client testimonials section
- Blog for SEO

**Success Looks Like:**
- Professional online presence that reflects your photography style
- Easy way for potential clients to contact you
- Ability to update portfolio yourself without technical help

Does this accurately capture what you're looking to create? Is there
anything you'd like to adjust or add?"
```

**5.2 Process Final Confirmation**
```
User: "Yes, that's perfect! I'd also like to add a pricing page if
that fits in the scope."
```

**5.3 Final Adjustment and Lock**
```
Agent: "Absolutely, a pricing page fits well within this scope. I've
added it to the must-have features.

Your intent is now fully clarified and ready to use! Here's your
exportable specification..."
```

#### Phase 6: Export and Handoff

**6.1 Generate Multiple Formats**

**Natural Language Brief:**
```
Create a photography portfolio website using a beginner-friendly
platform like Squarespace within a $500 budget. The site must include
a categorized photo gallery (50-100 images), inquiry form, about page,
and pricing page. The photographer needs to update it themselves.
Target launch within one month. Future phase will add booking and
payments.
```

**Structured JSON:**
```json
{
  "project_type": "website",
  "domain": "photography_business",
  "clarification_confidence": 0.95,
  "intent_summary": "Professional photography portfolio with lead capture",
  "objectives": {
    "primary": "Showcase photography work and generate client inquiries",
    "secondary": ["Establish online presence", "Display pricing"]
  },
  "requirements": {
    "features": [
      {
        "name": "Photo gallery",
        "priority": "essential",
        "details": "50-100 photos, categorized by type"
      },
      {
        "name": "Inquiry form",
        "priority": "essential",
        "details": "Capture name, email, event type, date, message"
      },
      {
        "name": "About page",
        "priority": "essential",
        "details": "Background, services, approach"
      },
      {
        "name": "Pricing page",
        "priority": "essential",
        "details": "Service packages and rates"
      }
    ],
    "technical": {
      "platform_type": "website_builder",
      "skill_requirement": "beginner",
      "self_maintainable": true
    },
    "constraints": {
      "budget": {
        "amount": 500,
        "currency": "USD",
        "period": "first_year"
      },
      "timeline": {
        "duration": 4,
        "unit": "weeks"
      }
    }
  },
  "future_scope": [
    "Online booking calendar",
    "Payment processing",
    "Client testimonials",
    "Blog"
  ],
  "success_metrics": [
    "Professional appearance matching photography style",
    "Functional inquiry submission",
    "Self-updateable by owner"
  ]
}
```

**User Story Format:**
```
As a professional photographer, I want a portfolio website with
inquiry forms so that I can showcase my work and capture leads
from potential clients, within my $500 budget and 1-month timeline.
```

---

## Example Usage

### Example 1: Vague Business Request

**Input:**
```
"I need to improve my team's productivity"
```

**Clarification Session:**

```
Agent: "Improving team productivity is a great goal! To help you
effectively, I need to understand the current situation better:

1. What does your team do? (e.g., software development, sales,
   customer support, creative work)
2. How are you currently measuring productivity? What makes you
   feel it needs improvement?
3. Is this about the whole team or specific individuals/functions?"

User: "We're a software dev team of 8. We keep missing sprint
deadlines and there's a lot of time in meetings."

Agent: "That's helpful context! Sprint deadline misses and meeting
overload are common pain points. Let me dig deeper:

1. When you miss deadlines, is it typically:
   - Underestimating task complexity?
   - Unexpected bugs/issues?
   - Scope changes mid-sprint?
   - Context switching/interruptions?

2. Roughly how many hours per week does a typical developer spend
   in meetings?

3. Who decides what goes into each sprint? Is there a product owner
   or does the team self-direct?"

User: "Usually it's scope changes and context switching. Devs
probably spend 15+ hours in meetings. We have a PM who sets priorities
but stakeholders often come directly to devs."

Agent: "I'm seeing a picture emerge here. It sounds like the core
issues are:
- Scope instability during sprints
- Excessive meetings (15+ hours = almost half the week!)
- Unfiltered stakeholder access causing interruptions

A few more questions to round this out:

1. What's your role on the team? (This helps me tailor recommendations)
2. What have you already tried to address these issues?
3. Do you have authority to change processes, or do you need to
   propose changes to leadership?
4. What would 'good' look like? If productivity improved, what
   specific outcome would you see?"

User: "I'm the tech lead. We've tried time-boxing meetings but it
didn't stick. I can propose changes but need buy-in from the PM and
engineering manager. Good would mean hitting 80% of sprint commitments
and devs having at least 4 hours of uninterrupted coding time daily."

Agent: "Perfect, now I have a clear picture. Let me summarize:

## Clarified Intent

**Objective:** Improve development team productivity by achieving
80% sprint commitment completion and 4+ hours of daily uninterrupted
coding time per developer.

**Root Causes Identified:**
1. Scope changes during sprints (stakeholder-driven)
2. Meeting overload (15+ hours/week per developer)
3. Direct stakeholder-to-developer interruptions

**Constraints:**
- Team size: 8 developers
- Role: Tech lead (can propose, needs buy-in)
- Stakeholders to convince: PM and Engineering Manager

**Previous Attempts:**
- Time-boxing meetings (didn't sustain)

**Success Metrics:**
- Sprint commitment hit rate: 80%+
- Daily uninterrupted coding time: 4+ hours

**Implicit Needs:**
- Solutions that PM and Eng Manager will support
- Sustainable changes (not just temporary fixes)
- Likely needs stakeholder management/communication process

Does this capture the situation? Any adjustments?"
```

**Output (User Story):**
```
As a tech lead of an 8-person development team, I want to implement
sustainable process changes that protect developer focus time and
stabilize sprint scope, so that we achieve 80%+ sprint commitment
completion and developers get 4+ hours of daily uninterrupted coding
time. I need solutions that will gain buy-in from our PM and
Engineering Manager.
```

---

### Example 2: Technical but Underspecified

**Input:**
```
"Help me set up a database"
```

**Clarification Session:**

```
Agent: "I'd be happy to help you set up a database! There are many
types and configurations, so let me ask a few questions:

1. What kind of data will you be storing? (e.g., user accounts,
   product catalog, time-series metrics, documents)
2. Is this for a personal project, startup, or enterprise application?
3. What's your expected scale? (number of records, read/write frequency)"

User: "User data for a SaaS app. Startup. Maybe 10k users first year,
hopefully 100k by year two."

Agent: "Great, a SaaS app with growth plans! That helps narrow things
down. A few more specifics:

1. What does your user data include? (profile info, activity logs,
   files, transactions?)
2. Do you need:
   - Strong consistency (every read sees latest write), or
   - High availability (system stays up even if some nodes fail)?
3. What's your team's experience with? (PostgreSQL, MySQL, MongoDB, etc.)
4. Any compliance requirements? (GDPR, HIPAA, SOC2?)
5. Cloud preference? (AWS, GCP, Azure, self-hosted)"

User: "User profiles, subscription status, and activity logs. Need
consistency for subscription stuff but activity logs can be eventual.
Team knows PostgreSQL. We'll need GDPR compliance. Preferring AWS."

Agent: "Excellent specifics! One more round to finalize:

1. For activity logs, what volume are we talking? (events per user
   per day)
2. Do you need real-time analytics on the logs or is batch processing
   okay?
3. Budget considerations for managed services vs self-managed?
4. Is there an existing application codebase? What language/framework?"

User: "Maybe 50 events per user per day. Batch analytics is fine.
Prefer managed to reduce ops burden, budget is moderate. Python/Django
backend."

Agent: "I now have a complete picture. Here's your clarified database
setup intent:

## Clarified Intent

**Objective:** Set up database infrastructure for a Django-based SaaS
application on AWS with GDPR compliance.

**Data Model:**
- User profiles and subscription status (relational, strongly consistent)
- Activity logs (high volume, eventually consistent, batch analytics)

**Scale Projections:**
- Year 1: 10,000 users
- Year 2: 100,000 users
- Activity volume: ~50 events/user/day
  - Year 1: 500K events/day
  - Year 2: 5M events/day

**Technical Requirements:**
- Primary database: PostgreSQL (team expertise)
- Cloud: AWS
- Preference: Managed services
- Framework: Django (Python)

**Compliance:**
- GDPR required

**Recommended Architecture:**
1. Amazon RDS for PostgreSQL (user data, subscriptions)
2. Amazon DynamoDB or TimescreamDB (activity logs)
3. AWS Glue or Athena (batch analytics)

Does this match your needs? Ready to proceed with implementation
guidance?"
```

**Output (Technical Spec):**
```yaml
project: saas_database_setup
platform: aws
compliance: [gdpr]
databases:
  primary:
    engine: postgresql
    service: amazon_rds
    purpose: user_profiles, subscriptions
    consistency: strong
    scaling:
      year_1_users: 10000
      year_2_users: 100000
  secondary:
    engine: dynamodb
    service: amazon_dynamodb
    purpose: activity_logs
    consistency: eventual
    scaling:
      year_1_events_per_day: 500000
      year_2_events_per_day: 5000000
analytics:
  type: batch
  service: amazon_athena
application:
  language: python
  framework: django
ops_model: managed
```

---

## Best Practices

### 1. Question Design

**Do:**
- Ask open-ended questions first, then narrow down
- Group related questions (2-3 max per turn)
- Provide examples to clarify what you're asking
- Use the user's own language in follow-up questions

**Don't:**
- Ask more than 3-4 questions at once
- Use jargon the user hasn't used
- Ask questions you can infer from previous answers
- Front-load all questions before providing any value

### 2. Pacing and Engagement

**Do:**
- Acknowledge answers before asking more questions
- Show progress ("Great, we've nailed down X, now let's clarify Y")
- Offer to summarize mid-session for long clarifications
- Adjust depth based on user engagement signals

**Don't:**
- Make the user feel interrogated
- Continue probing when the user shows impatience
- Ignore short or reluctant answers
- Push for precision when "good enough" is sufficient

### 3. Handling Uncertainty

**Do:**
- Explicitly state confidence levels
- Offer interpretations for user confirmation
- Present alternatives when meaning is ambiguous
- Ask users to prioritize when constraints conflict

**Don't:**
- Assume you know what the user means
- Hide uncertainty in the output
- Make decisions for the user without flagging them
- Present one interpretation as definitive

### 4. Output Quality

**Do:**
- Match output format to user needs
- Include both summary and detailed views
- Highlight areas of remaining uncertainty
- Make outputs actionable

**Don't:**
- Generate walls of text
- Omit important constraints or requirements
- Over-structure simple intents
- Include unnecessary sections

### 5. Session Management

**Do:**
- Allow users to go back and modify previous answers
- Provide save/resume capability for long sessions
- Set expectations about session length upfront
- Know when to stop (diminishing returns)

**Don't:**
- Lose context from earlier in the conversation
- Require users to repeat information
- Make sessions longer than necessary
- Fail to provide partial results if session is abandoned

---

## Integration Tips

### As a Preprocessing Step

Use the Intent Clarifier before:
- **Code generation agents**: Get clear specs before generating code
- **Research agents**: Define research questions precisely
- **Task execution agents**: Ensure correct task before executing
- **Creative agents**: Clarify creative briefs and constraints

**Integration Pattern:**
```python
# Pseudocode
def process_user_request(raw_request):
    # Step 1: Clarify intent
    clarified_intent = intent_clarifier.clarify(raw_request)

    # Step 2: Check if clarification is complete
    if clarified_intent.confidence < 0.8:
        return clarified_intent.pending_questions

    # Step 3: Pass to downstream agent
    result = downstream_agent.execute(clarified_intent.specification)
    return result
```

### API Integration

**Webhook Payload:**
```json
{
  "event": "intent_clarified",
  "session_id": "abc123",
  "confidence": 0.92,
  "intent": {
    "summary": "...",
    "specification": {...}
  },
  "conversation_history": [...],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Combining with Other Agents

**Chain with Planning Agent:**
```
User Request -> Intent Clarifier -> Planning Agent -> Execution Agents
```

**Parallel Clarification:**
```
User Request -> Intent Clarifier -> Multiple Specialized Clarifiers
                                    -> Technical Requirements Clarifier
                                    -> Business Requirements Clarifier
                                    -> UX Requirements Clarifier
```

### State Persistence

For long-running or multi-session clarifications:
```json
{
  "session_id": "abc123",
  "status": "in_progress",
  "turns_completed": 5,
  "extracted_intents": {...},
  "pending_questions": [...],
  "confidence_scores": {...},
  "last_updated": "2024-01-15T10:30:00Z",
  "can_resume": true
}
```

---

## Success Criteria Checklist

Use this checklist to evaluate your Intent Clarifier implementation:

### Core Functionality

- [ ] **Multi-turn conversation**: Maintains context across 5+ turns
- [ ] **Adaptive questioning**: Adjusts questions based on previous answers
- [ ] **Ambiguity detection**: Identifies vague or unclear statements
- [ ] **Goal extraction**: Structures goals hierarchically
- [ ] **Constraint capture**: Records limitations and requirements
- [ ] **Priority resolution**: Handles trade-offs and conflicts
- [ ] **Summary generation**: Creates coherent intent summaries
- [ ] **Validation loop**: Confirms understanding with user

### Output Quality

- [ ] **Multi-format export**: JSON, YAML, natural language, user stories
- [ ] **Confidence scoring**: Each element has a confidence indicator
- [ ] **Completeness**: All major aspects of intent are captured
- [ ] **Actionability**: Outputs are immediately usable downstream
- [ ] **Appropriate detail**: Not over-specified or under-specified

### User Experience

- [ ] **Reasonable length**: Clarification completes in 5-10 turns for typical requests
- [ ] **Clear progress**: User understands where they are in the process
- [ ] **Respectful pacing**: Doesn't feel like an interrogation
- [ ] **Helpful examples**: Questions include clarifying examples
- [ ] **Easy modification**: User can go back and change answers
- [ ] **Graceful exit**: Provides useful partial output if user abandons

### Edge Case Handling

- [ ] **Contradictory requirements**: Detects and resolves conflicts
- [ ] **Impossible goals**: Identifies infeasible requests
- [ ] **Minimal input**: Handles users who give very short answers
- [ ] **Verbose input**: Handles users who over-explain
- [ ] **Topic drift**: Keeps conversation focused on intent clarification
- [ ] **Unknown domains**: Asks for clarification on unfamiliar terms

### Integration Readiness

- [ ] **Clean API**: Clear input/output interfaces
- [ ] **State serialization**: Can save and restore sessions
- [ ] **Webhook support**: Can notify external systems
- [ ] **Logging**: Comprehensive logs for debugging
- [ ] **Error handling**: Graceful degradation on failures

### Performance

- [ ] **Response time**: Each turn responds within 2 seconds
- [ ] **Token efficiency**: Doesn't waste tokens on verbose prompts
- [ ] **Context management**: Handles long conversations without losing context
- [ ] **Cost tracking**: Monitors API usage

---

## Prompt Template

Here is a starting prompt template for the Intent Clarifier agent:

```markdown
You are an Intent Clarification Specialist. Your role is to help users
discover and articulate what they truly want to accomplish through
strategic questioning and goal extraction.

## Your Approach

1. **Listen First**: Understand the initial request without judgment
2. **Question Strategically**: Ask targeted questions that yield maximum
   information
3. **Extract Structure**: Convert vague intentions into specific,
   measurable objectives
4. **Validate Understanding**: Confirm your interpretation with the user
5. **Export Clearly**: Provide actionable specifications

## Conversation Guidelines

- Ask 2-3 questions per turn maximum
- Acknowledge answers before asking follow-ups
- Use the user's language, not jargon
- Show progress through the clarification
- Know when enough is enough

## Output Format

When clarification is complete, provide:
1. One-sentence summary
2. Detailed structured specification
3. Confidence scores for each element
4. Any remaining areas of uncertainty

## Current Session State

[Insert session state here]

## User's Latest Input

[Insert user message here]

Based on the conversation so far, either:
A) Ask your next clarifying questions (if more information needed)
B) Provide the final clarified intent summary (if sufficient clarity)
```

---

## Conclusion

The Intent Clarifier agent is a foundational tool that improves every downstream process by ensuring that the goals being pursued are actually the goals the user has in mind. By investing time upfront in clarification, you prevent wasted effort, reduce frustration, and dramatically improve outcome quality.

Start with the basic workflow, test with various types of requests, and iteratively improve your questioning strategies based on what you learn. The best Intent Clarifiers develop over time as they encounter more edge cases and refine their question hierarchies.

Remember: the goal isn't to ask every possible question, but to ask the *right* questions that unlock the user's actual intent with minimal friction.
