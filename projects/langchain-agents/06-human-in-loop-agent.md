# Project 06: Build an Agent with Human-in-the-Loop Approval

## Overview
Build an **agent with human-in-the-loop (HITL)** approval checkpoints that pauses execution at critical decision points to get human confirmation. This pattern is essential for high-stakes operations where autonomous execution could have significant consequences, ensuring human oversight while maintaining automation benefits.

## Learning Objectives
- Implement human approval checkpoints in agent workflows
- Use LangGraph interrupt and resume functionality
- Design approval interfaces and prompts
- Handle approval/rejection logic
- Implement conditional routing based on human input
- Build audit trails for decision tracking
- Create escalation and fallback mechanisms

## Difficulty Level
**Intermediate to Advanced** - Requires LangGraph state management and workflow control.

## Technical Stack
- **Framework**: LangChain, LangGraph
- **LLM**: OpenAI GPT-4 or GPT-3.5-turbo
- **State Management**: LangGraph with interrupts
- **Storage**: SQLite for approval history
- **Notifications**: Email, Slack (optional)
- **UI**: CLI, Web interface (optional)
- **Additional**: Pydantic for validation

## Project Requirements

### Human-in-the-Loop Features
- **Approval Checkpoints**: Pause before critical actions
- **Action Preview**: Show what will happen before execution
- **Approval Options**: Approve, reject, modify, or request more info
- **Context Provision**: Give humans enough context to decide
- **Audit Trail**: Track all approvals and rejections
- **Timeout Handling**: Default behavior when no response

### Critical Action Types
- Data modification (UPDATE, DELETE)
- External API calls
- Financial transactions
- Email/message sending
- Code deployment
- Configuration changes
- Resource allocation

### Workflow Control
- Pause execution at checkpoints
- Resume after approval
- Cancel on rejection
- Modify parameters before proceeding
- Escalation to different approvers
- Batch approval for multiple items

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langgraph>=0.0.20
pydantic>=2.0.0
python-dotenv>=1.0.0
sqlalchemy>=2.0.0
tabulate>=0.9.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Define Approval System

```python
# approval_system.py
from typing import Optional, Literal, Dict, Any
from pydantic import BaseModel, Field
from datetime import datetime
import sqlite3
import json

class ApprovalRequest(BaseModel):
    """Request for human approval."""

    id: str = Field(description="Unique request ID")
    action_type: str = Field(description="Type of action requiring approval")
    description: str = Field(description="Human-readable description")
    details: Dict[str, Any] = Field(description="Detailed action parameters")
    risk_level: Literal["low", "medium", "high", "critical"] = Field(description="Risk assessment")
    recommended_action: str = Field(description="Agent's recommendation")
    timestamp: datetime = Field(default_factory=datetime.now)

class ApprovalResponse(BaseModel):
    """Human response to approval request."""

    request_id: str
    decision: Literal["approve", "reject", "modify", "request_info"]
    feedback: Optional[str] = None
    modifications: Optional[Dict[str, Any]] = None
    timestamp: datetime = Field(default_factory=datetime.now)

class ApprovalLogger:
    """Log approval requests and responses."""

    def __init__(self, db_path: str = "approvals.db"):
        """Initialize approval logger."""
        self.db_path = db_path
        self._setup_database()

    def _setup_database(self):
        """Create database tables."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS approval_requests (
                id TEXT PRIMARY KEY,
                action_type TEXT NOT NULL,
                description TEXT,
                details TEXT,
                risk_level TEXT,
                recommended_action TEXT,
                timestamp TEXT,
                status TEXT DEFAULT 'pending'
            )
        """)

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS approval_responses (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                request_id TEXT,
                decision TEXT,
                feedback TEXT,
                modifications TEXT,
                timestamp TEXT,
                FOREIGN KEY (request_id) REFERENCES approval_requests(id)
            )
        """)

        conn.commit()
        conn.close()

    def log_request(self, request: ApprovalRequest):
        """Log an approval request."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO approval_requests
            (id, action_type, description, details, risk_level, recommended_action, timestamp)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            request.id,
            request.action_type,
            request.description,
            json.dumps(request.details),
            request.risk_level,
            request.recommended_action,
            request.timestamp.isoformat()
        ))

        conn.commit()
        conn.close()

    def log_response(self, response: ApprovalResponse):
        """Log an approval response."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO approval_responses
            (request_id, decision, feedback, modifications, timestamp)
            VALUES (?, ?, ?, ?, ?)
        """, (
            response.request_id,
            response.decision,
            response.feedback,
            json.dumps(response.modifications) if response.modifications else None,
            response.timestamp.isoformat()
        ))

        # Update request status
        cursor.execute("""
            UPDATE approval_requests
            SET status = ?
            WHERE id = ?
        """, (response.decision, response.request_id))

        conn.commit()
        conn.close()

    def get_pending_requests(self):
        """Get all pending approval requests."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT * FROM approval_requests
            WHERE status = 'pending'
            ORDER BY timestamp DESC
        """)

        rows = cursor.fetchall()
        conn.close()

        return rows

    def get_approval_history(self, limit: int = 50):
        """Get approval history."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT r.*, res.decision, res.feedback, res.timestamp as response_timestamp
            FROM approval_requests r
            LEFT JOIN approval_responses res ON r.id = res.request_id
            ORDER BY r.timestamp DESC
            LIMIT ?
        """, (limit,))

        rows = cursor.fetchall()
        conn.close()

        return rows
```

### Step 3: Build Agent State with Approval

```python
# state.py
from typing import TypedDict, Annotated, Sequence, Optional
from langchain.schema import BaseMessage
from approval_system import ApprovalRequest, ApprovalResponse
import operator

class AgentState(TypedDict):
    """State for agent with human approval."""

    # Task information
    task: str
    messages: Annotated[Sequence[BaseMessage], operator.add]

    # Current action
    current_action: Optional[dict]
    action_plan: list[dict]  # List of planned actions

    # Approval workflow
    pending_approval: Optional[ApprovalRequest]
    approval_response: Optional[ApprovalResponse]
    requires_approval: bool

    # Execution state
    executed_actions: list[dict]
    final_result: Optional[str]

    # Control flow
    next_step: str  # Which node to execute next
    iterations: int
    error: Optional[str]

def create_initial_state(task: str) -> AgentState:
    """Create initial state."""
    return AgentState(
        task=task,
        messages=[],
        current_action=None,
        action_plan=[],
        pending_approval=None,
        approval_response=None,
        requires_approval=False,
        executed_actions=[],
        final_result=None,
        next_step="planner",
        iterations=0,
        error=None
    )
```

### Step 4: Build Agent with Approval Checkpoints

```python
# hitl_agent.py
import os
import uuid
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from state import AgentState, create_initial_state
from approval_system import ApprovalRequest, ApprovalResponse, ApprovalLogger

load_dotenv()

class PlannerAgent:
    """Agent that plans actions."""

    def __init__(self):
        """Initialize planner."""
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.3,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.planning_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are a planning assistant. Break down tasks into
actionable steps. For each step, specify:
1. Action type (research, write, send_email, modify_data, etc.)
2. Description
3. Risk level (low, medium, high, critical)
4. Whether it requires human approval

High-risk actions that require approval:
- Sending emails
- Modifying data
- Making purchases
- Deploying code
- Changing configurations
"""),
            ("human", "Task: {task}\n\nCreate an action plan.")
        ])

    def plan(self, state: AgentState) -> AgentState:
        """Create action plan."""
        print(f"\n📋 Planning actions for: {state['task']}")

        chain = self.planning_prompt | self.llm
        response = chain.invoke({"task": state["task"]})

        # Parse response into action plan
        # Simplified - in production, use structured output
        action_plan = self._parse_plan(response.content)

        state["action_plan"] = action_plan
        state["next_step"] = "executor"
        state["iterations"] += 1

        print(f"   Created plan with {len(action_plan)} actions")

        return state

    def _parse_plan(self, plan_text: str) -> list[dict]:
        """Parse plan text into structured actions."""
        # Simplified parsing - in production, use structured output
        actions = []

        # Mock action plan for demonstration
        actions.append({
            "id": str(uuid.uuid4()),
            "type": "research",
            "description": "Research the topic",
            "risk_level": "low",
            "requires_approval": False
        })

        actions.append({
            "id": str(uuid.uuid4()),
            "type": "send_email",
            "description": "Send results via email",
            "risk_level": "medium",
            "requires_approval": True,
            "details": {
                "to": "user@example.com",
                "subject": "Research Results",
                "body": "Results from the research..."
            }
        })

        return actions

class ExecutorAgent:
    """Agent that executes actions (with approval checkpoints)."""

    def __init__(self, approval_logger: ApprovalLogger):
        """Initialize executor."""
        self.llm = ChatOpenAI(
            model="gpt-3.5-turbo",
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )
        self.approval_logger = approval_logger

    def execute(self, state: AgentState) -> AgentState:
        """Execute next action in plan."""

        # Check if we have pending approval response
        if state.get("approval_response"):
            return self._handle_approval_response(state)

        # Get next action to execute
        remaining_actions = [
            a for a in state["action_plan"]
            if a["id"] not in [e["id"] for e in state["executed_actions"]]
        ]

        if not remaining_actions:
            # All actions completed
            state["next_step"] = "finalizer"
            return state

        current_action = remaining_actions[0]
        state["current_action"] = current_action

        print(f"\n⚙️  Executing: {current_action['description']}")

        # Check if action requires approval
        if current_action.get("requires_approval", False):
            print("   🛑 This action requires human approval")
            return self._request_approval(state)

        # Execute action without approval
        result = self._execute_action(current_action)

        state["executed_actions"].append({
            **current_action,
            "result": result,
            "approved": False  # No approval needed
        })

        state["iterations"] += 1

        return state

    def _request_approval(self, state: AgentState) -> AgentState:
        """Request human approval for action."""

        action = state["current_action"]

        approval_request = ApprovalRequest(
            id=str(uuid.uuid4()),
            action_type=action["type"],
            description=action["description"],
            details=action.get("details", {}),
            risk_level=action["risk_level"],
            recommended_action="approve"
        )

        # Log the request
        self.approval_logger.log_request(approval_request)

        # Set state to wait for approval
        state["pending_approval"] = approval_request
        state["requires_approval"] = True
        state["next_step"] = "wait_approval"

        return state

    def _handle_approval_response(self, state: AgentState) -> AgentState:
        """Handle human approval response."""

        response = state["approval_response"]
        action = state["current_action"]

        print(f"\n✓ Received approval decision: {response.decision}")

        if response.decision == "approve":
            # Execute the action
            result = self._execute_action(action)

            state["executed_actions"].append({
                **action,
                "result": result,
                "approved": True,
                "approval_feedback": response.feedback
            })

            # Clear approval state
            state["pending_approval"] = None
            state["approval_response"] = None
            state["requires_approval"] = False

        elif response.decision == "reject":
            print("   Action rejected by human")

            # Skip this action
            state["executed_actions"].append({
                **action,
                "result": "Rejected by human",
                "approved": False,
                "rejection_reason": response.feedback
            })

            state["pending_approval"] = None
            state["approval_response"] = None
            state["requires_approval"] = False

        elif response.decision == "modify":
            print(f"   Modifying action: {response.modifications}")

            # Modify action with human input
            action.update(response.modifications or {})

            # Execute modified action
            result = self._execute_action(action)

            state["executed_actions"].append({
                **action,
                "result": result,
                "approved": True,
                "modified": True,
                "modifications": response.modifications
            })

            state["pending_approval"] = None
            state["approval_response"] = None
            state["requires_approval"] = False

        state["iterations"] += 1

        return state

    def _execute_action(self, action: dict) -> str:
        """Execute an action (mock implementation)."""

        action_type = action["type"]

        if action_type == "research":
            return "Research completed successfully"

        elif action_type == "send_email":
            # Mock email sending
            details = action.get("details", {})
            return f"Email sent to {details.get('to', 'unknown')}"

        elif action_type == "modify_data":
            return "Data modified successfully"

        else:
            return f"Action '{action_type}' executed"

class FinalizerAgent:
    """Agent that summarizes results."""

    def __init__(self):
        """Initialize finalizer."""
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.3,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

    def finalize(self, state: AgentState) -> AgentState:
        """Create final summary."""

        print("\n📊 Finalizing results...")

        summary = f"Completed {len(state['executed_actions'])} actions:\n"

        for i, action in enumerate(state['executed_actions'], 1):
            status = "✓" if action.get('approved') is not False else "○"
            summary += f"{status} {i}. {action['description']}: {action['result']}\n"

        state["final_result"] = summary
        state["next_step"] = "end"

        return state
```

### Step 5: Build LangGraph Workflow with Interrupts

```python
# workflow.py
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from state import AgentState, create_initial_state
from hitl_agent import PlannerAgent, ExecutorAgent, FinalizerAgent
from approval_system import ApprovalLogger, ApprovalResponse

class HITLWorkflow:
    """Human-in-the-loop workflow using LangGraph."""

    def __init__(self):
        """Initialize workflow."""
        self.approval_logger = ApprovalLogger()

        self.planner = PlannerAgent()
        self.executor = ExecutorAgent(self.approval_logger)
        self.finalizer = FinalizerAgent()

        # Use memory saver for state persistence
        self.memory = MemorySaver()

        self.workflow = self._build_workflow()

    def _build_workflow(self):
        """Build LangGraph workflow with approval checkpoints."""

        workflow = StateGraph(AgentState)

        # Add nodes
        workflow.add_node("planner", self.planner.plan)
        workflow.add_node("executor", self.executor.execute)
        workflow.add_node("finalizer", self.finalizer.finalize)

        # Routing logic
        def route_from_executor(state: AgentState) -> str:
            """Determine next step from executor."""

            if state.get("requires_approval"):
                # Interrupt for human approval
                return "wait_approval"

            if state["next_step"] == "finalizer":
                return "finalizer"

            # Continue executing
            return "executor"

        # Set entry point
        workflow.set_entry_point("planner")

        # Add edges
        workflow.add_edge("planner", "executor")

        workflow.add_conditional_edges(
            "executor",
            route_from_executor,
            {
                "wait_approval": END,  # Interrupt here
                "executor": "executor",
                "finalizer": "finalizer"
            }
        )

        workflow.add_edge("finalizer", END)

        # Compile with checkpointer for state persistence
        return workflow.compile(checkpointer=self.memory)

    def run(self, task: str, thread_id: str = "default"):
        """
        Run workflow (will pause at approval checkpoints).

        Args:
            task: Task to execute
            thread_id: Unique thread ID for state persistence

        Returns:
            State at checkpoint or final state
        """
        config = {"configurable": {"thread_id": thread_id}}

        initial_state = create_initial_state(task)

        # Run until interruption or completion
        result = self.workflow.invoke(initial_state, config)

        return result

    def resume_with_approval(
        self,
        thread_id: str,
        approval_response: ApprovalResponse
    ):
        """
        Resume workflow after human approval.

        Args:
            thread_id: Thread ID to resume
            approval_response: Human approval decision

        Returns:
            Updated state
        """
        # Log the approval response
        self.approval_logger.log_response(approval_response)

        # Get current state
        config = {"configurable": {"thread_id": thread_id}}
        current_state = self.workflow.get_state(config)

        # Update state with approval response
        updated_values = {
            "approval_response": approval_response,
            "requires_approval": False,
            "next_step": "executor"
        }

        # Resume execution
        result = self.workflow.invoke(updated_values, config)

        return result
```

### Step 6: CLI Interface with Approval Prompts

```python
# main.py
from workflow import HITLWorkflow
from approval_system import ApprovalResponse
import uuid

def display_approval_request(request):
    """Display approval request to user."""
    print("\n" + "="*80)
    print("🔔 APPROVAL REQUIRED")
    print("="*80)
    print(f"\nAction Type: {request.action_type}")
    print(f"Risk Level: {request.risk_level.upper()}")
    print(f"\nDescription:")
    print(f"  {request.description}")

    if request.details:
        print(f"\nDetails:")
        for key, value in request.details.items():
            print(f"  {key}: {value}")

    print(f"\nRecommendation: {request.recommended_action}")
    print("\n" + "="*80)

def get_approval_decision(request) -> ApprovalResponse:
    """Get approval decision from user."""

    while True:
        print("\nOptions:")
        print("  1. Approve")
        print("  2. Reject")
        print("  3. Modify")
        print("  4. Request more information")

        choice = input("\nYour decision (1-4): ").strip()

        if choice == "1":
            feedback = input("Optional feedback: ").strip() or None
            return ApprovalResponse(
                request_id=request.id,
                decision="approve",
                feedback=feedback
            )

        elif choice == "2":
            reason = input("Rejection reason: ").strip()
            return ApprovalResponse(
                request_id=request.id,
                decision="reject",
                feedback=reason
            )

        elif choice == "3":
            print("\nModify action parameters:")
            # Simplified - in production, allow editing all fields
            modifications = {}

            if "to" in request.details:
                new_to = input(f"Email to ({request.details['to']}): ").strip()
                if new_to:
                    modifications["details"] = {"to": new_to}

            return ApprovalResponse(
                request_id=request.id,
                decision="modify",
                modifications=modifications
            )

        elif choice == "4":
            info_request = input("What information do you need? ").strip()
            return ApprovalResponse(
                request_id=request.id,
                decision="request_info",
                feedback=info_request
            )

        else:
            print("Invalid choice. Please try again.")

def main():
    """Interactive CLI for HITL agent."""

    print("Human-in-the-Loop Agent")
    print("="*80)
    print("This agent will ask for your approval before critical actions.")
    print()

    workflow = HITLWorkflow()

    while True:
        task = input("\nEnter task (or 'quit'): ").strip()

        if task.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if not task:
            continue

        # Generate unique thread ID
        thread_id = str(uuid.uuid4())

        print(f"\n🚀 Starting task (Thread: {thread_id[:8]}...)")

        # Run workflow
        state = workflow.run(task, thread_id)

        # Check if we hit an approval checkpoint
        while state.get("requires_approval") and state.get("pending_approval"):

            # Display approval request
            display_approval_request(state["pending_approval"])

            # Get human decision
            approval = get_approval_decision(state["pending_approval"])

            # Resume workflow with approval
            print("\n▶️  Resuming workflow...")
            state = workflow.resume_with_approval(thread_id, approval)

        # Display final results
        if state.get("final_result"):
            print("\n" + "="*80)
            print("✓ TASK COMPLETE")
            print("="*80)
            print(f"\n{state['final_result']}")
            print("="*80)

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example Workflow
```
Enter task: Send a summary email to the team

🚀 Starting task (Thread: a3f2b1c4...)

📋 Planning actions for: Send a summary email to the team
   Created plan with 2 actions

⚙️  Executing: Research the topic
   ✓ Action completed

⚙️  Executing: Send results via email
   🛑 This action requires human approval

================================================================================
🔔 APPROVAL REQUIRED
================================================================================

Action Type: send_email
Risk Level: MEDIUM

Description:
  Send results via email

Details:
  to: user@example.com
  subject: Research Results
  body: Results from the research...

Recommendation: approve

================================================================================

Options:
  1. Approve
  2. Reject
  3. Modify
  4. Request more information

Your decision (1-4): 1
Optional feedback: Looks good

▶️  Resuming workflow...

✓ Received approval decision: approve

================================================================================
✓ TASK COMPLETE
================================================================================

Completed 2 actions:
○ 1. Research the topic: Research completed successfully
✓ 2. Send results via email: Email sent to user@example.com

================================================================================
```

## Bonus Challenges

1. **Approval Delegation**:
   - Route to different approvers based on risk
   - Implement approval hierarchies
   - Support group approvals

2. **Async Approvals**:
   - Web-based approval interface
   - Email/Slack notifications
   - Mobile approval app

3. **Smart Routing**:
   - Auto-approve low-risk actions from trusted users
   - Batch similar approvals
   - Learn approval patterns

4. **Enhanced Context**:
   - Show similar past approvals
   - Provide impact analysis
   - Suggest alternatives

5. **Compliance Features**:
   - Audit trail export
   - Compliance rule checking
   - Approval analytics

6. **Rollback Capability**:
   - Undo approved actions
   - State snapshots
   - Reversal workflows

7. **Testing Mode**:
   - Dry-run mode
   - Simulation of outcomes
   - A/B testing different approaches

## Resources

### Documentation
- [LangGraph Interrupts](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/)
- [LangGraph Checkpointing](https://langchain-ai.github.io/langgraph/how-tos/persistence/)
- [State Management](https://langchain-ai.github.io/langgraph/concepts/#state)

### Patterns
- [Human-in-the-Loop Patterns](https://martinfowler.com/articles/human-in-the-loop.html)
- [Approval Workflow Design](https://www.workflowgen.com/approval-workflow/)

## Success Criteria

- [ ] Agent pauses at approval checkpoints
- [ ] Approval requests show clear context
- [ ] Human can approve/reject/modify actions
- [ ] Workflow resumes correctly after approval
- [ ] State persists across interruptions
- [ ] Approval decisions are logged
- [ ] Risk levels are assessed correctly
- [ ] Final results include approval status
- [ ] Error handling works properly
- [ ] Code is modular and extensible

## Testing Checklist

- [ ] Test approval workflow
- [ ] Test rejection workflow
- [ ] Test modification workflow
- [ ] Test multiple sequential approvals
- [ ] Test state persistence
- [ ] Test workflow resumption
- [ ] Test with different risk levels
- [ ] Test approval logging
- [ ] Test timeout scenarios
- [ ] Verify audit trail accuracy

## Next Steps

After completing this project:
1. Move on to Project 07: Conversational Agent with Memory
2. Build web-based approval interface
3. Add Slack/email notifications
4. Implement approval analytics
5. Create approval delegation system
