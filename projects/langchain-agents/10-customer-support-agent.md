# Project 10: Build a Customer Support Agent with Ticket Creation

## Overview
Build an **intelligent customer support agent** that can handle customer inquiries, access knowledge bases, escalate issues, and create support tickets. This agent combines conversational AI, information retrieval, workflow automation, and CRM integration to provide efficient, scalable customer support.

## Learning Objectives
- Implement customer support conversation flows
- Build knowledge base retrieval systems
- Create ticket management integration
- Design escalation logic
- Handle multi-intent conversations
- Track customer satisfaction
- Implement support analytics

## Difficulty Level
**Advanced** - Requires integration of multiple systems and business logic.

## Technical Stack
- **Framework**: LangChain, LangGraph
- **LLM**: OpenAI GPT-4 or GPT-3.5-turbo
- **Knowledge Base**: RAG with vector store (Chroma)
- **Ticketing**: Mock API (can integrate with Zendesk, Jira, etc.)
- **Memory**: Conversation history
- **Classification**: Intent recognition
- **Analytics**: Usage tracking
- **Database**: SQLite for tickets and history

## Project Requirements

### Support Capabilities
- Answer common questions from knowledge base
- Handle account inquiries
- Process refund/exchange requests
- Troubleshoot technical issues
- Escalate complex problems
- Create support tickets
- Track conversation context

### Ticket Management
- Create tickets with proper categorization
- Assign priority levels
- Add customer context
- Attach conversation history
- Track ticket status
- Send confirmation to customer

### Customer Experience
- Natural, empathetic responses
- Quick response times
- Personalization
- Proactive assistance
- Satisfaction surveys
- Follow-up capabilities

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langchain-community>=0.0.20
langgraph>=0.0.20
chromadb>=0.4.22
pydantic>=2.0.0
python-dotenv>=1.0.0
sqlalchemy>=2.0.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Build Ticket System

```python
# ticket_system.py
from typing import Literal, Optional
from pydantic import BaseModel, Field
from datetime import datetime
import sqlite3
import json

class SupportTicket(BaseModel):
    """Support ticket model."""

    ticket_id: str = Field(description="Unique ticket ID")
    customer_id: str = Field(description="Customer identifier")
    customer_name: str = Field(description="Customer name")
    email: Optional[str] = None
    category: Literal["technical", "billing", "account", "product", "other"]
    priority: Literal["low", "medium", "high", "critical"]
    subject: str
    description: str
    status: Literal["open", "in_progress", "resolved", "closed"] = "open"
    created_at: datetime = Field(default_factory=datetime.now)
    updated_at: datetime = Field(default_factory=datetime.now)
    assigned_to: Optional[str] = None
    conversation_history: str = ""
    resolution: Optional[str] = None

class TicketSystem:
    """Manage support tickets."""

    def __init__(self, db_path: str = "support_tickets.db"):
        """Initialize ticket system."""
        self.db_path = db_path
        self._setup_database()

    def _setup_database(self):
        """Create database tables."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS tickets (
                ticket_id TEXT PRIMARY KEY,
                customer_id TEXT NOT NULL,
                customer_name TEXT NOT NULL,
                email TEXT,
                category TEXT NOT NULL,
                priority TEXT NOT NULL,
                subject TEXT NOT NULL,
                description TEXT NOT NULL,
                status TEXT DEFAULT 'open',
                created_at TEXT NOT NULL,
                updated_at TEXT NOT NULL,
                assigned_to TEXT,
                conversation_history TEXT,
                resolution TEXT
            )
        """)

        conn.commit()
        conn.close()

    def create_ticket(self, ticket: SupportTicket) -> str:
        """Create a new support ticket."""

        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO tickets (
                ticket_id, customer_id, customer_name, email,
                category, priority, subject, description,
                status, created_at, updated_at, assigned_to,
                conversation_history, resolution
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            ticket.ticket_id,
            ticket.customer_id,
            ticket.customer_name,
            ticket.email,
            ticket.category,
            ticket.priority,
            ticket.subject,
            ticket.description,
            ticket.status,
            ticket.created_at.isoformat(),
            ticket.updated_at.isoformat(),
            ticket.assigned_to,
            ticket.conversation_history,
            ticket.resolution
        ))

        conn.commit()
        conn.close()

        print(f"\n✓ Ticket created: {ticket.ticket_id}")
        return ticket.ticket_id

    def get_ticket(self, ticket_id: str) -> Optional[SupportTicket]:
        """Retrieve ticket by ID."""

        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("SELECT * FROM tickets WHERE ticket_id = ?", (ticket_id,))
        row = cursor.fetchone()

        conn.close()

        if not row:
            return None

        # Reconstruct ticket object
        return SupportTicket(
            ticket_id=row[0],
            customer_id=row[1],
            customer_name=row[2],
            email=row[3],
            category=row[4],
            priority=row[5],
            subject=row[6],
            description=row[7],
            status=row[8],
            created_at=datetime.fromisoformat(row[9]),
            updated_at=datetime.fromisoformat(row[10]),
            assigned_to=row[11],
            conversation_history=row[12] or "",
            resolution=row[13]
        )

    def update_ticket_status(self, ticket_id: str, status: str):
        """Update ticket status."""

        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            UPDATE tickets
            SET status = ?, updated_at = ?
            WHERE ticket_id = ?
        """, (status, datetime.now().isoformat(), ticket_id))

        conn.commit()
        conn.close()

    def get_customer_tickets(self, customer_id: str) -> list:
        """Get all tickets for a customer."""

        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute(
            "SELECT ticket_id, subject, status, created_at FROM tickets WHERE customer_id = ?",
            (customer_id,)
        )

        tickets = cursor.fetchall()
        conn.close()

        return tickets

    def generate_ticket_id(self) -> str:
        """Generate unique ticket ID."""
        import uuid
        return f"TKT-{str(uuid.uuid4())[:8].upper()}"
```

### Step 3: Build Knowledge Base

```python
# knowledge_base.py
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain.schema import Document
import os

class KnowledgeBase:
    """Support knowledge base using RAG."""

    def __init__(self, persist_directory: str = "./kb_chroma"):
        """Initialize knowledge base."""

        self.embeddings = OpenAIEmbeddings(
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.persist_directory = persist_directory

        # Load or create vector store
        if os.path.exists(persist_directory):
            self.vectorstore = Chroma(
                persist_directory=persist_directory,
                embedding_function=self.embeddings
            )
        else:
            # Create with sample knowledge
            self.vectorstore = self._create_sample_kb()

    def _create_sample_kb(self) -> Chroma:
        """Create sample knowledge base."""

        kb_documents = [
            Document(
                page_content="To reset your password, go to Settings > Security > Reset Password. Enter your current password and new password.",
                metadata={"category": "account", "topic": "password_reset"}
            ),
            Document(
                page_content="Refunds are processed within 5-7 business days. You'll receive an email confirmation once the refund is initiated.",
                metadata={"category": "billing", "topic": "refunds"}
            ),
            Document(
                page_content="To cancel your subscription, go to Account > Billing > Cancel Subscription. You'll have access until the end of your billing period.",
                metadata={"category": "billing", "topic": "cancellation"}
            ),
            Document(
                page_content="If the app crashes, try: 1) Clear app cache, 2) Update to latest version, 3) Restart your device. Contact support if issue persists.",
                metadata={"category": "technical", "topic": "app_crash"}
            ),
            Document(
                page_content="Shipping typically takes 3-5 business days for standard delivery, 1-2 days for express. Track your order in the Orders section.",
                metadata={"category": "product", "topic": "shipping"}
            ),
            Document(
                page_content="To update your email address, go to Settings > Profile > Email. Verify the new email via the confirmation link sent.",
                metadata={"category": "account", "topic": "email_update"}
            ),
        ]

        vectorstore = Chroma.from_documents(
            documents=kb_documents,
            embedding=self.embeddings,
            persist_directory=self.persist_directory
        )

        print("✓ Knowledge base created")
        return vectorstore

    def search(self, query: str, k: int = 3) -> list:
        """Search knowledge base."""

        results = self.vectorstore.similarity_search(query, k=k)
        return results

    def add_article(self, content: str, category: str, topic: str):
        """Add new article to knowledge base."""

        doc = Document(
            page_content=content,
            metadata={"category": category, "topic": topic}
        )

        self.vectorstore.add_documents([doc])
```

### Step 4: Build Support Agent

```python
# support_agent.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain
from ticket_system import TicketSystem, SupportTicket
from knowledge_base import KnowledgeBase
from typing import Optional, Dict
import uuid

load_dotenv()

class CustomerSupportAgent:
    """AI-powered customer support agent."""

    def __init__(self):
        """Initialize support agent."""

        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.ticket_system = TicketSystem()
        self.knowledge_base = KnowledgeBase()
        self.memory = ConversationBufferMemory(
            memory_key="chat_history",
            return_messages=True
        )

        # Customer context
        self.customer_id = None
        self.customer_name = None
        self.customer_email = None

        # Intent classifier
        self.intent_prompt = ChatPromptTemplate.from_messages([
            ("system", """Classify the customer's intent into one of these categories:
- greeting: Hello, hi, etc.
- question: Asking for information
- issue: Reporting a problem
- request: Requesting action (refund, cancellation, etc.)
- ticket_status: Asking about existing ticket
- escalation: Complex issue needing human help

Respond with just the category name."""),
            ("human", "{message}")
        ])

    def set_customer(self, customer_id: str, name: str, email: str = None):
        """Set customer context."""
        self.customer_id = customer_id
        self.customer_name = name
        self.customer_email = email

    def classify_intent(self, message: str) -> str:
        """Classify user intent."""

        chain = self.intent_prompt | self.llm
        response = chain.invoke({"message": message})

        return response.content.strip().lower()

    def handle_message(self, message: str) -> Dict:
        """
        Process customer message.

        Returns:
            Response dict with answer, actions taken, etc.
        """
        # Classify intent
        intent = self.classify_intent(message)

        print(f"\n[Intent: {intent}]")

        response_data = {
            "intent": intent,
            "answer": "",
            "ticket_created": None,
            "kb_used": False
        }

        # Handle based on intent
        if intent == "greeting":
            response_data["answer"] = self._handle_greeting()

        elif intent == "question":
            response_data = self._handle_question(message)

        elif intent in ["issue", "request"]:
            response_data = self._handle_issue(message)

        elif intent == "ticket_status":
            response_data = self._handle_ticket_status(message)

        elif intent == "escalation":
            response_data = self._handle_escalation(message)

        else:
            response_data["answer"] = self._general_response(message)

        return response_data

    def _handle_greeting(self) -> str:
        """Handle greeting."""
        return f"Hello {self.customer_name}! I'm your support assistant. How can I help you today?"

    def _handle_question(self, message: str) -> Dict:
        """Handle question using knowledge base."""

        # Search knowledge base
        kb_results = self.knowledge_base.search(message, k=2)

        if kb_results:
            # Use KB to answer
            context = "\n\n".join([doc.page_content for doc in kb_results])

            answer_prompt = ChatPromptTemplate.from_messages([
                ("system", """You are a helpful customer support agent. Use the knowledge base
to answer the customer's question. Be friendly and concise."""),
                ("human", """Knowledge Base:
{kb_context}

Customer Question: {question}

Provide a helpful answer based on the knowledge base.""")
            ])

            chain = answer_prompt | self.llm
            response = chain.invoke({
                "kb_context": context,
                "question": message
            })

            return {
                "intent": "question",
                "answer": response.content,
                "kb_used": True,
                "ticket_created": None
            }
        else:
            return {
                "intent": "question",
                "answer": "I don't have specific information about that. Let me create a ticket for our team to help you.",
                "kb_used": False,
                "ticket_created": self._create_ticket("question", message)
            }

    def _handle_issue(self, message: str) -> Dict:
        """Handle reported issue."""

        # Check if we can solve with KB first
        kb_results = self.knowledge_base.search(message, k=2)

        if kb_results and kb_results[0].metadata.get("category") == "technical":
            # Try KB solution first
            context = kb_results[0].page_content

            response = f"I found this solution: {context}\n\nDoes this help? If not, I can create a support ticket for you."

            return {
                "intent": "issue",
                "answer": response,
                "kb_used": True,
                "ticket_created": None
            }
        else:
            # Create ticket
            ticket_id = self._create_ticket("issue", message)

            return {
                "intent": "issue",
                "answer": f"I've created a support ticket ({ticket_id}) for your issue. Our team will respond within 24 hours.",
                "kb_used": False,
                "ticket_created": ticket_id
            }

    def _handle_ticket_status(self, message: str) -> Dict:
        """Handle ticket status inquiry."""

        # Get customer's tickets
        tickets = self.ticket_system.get_customer_tickets(self.customer_id)

        if not tickets:
            answer = "You don't have any open support tickets."
        else:
            answer = "Your support tickets:\n"
            for ticket_id, subject, status, created_at in tickets:
                answer += f"- {ticket_id}: {subject} ({status})\n"

        return {
            "intent": "ticket_status",
            "answer": answer,
            "kb_used": False,
            "ticket_created": None
        }

    def _handle_escalation(self, message: str) -> Dict:
        """Handle escalation to human."""

        ticket_id = self._create_ticket("escalation", message, priority="high")

        answer = f"I've escalated this to our senior support team (Ticket: {ticket_id}). You'll receive a response within 4 hours."

        return {
            "intent": "escalation",
            "answer": answer,
            "kb_used": False,
            "ticket_created": ticket_id
        }

    def _general_response(self, message: str) -> str:
        """General conversational response."""

        prompt = ChatPromptTemplate.from_messages([
            ("system", "You are a friendly customer support agent. Provide helpful, empathetic responses."),
            ("human", "{message}")
        ])

        chain = prompt | self.llm
        response = chain.invoke({"message": message})

        return response.content

    def _create_ticket(
        self,
        category: str,
        message: str,
        priority: str = "medium"
    ) -> str:
        """Create support ticket."""

        # Generate ticket details
        ticket = SupportTicket(
            ticket_id=self.ticket_system.generate_ticket_id(),
            customer_id=self.customer_id,
            customer_name=self.customer_name,
            email=self.customer_email,
            category=category if category in ["technical", "billing", "account", "product"] else "other",
            priority=priority,
            subject=message[:100],
            description=message,
            conversation_history=self.memory.load_memory_variables({}).get("chat_history", "")
        )

        ticket_id = self.ticket_system.create_ticket(ticket)
        return ticket_id
```

### Step 5: Main Application

```python
# main.py
from support_agent import CustomerSupportAgent

def display_banner():
    """Display welcome banner."""
    print("\n" + "="*80)
    print("CUSTOMER SUPPORT AI AGENT")
    print("="*80)
    print("Your intelligent support assistant with knowledge base and ticketing.")
    print()

def main():
    """Interactive customer support interface."""

    display_banner()

    # Initialize agent
    agent = CustomerSupportAgent()

    # Customer identification
    print("Welcome! Let's get you connected to support.\n")
    customer_id = input("Customer ID: ").strip() or "CUST-001"
    customer_name = input("Your name: ").strip() or "Valued Customer"
    customer_email = input("Email (optional): ").strip() or None

    agent.set_customer(customer_id, customer_name, customer_email)

    print(f"\n✓ Connected as {customer_name}")
    print("\nType 'quit' to exit, 'tickets' to see your tickets, 'help' for commands.\n")

    # Conversation loop
    while True:
        message = input(f"\n{customer_name}: ").strip()

        if not message:
            continue

        if message.lower() in ['quit', 'exit', 'q']:
            print("\nThank you for contacting support. Have a great day!")
            break

        if message.lower() == 'tickets':
            tickets = agent.ticket_system.get_customer_tickets(customer_id)
            if tickets:
                print("\nYour Tickets:")
                for ticket_id, subject, status, created_at in tickets:
                    print(f"  {ticket_id}: {subject} [{status}]")
            else:
                print("\nYou have no tickets.")
            continue

        if message.lower() == 'help':
            print("\nAvailable commands:")
            print("  quit - Exit support chat")
            print("  tickets - View your support tickets")
            print("  help - Show this help message")
            continue

        # Process message
        try:
            response = agent.handle_message(message)

            print(f"\nSupport Agent: {response['answer']}")

            # Show additional info
            if response.get('kb_used'):
                print("[ℹ️  Answer from knowledge base]")

            if response.get('ticket_created'):
                print(f"[🎫 Ticket created: {response['ticket_created']}]")

        except Exception as e:
            print(f"\n✗ Error: {e}")
            print("Please try again or type 'quit' to exit.")

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example Conversation
```
CUSTOMER SUPPORT AI AGENT
================================================================================
Your intelligent support assistant with knowledge base and ticketing.

Welcome! Let's get you connected to support.

Customer ID: CUST-123
Your name: Alice Johnson
Email (optional): alice@example.com

✓ Connected as Alice Johnson

Type 'quit' to exit, 'tickets' to see your tickets, 'help' for commands.

Alice Johnson: Hi!

[Intent: greeting]

Support Agent: Hello Alice Johnson! I'm your support assistant. How can I help you today?

Alice Johnson: How do I reset my password?

[Intent: question]

Support Agent: To reset your password, go to Settings > Security > Reset Password.
Enter your current password and new password. You'll need to verify via email.
[ℹ️  Answer from knowledge base]

Alice Johnson: The app keeps crashing on my phone

[Intent: issue]

Support Agent: I found this solution: If the app crashes, try: 1) Clear app cache,
2) Update to latest version, 3) Restart your device. Contact support if issue persists.

Does this help? If not, I can create a support ticket for you.
[ℹ️  Answer from knowledge base]

Alice Johnson: Still not working after trying that

[Intent: escalation]

✓ Ticket created: TKT-A3F2B1C4

Support Agent: I've escalated this to our senior support team (Ticket: TKT-A3F2B1C4).
You'll receive a response within 4 hours.
[🎫 Ticket created: TKT-A3F2B1C4]
```

## Bonus Challenges

1. **Sentiment Analysis**:
   - Detect customer frustration
   - Escalate negative sentiment
   - Track satisfaction scores

2. **Multi-Channel Support**:
   - Email integration
   - Chat widget
   - SMS support
   - Voice integration

3. **Analytics Dashboard**:
   - Response time metrics
   - Resolution rates
   - Common issues tracking
   - Agent performance

4. **Advanced Routing**:
   - Skill-based assignment
   - Load balancing
   - Priority queuing
   - SLA management

5. **Self-Service Tools**:
   - Interactive troubleshooters
   - Video tutorials
   - Step-by-step guides
   - FAQ suggestions

6. **Proactive Support**:
   - Predict customer needs
   - Send helpful tips
   - Product recommendations
   - Issue prevention

7. **Integration**:
   - CRM systems (Salesforce)
   - Ticketing (Zendesk, Jira)
   - Communication (Slack, Teams)
   - Payment systems

## Resources

### Documentation
- [LangChain Chatbots](https://python.langchain.com/docs/use_cases/chatbots/)
- [RAG Documentation](https://python.langchain.com/docs/use_cases/question_answering/)

### Platforms
- [Zendesk API](https://developer.zendesk.com/)
- [Intercom](https://developers.intercom.com/)
- [Freshdesk](https://developers.freshdesk.com/)

### Best Practices
- [Customer Support AI](https://www.zendesk.com/blog/customer-service-ai/)
- [Support Automation](https://www.intercom.com/blog/support-automation/)

## Success Criteria

- [ ] Agent handles common questions accurately
- [ ] Knowledge base retrieval works effectively
- [ ] Tickets are created with proper details
- [ ] Intent classification is accurate
- [ ] Escalation logic functions correctly
- [ ] Customer context is maintained
- [ ] Responses are friendly and helpful
- [ ] Error handling is robust
- [ ] Performance is acceptable
- [ ] Code is production-ready

## Testing Checklist

- [ ] Test greeting flow
- [ ] Test knowledge base queries
- [ ] Test ticket creation
- [ ] Test ticket status checking
- [ ] Test escalation scenarios
- [ ] Test multi-turn conversations
- [ ] Verify context maintenance
- [ ] Test with various intents
- [ ] Check error handling
- [ ] Verify customer satisfaction

## Next Steps

After completing this project:
1. Deploy as web service
2. Integrate with real ticketing system
3. Add sentiment analysis
4. Build analytics dashboard
5. Implement multi-channel support
6. Add voice capabilities
7. Create admin interface

---

## Congratulations!

You've completed all 10 LangChain agent projects! You now have comprehensive knowledge of:
- Agent architectures (ReAct, Plan-and-Execute)
- RAG and retrieval systems
- Multi-agent coordination
- Human-in-the-loop workflows
- Memory management
- Code execution
- Research automation
- Customer support systems

Continue building and experimenting with these patterns to create even more sophisticated agentic systems!