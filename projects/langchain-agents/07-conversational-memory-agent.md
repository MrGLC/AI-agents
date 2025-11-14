# Project 07: Build a Conversational Agent with Memory

## Overview
Build a **conversational agent with advanced memory capabilities** that can maintain context across multiple interactions, remember user preferences, track conversation history, and provide personalized responses. This agent demonstrates how to implement different memory types for natural, context-aware conversations.

## Learning Objectives
- Implement various memory types (buffer, summary, entity, vector)
- Manage conversation context windows
- Build personalized user experiences
- Handle long-term vs short-term memory
- Implement memory retrieval strategies
- Design memory-aware prompts
- Optimize memory for performance

## Difficulty Level
**Intermediate** - Requires understanding of LangChain memory systems and conversation management.

## Technical Stack
- **Framework**: LangChain
- **LLM**: OpenAI GPT-4 or GPT-3.5-turbo
- **Memory Types**:
  - ConversationBufferMemory
  - ConversationSummaryMemory
  - ConversationEntityMemory
  - VectorStoreMemory
- **Storage**: Redis, SQLite, or in-memory
- **Vector Store**: Chroma (for semantic memory)
- **Additional**: Embeddings for semantic search

## Project Requirements

### Memory Capabilities
- **Short-term Memory**: Recent conversation history
- **Long-term Memory**: Persistent user information
- **Entity Memory**: Track mentioned entities (people, places, things)
- **Semantic Memory**: Search past conversations by meaning
- **User Preferences**: Remember user-specific settings
- **Context Management**: Sliding windows, summarization

### Conversation Features
- Multi-turn dialogue
- Follow-up questions
- Context awareness
- Personalization
- Topic tracking
- Conversation summarization

### Memory Management
- Configurable memory limits
- Automatic summarization
- Memory pruning
- Persistence and loading
- Memory search and retrieval

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langchain-community>=0.0.20
chromadb>=0.4.22
redis>=5.0.0
python-dotenv>=1.0.0
sqlalchemy>=2.0.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Implement Basic Buffer Memory

```python
# memory/buffer_memory.py
from langchain.memory import ConversationBufferMemory
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain
from langchain.prompts import PromptTemplate
import os

class BufferMemoryAgent:
    """Conversational agent with buffer memory."""

    def __init__(self, model_name: str = "gpt-3.5-turbo"):
        """Initialize agent with buffer memory."""

        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create memory
        self.memory = ConversationBufferMemory(
            memory_key="chat_history",
            return_messages=True
        )

        # Create conversation chain
        self.conversation = ConversationChain(
            llm=self.llm,
            memory=self.memory,
            verbose=True
        )

    def chat(self, message: str) -> str:
        """Send a message and get response."""
        response = self.conversation.predict(input=message)
        return response

    def get_history(self) -> str:
        """Get conversation history."""
        return self.memory.load_memory_variables({})

    def clear_memory(self):
        """Clear conversation memory."""
        self.memory.clear()
        print("Memory cleared.")

# Example usage
if __name__ == "__main__":
    agent = BufferMemoryAgent()

    print(agent.chat("Hi! My name is Alice."))
    print(agent.chat("What's my name?"))
    print(agent.chat("What did I just tell you?"))

    print("\nHistory:", agent.get_history())
```

### Step 3: Implement Window Buffer Memory

```python
# memory/window_memory.py
from langchain.memory import ConversationBufferWindowMemory
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain
import os

class WindowMemoryAgent:
    """Agent with sliding window memory (only keeps last K messages)."""

    def __init__(self, window_size: int = 5, model_name: str = "gpt-3.5-turbo"):
        """
        Initialize agent with window memory.

        Args:
            window_size: Number of message pairs to remember
        """

        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create window memory
        self.memory = ConversationBufferWindowMemory(
            k=window_size,
            memory_key="chat_history",
            return_messages=True
        )

        self.conversation = ConversationChain(
            llm=self.llm,
            memory=self.memory,
            verbose=True
        )

    def chat(self, message: str) -> str:
        """Send message with windowed context."""
        response = self.conversation.predict(input=message)
        return response

    def get_window(self):
        """Get current memory window."""
        return self.memory.load_memory_variables({})
```

### Step 4: Implement Summary Memory

```python
# memory/summary_memory.py
from langchain.memory import ConversationSummaryMemory
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain
import os

class SummaryMemoryAgent:
    """Agent that summarizes conversation history."""

    def __init__(self, model_name: str = "gpt-3.5-turbo"):
        """Initialize agent with summary memory."""

        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create summary memory
        self.memory = ConversationSummaryMemory(
            llm=self.llm,
            memory_key="chat_history",
            return_messages=True
        )

        self.conversation = ConversationChain(
            llm=self.llm,
            memory=self.memory,
            verbose=True
        )

    def chat(self, message: str) -> str:
        """Chat with automatic summarization."""
        response = self.conversation.predict(input=message)
        return response

    def get_summary(self) -> str:
        """Get current conversation summary."""
        variables = self.memory.load_memory_variables({})
        return variables.get("chat_history", "")
```

### Step 5: Implement Entity Memory

```python
# memory/entity_memory.py
from langchain.memory import ConversationEntityMemory
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain
from langchain.prompts import PromptTemplate
import os

class EntityMemoryAgent:
    """Agent that tracks entities (people, places, things)."""

    def __init__(self, model_name: str = "gpt-4"):
        """Initialize agent with entity memory."""

        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create entity memory
        self.memory = ConversationEntityMemory(
            llm=self.llm
        )

        # Custom prompt that uses entity memory
        template = """You are a helpful assistant that remembers details about people, places, and things.

Current conversation:
{history}

Context about entities:
{entities}

Last line:
Human: {input}
AI:"""

        prompt = PromptTemplate(
            input_variables=["history", "entities", "input"],
            template=template
        )

        self.conversation = ConversationChain(
            llm=self.llm,
            memory=self.memory,
            prompt=prompt,
            verbose=True
        )

    def chat(self, message: str) -> str:
        """Chat with entity tracking."""
        response = self.conversation.predict(input=message)
        return response

    def get_entities(self) -> dict:
        """Get tracked entities."""
        return self.memory.entity_store.store
```

### Step 6: Advanced Multi-Type Memory Agent

```python
# advanced_memory_agent.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.memory import (
    ConversationBufferWindowMemory,
    ConversationSummaryBufferMemory,
    CombinedMemory
)
from langchain.chains import ConversationChain
from langchain.prompts import PromptTemplate
import json
from datetime import datetime

load_dotenv()

class AdvancedMemoryAgent:
    """Agent with combined memory types and user profiles."""

    def __init__(self, user_id: str, model_name: str = "gpt-4"):
        """
        Initialize advanced memory agent.

        Args:
            user_id: Unique user identifier for personalization
        """
        self.user_id = user_id
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # User profile (long-term memory)
        self.user_profile = self._load_user_profile()

        # Short-term memory (recent messages)
        self.recent_memory = ConversationBufferWindowMemory(
            k=5,
            memory_key="recent_history",
            input_key="input"
        )

        # Summary memory (condensed history)
        self.summary_memory = ConversationSummaryBufferMemory(
            llm=self.llm,
            max_token_limit=300,
            memory_key="summary",
            input_key="input"
        )

        # Combine memories
        self.memory = CombinedMemory(
            memories=[self.recent_memory, self.summary_memory]
        )

        # Custom prompt with personalization
        self.prompt = self._create_prompt()

        # Create conversation chain
        self.conversation = ConversationChain(
            llm=self.llm,
            memory=self.memory,
            prompt=self.prompt,
            verbose=True
        )

    def _load_user_profile(self) -> dict:
        """Load or create user profile."""
        profile_file = f"user_profiles/{self.user_id}.json"

        try:
            with open(profile_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            # Create new profile
            os.makedirs("user_profiles", exist_ok=True)
            profile = {
                "user_id": self.user_id,
                "name": None,
                "preferences": {},
                "interests": [],
                "created_at": datetime.now().isoformat(),
                "conversation_count": 0
            }
            self._save_user_profile(profile)
            return profile

    def _save_user_profile(self, profile: dict = None):
        """Save user profile to disk."""
        if profile is None:
            profile = self.user_profile

        profile_file = f"user_profiles/{self.user_id}.json"
        with open(profile_file, 'w') as f:
            json.dump(profile, f, indent=2)

    def _create_prompt(self) -> PromptTemplate:
        """Create personalized prompt template."""

        template = """You are a helpful, friendly AI assistant with a good memory.

User Profile:
- Name: {user_name}
- Preferences: {user_preferences}
- Interests: {user_interests}

Conversation Summary (what happened before):
{summary}

Recent Conversation:
{recent_history}

Use the user's profile and conversation history to provide personalized, context-aware responses.
Remember details the user shares and reference them naturally in future conversations.

Current conversation:
Human: {input}
AI:"""

        return PromptTemplate(
            input_variables=[
                "user_name",
                "user_preferences",
                "user_interests",
                "summary",
                "recent_history",
                "input"
            ],
            template=template
        )

    def chat(self, message: str) -> str:
        """Send message with full context."""

        # Update conversation count
        self.user_profile["conversation_count"] += 1

        # Prepare context
        context = {
            "input": message,
            "user_name": self.user_profile.get("name", "there"),
            "user_preferences": json.dumps(self.user_profile.get("preferences", {})),
            "user_interests": ", ".join(self.user_profile.get("interests", []))
        }

        # Get response
        response = self.conversation.predict(**context)

        # Update profile based on conversation (simplified)
        self._update_profile_from_message(message)

        # Save profile
        self._save_user_profile()

        return response

    def _update_profile_from_message(self, message: str):
        """Extract and update profile information from message."""

        # Simplified profile extraction
        # In production, use LLM to extract structured information

        message_lower = message.lower()

        # Extract name
        if "my name is" in message_lower or "i'm" in message_lower or "i am" in message_lower:
            # Use LLM to extract name (simplified here)
            pass

        # Extract preferences
        if "i like" in message_lower or "i prefer" in message_lower:
            # Extract preferences
            pass

    def update_preference(self, key: str, value: str):
        """Manually update user preference."""
        self.user_profile["preferences"][key] = value
        self._save_user_profile()

    def add_interest(self, interest: str):
        """Add user interest."""
        if interest not in self.user_profile["interests"]:
            self.user_profile["interests"].append(interest)
            self._save_user_profile()

    def get_profile(self) -> dict:
        """Get user profile."""
        return self.user_profile

    def clear_conversation_memory(self):
        """Clear conversation memory but keep user profile."""
        self.memory.clear()
        print("Conversation memory cleared. User profile retained.")
```

### Step 7: Main Application with Multiple Memory Types

```python
# main.py
from memory.buffer_memory import BufferMemoryAgent
from memory.window_memory import WindowMemoryAgent
from memory.summary_memory import SummaryMemoryAgent
from advanced_memory_agent import AdvancedMemoryAgent

def display_menu():
    """Display memory type selection menu."""
    print("\n" + "="*80)
    print("Conversational Agent with Memory")
    print("="*80)
    print("\nSelect memory type:")
    print("  1. Buffer Memory (unlimited history)")
    print("  2. Window Memory (last 5 exchanges)")
    print("  3. Summary Memory (auto-summarize)")
    print("  4. Advanced Memory (personalized)")
    print("  5. Quit")
    print()

def chat_with_agent(agent, agent_type: str):
    """Chat loop with selected agent."""
    print(f"\n{'='*80}")
    print(f"Chatting with {agent_type}")
    print(f"{'='*80}")
    print("Commands: 'clear' - clear memory, 'history' - show history, 'back' - change agent\n")

    while True:
        user_input = input("You: ").strip()

        if not user_input:
            continue

        if user_input.lower() == 'back':
            break

        if user_input.lower() == 'clear':
            agent.clear_conversation_memory() if hasattr(agent, 'clear_conversation_memory') else agent.clear_memory()
            continue

        if user_input.lower() == 'history':
            if hasattr(agent, 'get_history'):
                print(agent.get_history())
            elif hasattr(agent, 'get_profile'):
                print(json.dumps(agent.get_profile(), indent=2))
            continue

        # Get response
        try:
            response = agent.chat(user_input)
            print(f"\nAssistant: {response}\n")
        except Exception as e:
            print(f"\nError: {e}\n")

def main():
    """Main application."""
    import json

    while True:
        display_menu()
        choice = input("Your choice (1-5): ").strip()

        if choice == '1':
            agent = BufferMemoryAgent()
            chat_with_agent(agent, "Buffer Memory Agent")

        elif choice == '2':
            agent = WindowMemoryAgent(window_size=5)
            chat_with_agent(agent, "Window Memory Agent")

        elif choice == '3':
            agent = SummaryMemoryAgent()
            chat_with_agent(agent, "Summary Memory Agent")

        elif choice == '4':
            user_id = input("Enter your user ID (e.g., 'alice'): ").strip() or "default_user"
            agent = AdvancedMemoryAgent(user_id)
            chat_with_agent(agent, "Advanced Memory Agent")

        elif choice == '5':
            print("Goodbye!")
            break

        else:
            print("Invalid choice. Please try again.")

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example Conversation with Buffer Memory
```
You: Hi! My name is Alice and I love Python programming.
Assistant: Hello Alice! It's great to meet you. Python is an excellent programming language...

You: What's my name?
Assistant: Your name is Alice, as you just told me!

You: What do I love?
Assistant: You mentioned that you love Python programming!
```

### Example with Advanced Memory
```
You: I prefer dark mode for my IDE.
Assistant: Noted! I'll remember that you prefer dark mode for your IDE...

[Later conversation]
You: What are my preferences?
Assistant: Based on our conversations, I know you prefer dark mode for your IDE.
```

## Bonus Challenges

1. **Vector-Based Semantic Memory**:
   - Store conversations in vector database
   - Search by semantic similarity
   - Find related past conversations

2. **Multi-Session Memory**:
   - Persist memory across sessions
   - Resume conversations from history
   - Timeline view of interactions

3. **Smart Summarization**:
   - Hierarchical summaries
   - Topic-based memory organization
   - Important moment detection

4. **Collaborative Memory**:
   - Share memory across agents
   - Team knowledge base
   - Collective learning

5. **Memory Analytics**:
   - Conversation insights
   - Topic frequency analysis
   - Engagement metrics

6. **Privacy Controls**:
   - Selective memory deletion
   - GDPR compliance
   - Data export functionality

7. **Adaptive Memory**:
   - Dynamic window sizing
   - Context-aware retrieval
   - Importance-based retention

## Resources

### Documentation
- [LangChain Memory](https://python.langchain.com/docs/modules/memory/)
- [Memory Types](https://python.langchain.com/docs/modules/memory/types/)
- [Conversation Patterns](https://python.langchain.com/docs/use_cases/chatbots/)

### Tutorials
- [Building Chatbots with Memory](https://blog.langchain.dev/conversational-memory/)
- [Advanced Memory Strategies](https://python.langchain.com/docs/use_cases/chatbots/memory_management)

## Success Criteria

- [ ] Agent maintains conversation context
- [ ] Follow-up questions work correctly
- [ ] Memory persists appropriately
- [ ] Different memory types function as expected
- [ ] User profiles save and load
- [ ] Memory clearing works
- [ ] Personalization is effective
- [ ] Performance is acceptable
- [ ] Memory limits are respected
- [ ] Code is well-structured

## Testing Checklist

- [ ] Test multi-turn conversations
- [ ] Test follow-up questions
- [ ] Test memory persistence
- [ ] Test memory limits
- [ ] Test memory clearing
- [ ] Test profile updates
- [ ] Test with long conversations
- [ ] Test memory retrieval accuracy
- [ ] Test error handling
- [ ] Verify personalization works

## Next Steps

After completing this project:
1. Move on to Project 08: Plan-and-Execute Agent
2. Implement vector-based semantic memory
3. Add Redis for distributed memory
4. Build web interface for conversations
5. Add voice input/output capabilities