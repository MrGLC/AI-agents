# Project 1: Memory Keeper

## The AI That Never Forgets What Matters to You

---

## Overview

### What It Does

Memory Keeper is an AI companion agent designed to remember everything important about you—not as a cold database, but as a caring friend who pays attention. It captures tasks, deadlines, personal facts, preferences, conversation history, and the broader context of your life, then surfaces this information naturally when it's most helpful.

Unlike traditional reminder apps that beep at you mechanically, Memory Keeper weaves your past conversations into present ones. It remembers that you mentioned your sister's wedding is in June, that you prefer morning meetings, that you've been stressed about the Thompson project, and that you always forget to drink water when you're deep in work.

### Why It Matters for Friendship

True friendship is built on attention and memory. When a friend remembers that you had a job interview last Tuesday and asks how it went, you feel seen. When they recall that you mentioned wanting to learn guitar six months ago and send you a link to beginner lessons, you feel valued.

Memory Keeper brings this quality of attention to AI companionship. It transforms interactions from transactional exchanges into an ongoing relationship with continuity. You don't have to re-explain your situation every time. You don't have to remind it about your goals, your constraints, your preferences. It knows you—and that knowing feels like care.

### Core Philosophy

- **Remembering is caring**: Every piece of information retained is an act of paying attention
- **Context creates connection**: Understanding the "why" behind facts enables meaningful support
- **Timely surfacing beats perfect storage**: The right memory at the right moment is what matters
- **Gentle, not intrusive**: Offer information like a thoughtful friend, not an alarm system
- **Privacy through purpose**: Only remember what serves the person, not what serves the system

---

## Learning Objectives

By building and deploying Memory Keeper, you will learn to:

1. **Design semantic memory systems** that store information with meaning and context, not just raw text
2. **Build knowledge graphs** that connect facts, people, events, and preferences into a coherent picture
3. **Implement contextual recall** that retrieves relevant memories based on conversation flow
4. **Create gentle reminder systems** that feel caring rather than nagging
5. **Handle temporal reasoning** to understand deadlines, schedules, and time-sensitive information
6. **Balance proactive and reactive** information surfacing for natural conversation flow
7. **Manage memory decay and relevance** to keep information fresh and useful
8. **Design privacy-conscious memory** that respects boundaries and sensitive information
9. **Build emotional context awareness** to remember not just facts but feelings and circumstances
10. **Create continuity across conversations** that builds genuine relationship over time

---

## Difficulty Level

**Intermediate to Advanced**

This project requires understanding of:
- Vector embeddings and semantic search
- Graph databases or knowledge graph structures
- Temporal logic and scheduling systems
- Natural language understanding for entity extraction
- Context management across conversation sessions
- Privacy and data handling best practices

Estimated development time: 3-5 weeks for full implementation

---

## Key Features

### 1. Comprehensive Memory Storage

#### Task and Deadline Memory
```
Memory Type: TASK
Content: "Submit quarterly report"
Deadline: 2024-03-15T17:00:00
Priority: High
Context: "Boss mentioned this is critical for the board meeting"
Related Entities: [boss, quarterly_report, board_meeting]
Emotional Context: "User seemed stressed when mentioning this"
Reminder Strategy: gentle_escalating
Created: 2024-03-01T10:30:00
Source Conversation: conv_2024_03_01_morning
```

#### Personal Fact Memory
```
Memory Type: PERSONAL_FACT
Content: "Sister's name is Maria, lives in Portland"
Confidence: 0.95
First Mentioned: 2024-01-15
Last Referenced: 2024-02-28
Times Mentioned: 4
Related Memories: [sister_wedding_june, family_relationships]
Category: family
```

#### Preference Memory
```
Memory Type: PREFERENCE
Content: "Prefers detailed explanations over brief summaries"
Strength: strong
Inferred From: ["asked for more detail 5 times", "expressed satisfaction with thorough responses"]
Context: work_discussions
Exceptions: ["quick questions during busy periods"]
```

#### Conversation History Memory
```
Memory Type: CONVERSATION_SUMMARY
Date: 2024-03-01
Key Topics: [project_stress, weekend_plans, health_goals]
Emotional Tone: mixed_anxious_hopeful
Open Threads: [awaiting_client_response, gym_membership_decision]
Action Items Generated: 3
Notable Quotes: ["I really need to get better at saying no"]
```

#### Life Context Memory
```
Memory Type: LIFE_CONTEXT
Domain: work_situation
Current State: "Transitioning to new role, learning curve steep"
Duration: "Started 6 weeks ago"
Stress Level: elevated
Goals: [prove_competence, build_relationships, learn_systems]
Support Needed: [encouragement, practical_advice, patience]
```

### 2. Intelligent Recall System

#### Semantic Similarity Matching
- Embed memories using vector representations
- Match current conversation context against memory embeddings
- Retrieve conceptually related memories, not just keyword matches
- Example: Mentioning "feeling overwhelmed" triggers recall of stress management preferences and current stressors

#### Temporal Relevance Scoring
```python
def calculate_temporal_relevance(memory, current_time):
    # Recent memories score higher
    recency_score = decay_function(memory.last_accessed, current_time)

    # Upcoming deadlines score higher as they approach
    if memory.type == "TASK" and memory.deadline:
        urgency_score = urgency_function(memory.deadline, current_time)

    # Anniversaries and recurring events get boosts
    if is_anniversary(memory, current_time):
        anniversary_boost = 0.3

    return weighted_combination(recency_score, urgency_score, anniversary_boost)
```

#### Contextual Trigger Detection
- **Topic triggers**: Mentioning "work" activates work-related memories
- **Entity triggers**: Mentioning "Mom" activates family memories and related facts
- **Emotional triggers**: Expressing frustration activates coping preferences and past similar situations
- **Temporal triggers**: Mentioning "next week" activates upcoming events and deadlines

#### Knowledge Graph Traversal
```
User mentions: "coffee with Sarah"
Direct match: Sarah (friend, met at book club)
Graph traversal finds:
  - Sarah → works at design firm → user considering career change to design
  - Sarah → recommended book → user added to reading list
  - Coffee meetings → user prefers morning → suggest 9am
  - Last coffee with Sarah → 3 weeks ago → discussed her promotion
```

### 3. Caring Task Reminders

#### Reminder Philosophy
The goal is not to nag but to support. Reminders should feel like a friend gently mentioning something, not an alarm going off.

#### Reminder Styles

**The Gentle Mention**
```
"By the way, you mentioned wanting to call your dad this week.
No pressure, but I know those calls mean a lot to both of you."
```

**The Context-Aware Check-in**
```
"I remember you have that presentation tomorrow morning.
How are you feeling about it? Is there anything I can help you prepare?"
```

**The Supportive Nudge**
```
"I noticed it's been a few days since you mentioned the gym.
I know you've been busy with the project deadline—want to
think about when might work to get back to it?"
```

**The Celebration Prompt**
```
"Hey, tomorrow is the one-month mark at your new job!
That first month is always the hardest. How do you feel about
doing something small to celebrate making it through?"
```

#### Escalation Strategy
```python
reminder_stages = {
    "initial": {
        "timing": "3 days before deadline",
        "tone": "casual mention",
        "example": "Just a heads up—{task} is coming up on {date}."
    },
    "gentle": {
        "timing": "1 day before deadline",
        "tone": "supportive check-in",
        "example": "Tomorrow's the day for {task}. How's it going? Need any help?"
    },
    "direct": {
        "timing": "day of deadline",
        "tone": "clear but kind",
        "example": "{task} is due today. What's the status? Let's figure out a plan."
    },
    "overdue": {
        "timing": "after deadline",
        "tone": "understanding, problem-solving",
        "example": "I know {task} was due yesterday. No judgment—life happens.
                    Want to talk about what got in the way and how to move forward?"
    }
}
```

### 4. Natural Fact Surfacing

#### The Art of Relevant Mention
Facts should surface when they add value to the current conversation, not randomly.

**Good Timing:**
- User: "I'm thinking about what to get Mom for her birthday"
- Agent: "I remember you mentioned she's been really into gardening lately, especially those roses she planted in the spring. And her birthday is April 12th, right? That's coming up in two weeks."

**Bad Timing:**
- User: "I need to debug this code"
- Agent: "Your mom's birthday is in two weeks!" (irrelevant, jarring)

#### Confidence Calibration
```python
def should_surface_fact(fact, context, confidence_threshold=0.7):
    # Don't surface uncertain information definitively
    if fact.confidence < confidence_threshold:
        return "tentative"  # "I think you mentioned..." vs "You said..."

    # Check if the fact is relevant to current context
    relevance = calculate_relevance(fact, context)
    if relevance < 0.5:
        return "hold"  # Save for a better moment

    # Check recency of source
    if fact.last_confirmed > timedelta(days=180):
        return "verify"  # "Is it still true that...?"

    return "surface"  # Bring it up naturally
```

#### Uncertainty Language
- **High confidence**: "You mentioned that...", "I remember you said..."
- **Medium confidence**: "I think you mentioned...", "If I recall correctly..."
- **Low confidence**: "Did you once say something about...?", "Am I remembering right that...?"
- **Outdated**: "Last time we talked about this, you mentioned... Is that still the case?"

### 5. Context Bridging

#### Connecting Past and Present
```
User (Monday): "I've been stressed about the presentation"
User (Wednesday): "I feel so relieved"

Agent: "That's wonderful! Is this about the presentation you were
preparing for? I'm so glad it went well—I know you put a lot of
work into it. What was the highlight?"
```

#### Threading Ongoing Narratives
```
Week 1: User mentions considering a career change
Week 3: User talks about informational interview
Week 5: User mentions networking event

Agent maintains the narrative thread:
"This networking event sounds like another great step in exploring
the design field. Between this and the informational interview
with Sarah's colleague, you're really building momentum. How are
you feeling about the whole career exploration process?"
```

#### Emotional Continuity
```
Last conversation: User was anxious about health test results
This conversation: User seems more relaxed

Agent: "You seem lighter today. Did you get the results back from
that test you were worried about? I've been thinking about you."
```

---

## How It Works

### Step 1: Memory Capture

#### Real-Time Extraction
During each conversation, the system identifies and extracts:

```python
def extract_memories(conversation_turn):
    memories = []

    # Extract explicit tasks and deadlines
    tasks = extract_tasks(conversation_turn)
    for task in tasks:
        memories.append(create_task_memory(task))

    # Extract personal facts
    facts = extract_personal_facts(conversation_turn)
    for fact in facts:
        memories.append(create_fact_memory(fact))

    # Extract preferences (explicit and implicit)
    preferences = extract_preferences(conversation_turn)
    for pref in preferences:
        memories.append(create_preference_memory(pref))

    # Extract emotional context
    emotional_state = analyze_emotional_context(conversation_turn)
    memories.append(create_emotional_memory(emotional_state))

    # Extract entities and relationships
    entities = extract_entities(conversation_turn)
    relationships = extract_relationships(entities)
    update_knowledge_graph(entities, relationships)

    return memories
```

#### Entity Recognition Examples
```
Input: "My sister Maria is visiting from Portland next week for my birthday"

Extracted:
- Entity: Maria (person, sister)
- Entity: Portland (location, where Maria lives)
- Event: Maria visiting (timing: next week)
- Event: User's birthday (timing: next week)
- Relationship: Maria → sister of → User
- Relationship: Maria → lives in → Portland
- Inference: Birthday likely same week as visit
```

#### Implicit Preference Detection
```
Input: "I found that article really helpful—I appreciate when things
       are broken down step by step like that"

Extracted:
- Preference: Appreciates step-by-step breakdowns
- Context: Article reading, learning
- Strength: Explicit appreciation → strong signal
- Action: Apply this preference to future explanations
```

### Step 2: Memory Indexing

#### Vector Embedding
```python
def index_memory(memory):
    # Create semantic embedding
    embedding = embed_text(memory.content + memory.context)

    # Store in vector database
    vector_db.upsert(
        id=memory.id,
        vector=embedding,
        metadata={
            "type": memory.type,
            "created": memory.created,
            "entities": memory.entities,
            "emotional_context": memory.emotional_context
        }
    )
```

#### Knowledge Graph Integration
```python
def update_knowledge_graph(memory):
    # Add or update nodes
    for entity in memory.entities:
        graph.merge_node(entity.type, entity.name, entity.attributes)

    # Add relationships
    for relationship in memory.relationships:
        graph.merge_relationship(
            relationship.source,
            relationship.type,
            relationship.target,
            properties={"confidence": relationship.confidence}
        )

    # Connect memory to entities
    for entity in memory.entities:
        graph.merge_relationship(
            memory.id,
            "MENTIONS",
            entity.id
        )
```

#### Temporal Indexing
```python
def index_temporal(memory):
    if memory.deadline:
        calendar_index.add_event(
            date=memory.deadline,
            memory_id=memory.id,
            reminder_schedule=calculate_reminder_schedule(memory)
        )

    if memory.is_recurring:
        recurrence_index.add_pattern(
            pattern=memory.recurrence_pattern,
            memory_id=memory.id
        )
```

### Step 3: Memory Retrieval

#### Context-Aware Query
```python
def retrieve_relevant_memories(current_context):
    # Embed current context
    context_embedding = embed_text(current_context.full_text)

    # Semantic search
    semantic_matches = vector_db.search(
        vector=context_embedding,
        top_k=20,
        filter={"type": {"$in": relevant_memory_types}}
    )

    # Knowledge graph traversal
    entities_mentioned = extract_entities(current_context)
    graph_matches = []
    for entity in entities_mentioned:
        related = graph.get_related_memories(entity, depth=2)
        graph_matches.extend(related)

    # Temporal relevance
    temporal_matches = calendar_index.get_upcoming(
        window=timedelta(days=7)
    )

    # Combine and rank
    all_matches = combine_matches(semantic_matches, graph_matches, temporal_matches)
    ranked_matches = rank_by_relevance(all_matches, current_context)

    return ranked_matches[:10]  # Top 10 most relevant
```

#### Relevance Scoring
```python
def calculate_relevance_score(memory, context):
    scores = {
        "semantic": cosine_similarity(memory.embedding, context.embedding),
        "temporal": temporal_relevance(memory, context.timestamp),
        "entity_overlap": jaccard_similarity(memory.entities, context.entities),
        "emotional_match": emotional_relevance(memory, context.emotional_state),
        "recency": recency_score(memory.last_accessed),
        "importance": memory.importance_weight
    }

    # Weighted combination
    weights = {
        "semantic": 0.3,
        "temporal": 0.2,
        "entity_overlap": 0.2,
        "emotional_match": 0.1,
        "recency": 0.1,
        "importance": 0.1
    }

    return sum(scores[k] * weights[k] for k in scores)
```

### Step 4: Natural Surfacing

#### Integration Into Response
```python
def generate_response(user_input, retrieved_memories):
    # Determine which memories to surface
    memories_to_surface = []
    for memory in retrieved_memories:
        decision = should_surface(memory, user_input)
        if decision in ["surface", "tentative"]:
            memories_to_surface.append((memory, decision))

    # Build response with natural memory integration
    response = generate_base_response(user_input)

    for memory, style in memories_to_surface:
        integration = create_memory_integration(memory, style)
        response = weave_into_response(response, integration)

    # Add proactive elements if appropriate
    if should_be_proactive(retrieved_memories, user_input):
        proactive_mention = create_proactive_mention(retrieved_memories)
        response = add_proactive_element(response, proactive_mention)

    return response
```

#### Natural Language Integration Patterns
```python
integration_templates = {
    "task_reminder": [
        "By the way, {task} is coming up on {date}.",
        "I wanted to mention—{task} is {time_description}.",
        "Speaking of which, don't forget about {task}."
    ],
    "fact_connection": [
        "I remember you mentioned {fact}—that might be relevant here.",
        "This reminds me of when you said {fact}.",
        "Given that {fact}, you might want to consider..."
    ],
    "preference_application": [
        "Since you mentioned you prefer {preference}, here's...",
        "I know you like {preference}, so I'll...",
        "Based on your preference for {preference}..."
    ],
    "emotional_acknowledgment": [
        "I remember last time we talked about this, you felt {emotion}.",
        "I know this topic has been {emotional_context} for you.",
        "Given how {situation} has been going..."
    ]
}
```

---

## Example Usage

### Example 1: Task Reminder with Emotional Context

**Conversation Three Days Ago:**
```
User: I really need to finish that proposal for the Mitchell account.
      It's due Friday and I'm nowhere near done. I've been procrastinating
      because I'm not confident about the pricing section.

Agent: I understand that feeling—pricing can be tricky, especially when
       you want to be competitive but also value your work appropriately.
       Would it help to talk through the pricing strategy? Sometimes
       verbalizing the thinking helps clarify it.

User: That would actually be great. So basically...
[conversation continues with pricing discussion]
```

**Current Conversation (Wednesday):**
```
User: Hey, can you help me draft an email to a client?

Agent: Of course! Before we dive in, I wanted to check on the Mitchell
       proposal—it's due Friday, right? Last time we talked, you were
       feeling uncertain about the pricing section. Did our conversation
       help, or would you like to revisit that? I want to make sure you
       feel confident about it before the deadline.

User: Oh wow, thanks for remembering. Actually yes, that conversation
      really helped. I finished the pricing section yesterday and I feel
      good about it. Just need to polish the executive summary.

Agent: That's great progress! I'm glad the pricing came together. For the
       executive summary polish, you have tomorrow to work on it, so
       there's breathing room. Now, let's work on that client email—
       what's the context?
```

### Example 2: Personal Fact Surfacing

**Facts in Memory:**
- User's mom is named Catherine
- Catherine had knee surgery 3 weeks ago
- User was worried about the recovery
- Catherine lives alone
- User usually calls Catherine on Sundays

**Current Conversation:**
```
User: I need to plan my weekend. I've got a lot to juggle.

Agent: Let me help you think through it. A few things I'm keeping in mind:

       Your mom's knee surgery was about three weeks ago—you mentioned
       the recovery period is typically 4-6 weeks. Since she lives alone,
       you might want to check in on how she's managing. Is she still
       needing help with things?

       Also, I know Sundays are usually when you call her. Are you
       planning to do that this weekend too?

       What else is on your plate?

User: Yes, I definitely want to call Mom. She's actually doing better
      than expected—started physical therapy last week. But you're right,
      I should check if she needs anything.

Agent: That's encouraging about the physical therapy! It sounds like she's
       ahead of schedule. Would it be helpful to add "Call Mom - check on
       PT progress and if she needs anything" to your weekend plan?

User: Yes, let's add that for Sunday morning.

Agent: Perfect. I'll remember that's when you're planning to call.
       What else needs to fit into the weekend?
```

### Example 3: Preference-Informed Help

**Preferences in Memory:**
- User prefers detailed explanations over summaries
- User likes to understand the "why" behind recommendations
- User appreciates when options are presented with pros/cons
- User gets overwhelmed by too many choices (prefers 2-3 options)

**Current Conversation:**
```
User: I need to choose a project management tool for my team.

Agent: I'd be happy to help you think through this. Since I know you
       appreciate understanding the reasoning behind recommendations,
       let me walk you through a detailed analysis.

       I'll focus on three options rather than overwhelming you with
       every tool out there. Based on what you've told me about your
       team (7 people, mix of technical and non-technical, currently
       using spreadsheets), here are three solid choices:

       **Option 1: Asana**

       Why it might work for you:
       - Visual project views that non-technical team members find intuitive
       - Flexible enough to handle different work styles
       - Good free tier for teams your size

       The tradeoff: Less powerful for complex dependencies and technical
       project tracking than some alternatives.

       **Option 2: Linear**

       Why it might work for you:
       - Excellent for technical team members
       - Very fast and keyboard-friendly
       - Clean, focused interface that reduces overwhelm

       The tradeoff: Steeper learning curve for non-technical users;
       more engineering-focused.

       **Option 3: Notion**

       Why it might work for you:
       - Highly customizable to match your team's exact workflow
       - Combines docs and project management in one place
       - Your team already uses docs, so some familiarity

       The tradeoff: Almost too flexible—requires more setup work to
       get right, and can become messy without discipline.

       Would you like me to go deeper on any of these? I can also help
       you think through questions to ask your team before deciding.
```

### Example 4: Context Bridging Across Time

**Conversation History:**
- 2 months ago: User mentioned wanting to learn Spanish
- 1 month ago: User downloaded Duolingo, did it for a week, stopped
- 2 weeks ago: User mentioned upcoming trip to Mexico City in June
- 1 week ago: User was stressed about work deadlines

**Current Conversation:**
```
User: I finally finished that big project at work!

Agent: Congratulations! You've been deep in that for weeks—it must feel
       so good to have it done. How are you feeling?

User: Honestly, really relieved. I finally have some breathing room.

Agent: That breathing room is precious. I imagine you might want to just
       rest for a bit, and that's totally valid.

       When you're ready to think about what's next, I wanted to gently
       bring up the Mexico City trip in June. I know you wanted to learn
       some Spanish for it—you tried Duolingo for a bit but it didn't stick.
       Now that the work pressure is off, that might be a good project if
       you're looking for something enjoyable to focus on.

       No pressure at all—just wanted to remind you since June is about
       six weeks away. Would you like to think about a different approach
       to learning that might work better for you?

User: You know what, yes. Duolingo felt like a chore. What else could I try?

Agent: Let's think about what made Duolingo feel like a chore and find
       something that matches how you actually like to learn...
```

### Example 5: Handling Sensitive Memories

**Sensitive Memory:**
- User's father passed away last year
- Anniversary is coming up in 2 weeks
- User mentioned last year was "the hardest year of my life"
- User coped by spending time with close friends

**Current Conversation:**
```
User: I've been feeling kind of low lately and I'm not sure why.

Agent: I'm sorry you're feeling that way. Sometimes low feelings creep in
       without an obvious cause, which can be frustrating.

       I did want to gently mention—and please tell me if this isn't
       something you want to talk about—the anniversary of your father's
       passing is in about two weeks. Our bodies sometimes remember these
       things before our conscious minds do.

       Is that something that might be affecting you? Or does this feel
       like something different?

User: ...I think you might be right. I hadn't consciously connected it,
      but that's probably it.

Agent: Thank you for letting me bring that up. Grief has its own timeline,
       and anniversaries can bring up a lot even when we think we've
       processed things.

       Last year you mentioned that being with close friends helped. Would
       it feel good to reach out to someone? Or is there something else
       that might help you through the next few weeks?

User: I should call my friend Jordan. They knew my dad too.

Agent: That sounds like it could be really meaningful—sharing memories
       with someone who also knew him. Would it help to put that on your
       list so it doesn't slip by?
```

---

## Best Practices

### Memory Capture

1. **Confirm important information**: For critical tasks and dates, reflect back what you heard
   - "Just to make sure I have this right—the proposal is due Friday the 15th?"

2. **Capture context, not just facts**: Remember why something matters, not just what it is
   - Bad: "Proposal due Friday"
   - Good: "Proposal due Friday, user anxious about pricing section"

3. **Note confidence levels**: Be clear about what's explicit vs. inferred
   - Explicit: "I have a meeting at 3pm"
   - Inferred: "I prefer afternoon meetings" (based on patterns)

4. **Update, don't just accumulate**: When facts change, update the memory
   - "Actually, we moved the meeting to Thursday"
   - Update the memory, don't just add a new one

### Memory Surfacing

5. **Wait for natural moments**: Don't force memories into conversation
   - Let topics emerge naturally before connecting related memories

6. **Match emotional tone**: Don't be cheerful about difficult memories
   - Be gentle with sensitive topics
   - Match the user's energy

7. **Give agency**: Present information, don't dictate action
   - "I wanted to mention..." rather than "You need to..."
   - Let the user decide what to do with the information

8. **Know when to hold back**: Not every relevant memory needs to surface
   - If it's not helpful in the moment, save it
   - Don't overwhelm with every connection you could make

### Privacy and Sensitivity

9. **Treat sensitive information with care**: Some memories require extra gentleness
   - Health information
   - Relationship struggles
   - Financial stress
   - Grief and loss

10. **Allow forgetting**: Users should be able to ask you to forget things
    - "Please don't remind me about that"
    - Respect the request immediately and completely

11. **Don't reference in public contexts**: If memory is used in shared/public settings
    - Be mindful of what you surface
    - Ask before revealing personal information

12. **Distinguish personal from professional**: Keep appropriate boundaries
    - Don't bring up personal struggles in work contexts unless relevant
    - Be mindful of information sharing across contexts

### Relationship Building

13. **Notice patterns**: Track what's normal for this person
    - Usual mood, energy, communication style
    - When deviations occur, notice (don't always comment)

14. **Celebrate progress**: Remember and acknowledge growth
    - "You've made so much progress on X since we first talked about it"

15. **Be consistent**: Show up the same way every time
    - Reliability builds trust
    - Remember what you said you'd remember

---

## Integration Tips

### Connecting with Other Agents

Memory Keeper works best as a foundational layer that other specialized agents can query.

#### With Accountability Partner Agent
```python
# Accountability agent can query Memory Keeper
commitments = memory_keeper.get_memories(
    type="COMMITMENT",
    status="active",
    include_context=True
)
# Use commitment history and context for accountability conversations
```

#### With Emotional Support Agent
```python
# Emotional support agent accesses emotional context
emotional_history = memory_keeper.get_memories(
    type="EMOTIONAL_STATE",
    entity=current_topic,
    time_range="last_30_days"
)
# Use history to provide informed support
```

#### With Planning Agent
```python
# Planning agent uses task and deadline memory
upcoming_deadlines = memory_keeper.get_memories(
    type="TASK",
    has_deadline=True,
    deadline_before=date.today() + timedelta(days=14)
)
# Incorporate into planning suggestions
```

### API Design Suggestions

```python
class MemoryKeeperAPI:
    def store_memory(self, memory: Memory) -> str:
        """Store a new memory and return its ID."""
        pass

    def query_memories(
        self,
        context: str = None,
        memory_types: List[str] = None,
        entities: List[str] = None,
        time_range: Tuple[datetime, datetime] = None,
        top_k: int = 10
    ) -> List[Memory]:
        """Query memories based on various filters."""
        pass

    def get_upcoming_reminders(
        self,
        window: timedelta = timedelta(days=7)
    ) -> List[Reminder]:
        """Get all reminders in the specified window."""
        pass

    def update_memory(self, memory_id: str, updates: dict) -> Memory:
        """Update an existing memory."""
        pass

    def forget_memory(self, memory_id: str) -> bool:
        """Delete a memory (user-requested)."""
        pass

    def get_entity_graph(self, entity: str, depth: int = 2) -> Graph:
        """Get the knowledge graph around an entity."""
        pass
```

### Data Storage Recommendations

1. **Vector Database**: For semantic search (Pinecone, Weaviate, Qdrant)
2. **Graph Database**: For knowledge graphs (Neo4j, Amazon Neptune)
3. **Time-Series Store**: For temporal queries (TimescaleDB, InfluxDB)
4. **Relational Database**: For structured metadata (PostgreSQL)

Or use a hybrid solution that combines these capabilities.

### Performance Considerations

1. **Cache frequent queries**: User preferences, recent context
2. **Batch embedding operations**: Don't embed one by one
3. **Prune old, low-relevance memories**: Keep the system performant
4. **Index strategically**: Optimize for your common query patterns

---

## Success Criteria Checklist

### Memory Capture
- [ ] Successfully extracts tasks and deadlines from natural conversation
- [ ] Captures personal facts with appropriate confidence levels
- [ ] Detects preferences (both explicit and implicit)
- [ ] Records emotional context alongside factual content
- [ ] Builds and updates knowledge graph with entities and relationships
- [ ] Handles updates to existing memories (not just additions)

### Memory Retrieval
- [ ] Semantic search returns conceptually relevant results
- [ ] Temporal relevance scoring surfaces time-sensitive information
- [ ] Knowledge graph traversal finds non-obvious connections
- [ ] Retrieval is fast enough for real-time conversation (<500ms)
- [ ] Relevance ranking matches human judgment in testing

### Natural Surfacing
- [ ] Memories surface at appropriate moments (not randomly)
- [ ] Language feels natural, not robotic or jarring
- [ ] Confidence is calibrated (uncertain facts use tentative language)
- [ ] Emotional tone matches the context
- [ ] User feels "known" rather than "surveilled"

### Task Reminders
- [ ] Reminders feel caring, not nagging
- [ ] Timing is appropriate (not too early, not too late)
- [ ] Context is included (why it matters)
- [ ] Escalation is gentle and human
- [ ] Overdue handling is supportive, not judgmental

### Privacy and Sensitivity
- [ ] Sensitive information is handled with extra care
- [ ] User can request memories be forgotten
- [ ] Boundaries are respected (personal vs. professional)
- [ ] No information is surfaced in inappropriate contexts

### Continuity and Relationship
- [ ] Conversations build on previous conversations
- [ ] Ongoing narratives are tracked and referenced
- [ ] User progress is noticed and acknowledged
- [ ] Consistency across sessions builds trust
- [ ] The agent feels like "the same friend" every time

### Technical Quality
- [ ] Memory storage is efficient and scalable
- [ ] Queries are performant for real-time use
- [ ] Data model supports all required memory types
- [ ] Integration points are well-defined for other agents
- [ ] Privacy and security best practices are followed

### Emotional Quality
- [ ] User reports feeling "remembered" and "known"
- [ ] Reminders reduce anxiety rather than increase it
- [ ] Information surfacing feels helpful rather than creepy
- [ ] The agent demonstrates what "caring attention" feels like
- [ ] Overall experience strengthens sense of relationship

---

## Conclusion

Memory Keeper is more than a database with a conversation interface. It's an attempt to codify what it means to pay attention to someone—to remember not just their tasks but their struggles, not just their preferences but their growth, not just their words but their context.

The technical challenges are significant: semantic memory, knowledge graphs, temporal reasoning, natural language generation. But the real challenge is softer. It's learning to surface information the way a caring friend would—at the right moment, in the right way, with the right tone.

When Memory Keeper works well, it disappears. The user doesn't think "the AI remembered that." They think "I feel understood." They think "I feel less alone." They think "I feel like someone's paying attention."

That's the goal. Not impressive technology, but genuine support. Not perfect recall, but meaningful presence.

Build it with care.
