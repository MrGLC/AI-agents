# Project 08: Build a Plan-and-Execute Agent for Complex Tasks

## Overview
Build a **plan-and-execute agent** that breaks down complex tasks into subtasks, creates a comprehensive plan, and executes each step systematically. This pattern is ideal for multi-step problems that require strategic thinking and organized execution, similar to how humans approach complex projects.

## Learning Objectives
- Implement plan-and-execute architecture
- Design effective planning prompts
- Build execution engines for subtasks
- Handle plan refinement and replanning
- Implement progress tracking
- Create error recovery mechanisms
- Optimize plan quality and efficiency

## Difficulty Level
**Advanced** - Requires understanding of agent architectures, planning strategies, and LangGraph workflows.

## Technical Stack
- **Framework**: LangChain, LangGraph
- **LLM**: OpenAI GPT-4 (recommended for planning)
- **Planning**: Few-shot prompting, structured outputs
- **Execution**: Tool integration, sub-agent delegation
- **State Management**: LangGraph StateGraph
- **Tools**: Various (search, calculation, file operations)

## Project Requirements

### Planning Capabilities
- Decompose complex tasks into subtasks
- Create ordered execution plans
- Estimate task dependencies
- Identify required resources/tools
- Assess feasibility before execution
- Support plan modifications

### Execution Features
- Execute subtasks in correct order
- Handle dependencies between steps
- Track progress and state
- Collect results from each step
- Aggregate final output
- Report execution status

### Error Handling
- Detect execution failures
- Replan when needed
- Skip or retry failed steps
- Provide alternative approaches
- Maintain partial progress

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langgraph>=0.0.20
langchain-experimental>=0.0.50
pydantic>=2.0.0
python-dotenv>=1.0.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Define Plan State

```python
# state.py
from typing import TypedDict, List, Optional, Literal
from pydantic import BaseModel, Field
from datetime import datetime

class SubTask(BaseModel):
    """Individual subtask in the plan."""
    id: int
    description: str
    tool: Optional[str] = None  # Tool needed for this task
    dependencies: List[int] = Field(default_factory=list)  # IDs of prerequisite tasks
    status: Literal["pending", "in_progress", "completed", "failed"] = "pending"
    result: Optional[str] = None
    error: Optional[str] = None
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None

class Plan(BaseModel):
    """Complete execution plan."""
    objective: str
    subtasks: List[SubTask]
    created_at: datetime = Field(default_factory=datetime.now)
    estimated_steps: int = 0
    completed_steps: int = 0

class PlanExecuteState(TypedDict):
    """State for plan-and-execute agent."""
    # Input
    objective: str  # Original user objective

    # Planning stage
    plan: Optional[Plan]
    planning_complete: bool

    # Execution stage
    current_task_id: Optional[int]
    execution_results: dict  # Task ID -> result mapping

    # Control
    next_step: str  # planner, executor, or end
    iterations: int
    error: Optional[str]

def create_initial_state(objective: str) -> PlanExecuteState:
    """Create initial state."""
    return PlanExecuteState(
        objective=objective,
        plan=None,
        planning_complete=False,
        current_task_id=None,
        execution_results={},
        next_step="planner",
        iterations=0,
        error=None
    )
```

### Step 3: Build Planner Agent

```python
# planner.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from state import PlanExecuteState, Plan, SubTask
import json

load_dotenv()

class PlannerAgent:
    """Agent that creates execution plans."""

    def __init__(self, model_name: str = "gpt-4"):
        """Initialize planner agent."""
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.3,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.planning_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are an expert planner. Break down complex objectives into clear, actionable subtasks.

For each subtask, specify:
1. A clear description of what needs to be done
2. Which tool to use (if any): search, calculate, code, write, read
3. Dependencies (which tasks must complete first)

Create a logical sequence that accomplishes the objective efficiently.
Number subtasks starting from 1.

Output format:
{{
  "subtasks": [
    {{
      "id": 1,
      "description": "Task description",
      "tool": "tool_name or null",
      "dependencies": []
    }},
    ...
  ]
}}"""),
            ("human", "Objective: {objective}\n\nCreate a detailed execution plan.")
        ])

    def plan(self, state: PlanExecuteState) -> PlanExecuteState:
        """
        Create execution plan.

        Args:
            state: Current state

        Returns:
            Updated state with plan
        """
        print(f"\n📋 Planning for objective: {state['objective']}")

        # Generate plan
        chain = self.planning_prompt | self.llm
        response = chain.invoke({"objective": state["objective"]})

        # Parse plan
        plan = self._parse_plan(response.content, state["objective"])

        if not plan:
            state["error"] = "Failed to create plan"
            state["next_step"] = "end"
            return state

        # Update state
        state["plan"] = plan
        state["planning_complete"] = True
        state["next_step"] = "executor"
        state["iterations"] += 1

        print(f"✓ Created plan with {len(plan.subtasks)} subtasks")
        self._display_plan(plan)

        return state

    def _parse_plan(self, response_text: str, objective: str) -> Optional[Plan]:
        """Parse LLM response into structured plan."""
        try:
            # Extract JSON from response
            if "```json" in response_text:
                json_text = response_text.split("```json")[1].split("```")[0].strip()
            elif "```" in response_text:
                json_text = response_text.split("```")[1].split("```")[0].strip()
            else:
                json_text = response_text.strip()

            # Parse JSON
            plan_data = json.loads(json_text)

            # Create SubTask objects
            subtasks = []
            for task_data in plan_data.get("subtasks", []):
                subtask = SubTask(
                    id=task_data["id"],
                    description=task_data["description"],
                    tool=task_data.get("tool"),
                    dependencies=task_data.get("dependencies", [])
                )
                subtasks.append(subtask)

            # Create Plan
            plan = Plan(
                objective=objective,
                subtasks=subtasks,
                estimated_steps=len(subtasks)
            )

            return plan

        except Exception as e:
            print(f"Error parsing plan: {e}")
            return None

    def _display_plan(self, plan: Plan):
        """Display plan in readable format."""
        print("\n" + "="*80)
        print("EXECUTION PLAN")
        print("="*80)
        for task in plan.subtasks:
            deps = f" (depends on: {task.dependencies})" if task.dependencies else ""
            tool = f" [{task.tool}]" if task.tool else ""
            print(f"{task.id}. {task.description}{tool}{deps}")
        print("="*80 + "\n")

    def replan(self, state: PlanExecuteState, reason: str) -> PlanExecuteState:
        """
        Create new plan based on execution feedback.

        Args:
            state: Current state
            reason: Why replanning is needed

        Returns:
            Updated state with new plan
        """
        print(f"\n🔄 Replanning due to: {reason}")

        # Enhanced prompt with context
        replan_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are replanning based on execution feedback.
Create an updated plan that addresses the issues encountered."""),
            ("human", """Original objective: {objective}

Original plan:
{original_plan}

Execution results so far:
{results}

Reason for replanning: {reason}

Create an updated execution plan.""")
        ])

        # Generate new plan
        chain = replan_prompt | self.llm
        response = chain.invoke({
            "objective": state["objective"],
            "original_plan": self._format_plan(state["plan"]),
            "results": json.dumps(state["execution_results"], indent=2),
            "reason": reason
        })

        # Parse and update plan
        new_plan = self._parse_plan(response.content, state["objective"])

        if new_plan:
            state["plan"] = new_plan
            print(f"✓ Created new plan with {len(new_plan.subtasks)} subtasks")
            self._display_plan(new_plan)

        return state

    def _format_plan(self, plan: Optional[Plan]) -> str:
        """Format plan for display."""
        if not plan:
            return "No plan"

        lines = []
        for task in plan.subtasks:
            lines.append(f"{task.id}. {task.description} - {task.status}")
        return "\n".join(lines)
```

### Step 4: Build Executor Agent

```python
# executor.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.tools import Tool
from state import PlanExecuteState, SubTask
from datetime import datetime
from typing import Optional

load_dotenv()

class ExecutorAgent:
    """Agent that executes plan subtasks."""

    def __init__(self):
        """Initialize executor agent."""
        self.llm = ChatOpenAI(
            model="gpt-3.5-turbo",
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create tools (simplified versions)
        self.tools = self._create_tools()
        self.tool_map = {tool.name: tool for tool in self.tools}

    def _create_tools(self):
        """Create execution tools."""

        def search(query: str) -> str:
            """Mock search tool."""
            return f"Search results for: {query}"

        def calculate(expression: str) -> str:
            """Mock calculator."""
            try:
                result = eval(expression)
                return f"Result: {result}"
            except Exception as e:
                return f"Error: {str(e)}"

        def write_text(content: str) -> str:
            """Mock write tool."""
            return f"Written: {content[:100]}..."

        return [
            Tool(name="search", func=search, description="Search for information"),
            Tool(name="calculate", func=calculate, description="Perform calculations"),
            Tool(name="write", func=write_text, description="Write content"),
        ]

    def execute(self, state: PlanExecuteState) -> PlanExecuteState:
        """
        Execute next subtask in plan.

        Args:
            state: Current state

        Returns:
            Updated state
        """
        plan = state["plan"]

        if not plan:
            state["error"] = "No plan available"
            state["next_step"] = "end"
            return state

        # Find next task to execute
        next_task = self._find_next_task(plan, state["execution_results"])

        if not next_task:
            # All tasks complete or no executable tasks
            if len(state["execution_results"]) == len(plan.subtasks):
                print("\n✅ All tasks completed!")
                state["next_step"] = "finalizer"
            else:
                state["error"] = "No executable tasks remaining"
                state["next_step"] = "end"
            return state

        # Execute the task
        print(f"\n⚙️  Executing Task {next_task.id}: {next_task.description}")

        # Mark as in progress
        next_task.status = "in_progress"
        next_task.started_at = datetime.now()
        state["current_task_id"] = next_task.id

        # Execute based on tool or direct LLM call
        result = self._execute_task(next_task, state)

        # Update task status
        if result.get("success"):
            next_task.status = "completed"
            next_task.result = result["output"]
            print(f"✓ Task {next_task.id} completed")
        else:
            next_task.status = "failed"
            next_task.error = result.get("error")
            print(f"✗ Task {next_task.id} failed: {result.get('error')}")

        next_task.completed_at = datetime.now()

        # Store result
        state["execution_results"][next_task.id] = result

        # Update plan stats
        plan.completed_steps = len([t for t in plan.subtasks if t.status == "completed"])

        state["iterations"] += 1

        return state

    def _find_next_task(self, plan: Plan, results: dict) -> Optional[SubTask]:
        """Find next executable task."""

        for task in plan.subtasks:
            # Skip if already executed
            if task.id in results:
                continue

            # Check if dependencies are met
            dependencies_met = all(
                dep_id in results and results[dep_id].get("success", False)
                for dep_id in task.dependencies
            )

            if dependencies_met:
                return task

        return None

    def _execute_task(self, task: SubTask, state: PlanExecuteState) -> dict:
        """Execute a single task."""

        try:
            # If task specifies a tool, use it
            if task.tool and task.tool in self.tool_map:
                tool = self.tool_map[task.tool]

                # Extract input from task description
                # In production, use LLM to determine tool input
                tool_input = task.description

                output = tool.func(tool_input)

                return {
                    "success": True,
                    "output": output,
                    "tool_used": task.tool
                }

            else:
                # Use LLM to complete task
                from langchain.prompts import ChatPromptTemplate

                prompt = ChatPromptTemplate.from_messages([
                    ("system", "You are executing a subtask. Provide a clear, concise result."),
                    ("human", """Task: {task_description}

Context from previous tasks:
{context}

Execute this task and provide the result.""")
                ])

                # Build context from previous results
                context = self._build_context(task, state)

                chain = prompt | self.llm
                response = chain.invoke({
                    "task_description": task.description,
                    "context": context
                })

                return {
                    "success": True,
                    "output": response.content,
                    "tool_used": "llm"
                }

        except Exception as e:
            return {
                "success": False,
                "error": str(e)
            }

    def _build_context(self, task: SubTask, state: PlanExecuteState) -> str:
        """Build context from dependency results."""

        if not task.dependencies:
            return "No previous context"

        context_parts = []
        for dep_id in task.dependencies:
            if dep_id in state["execution_results"]:
                result = state["execution_results"][dep_id]
                context_parts.append(f"Task {dep_id}: {result.get('output', 'N/A')}")

        return "\n".join(context_parts) if context_parts else "No context available"
```

### Step 5: Build LangGraph Workflow

```python
# workflow.py
from langgraph.graph import StateGraph, END
from state import PlanExecuteState, create_initial_state
from planner import PlannerAgent
from executor import ExecutorAgent

class PlanExecuteWorkflow:
    """Plan-and-Execute workflow using LangGraph."""

    def __init__(self):
        """Initialize workflow."""
        self.planner = PlannerAgent()
        self.executor = ExecutorAgent()
        self.workflow = self._build_workflow()

    def _build_workflow(self):
        """Build workflow graph."""

        workflow = StateGraph(PlanExecuteState)

        # Add nodes
        workflow.add_node("planner", self.planner.plan)
        workflow.add_node("executor", self.executor.execute)
        workflow.add_node("finalizer", self._finalize)

        # Routing logic
        def route_from_executor(state: PlanExecuteState) -> str:
            """Route from executor."""
            next_step = state.get("next_step", "executor")

            if next_step == "end":
                return END
            elif next_step == "finalizer":
                return "finalizer"
            else:
                return "executor"

        # Set entry point
        workflow.set_entry_point("planner")

        # Add edges
        workflow.add_edge("planner", "executor")

        workflow.add_conditional_edges(
            "executor",
            route_from_executor,
            {
                "executor": "executor",
                "finalizer": "finalizer",
                END: END
            }
        )

        workflow.add_edge("finalizer", END)

        return workflow.compile()

    def _finalize(self, state: PlanExecuteState) -> PlanExecuteState:
        """Create final summary."""

        print("\n" + "="*80)
        print("EXECUTION SUMMARY")
        print("="*80)

        plan = state["plan"]
        if plan:
            print(f"\nObjective: {plan.objective}")
            print(f"Total tasks: {len(plan.subtasks)}")
            print(f"Completed: {plan.completed_steps}")
            print(f"Success rate: {plan.completed_steps/len(plan.subtasks)*100:.1f}%")

            print("\nTask Results:")
            for task in plan.subtasks:
                status_icon = "✓" if task.status == "completed" else "✗"
                print(f"{status_icon} Task {task.id}: {task.description}")
                if task.result:
                    print(f"   Result: {task.result[:100]}...")

        print("="*80 + "\n")

        state["next_step"] = "end"
        return state

    def run(self, objective: str):
        """Run plan-and-execute workflow."""

        print(f"\n{'='*80}")
        print(f"Plan-and-Execute Agent")
        print(f"{'='*80}\n")

        initial_state = create_initial_state(objective)

        try:
            final_state = self.workflow.invoke(initial_state)
            return final_state

        except Exception as e:
            print(f"\n❌ Error: {e}")
            return {"error": str(e)}
```

### Step 6: Main Application

```python
# main.py
from workflow import PlanExecuteWorkflow

def main():
    """Interactive plan-and-execute agent."""

    print("Plan-and-Execute Agent")
    print("="*80)
    print("This agent breaks down complex objectives into subtasks and executes them.")
    print("Type 'quit' to exit.\n")

    workflow = PlanExecuteWorkflow()

    while True:
        objective = input("\nEnter your objective: ").strip()

        if objective.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if not objective:
            continue

        # Run workflow
        result = workflow.run(objective)

        # Results are displayed by the finalizer
        if result.get("error"):
            print(f"\n❌ Error: {result['error']}")

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example: Research and Write Report
```
Enter your objective: Research AI trends in 2024 and write a summary report

================================================================================
Plan-and-Execute Agent
================================================================================

📋 Planning for objective: Research AI trends in 2024 and write a summary report
✓ Created plan with 4 subtasks

================================================================================
EXECUTION PLAN
================================================================================
1. Research current AI trends in 2024 [search]
2. Identify top 5 most significant trends [llm] (depends on: [1])
3. Write detailed summary of each trend [llm] (depends on: [2])
4. Compile into final report [write] (depends on: [3])
================================================================================

⚙️  Executing Task 1: Research current AI trends in 2024
✓ Task 1 completed

⚙️  Executing Task 2: Identify top 5 most significant trends
✓ Task 2 completed

⚙️  Executing Task 3: Write detailed summary of each trend
✓ Task 3 completed

⚙️  Executing Task 4: Compile into final report
✓ Task 4 completed

✅ All tasks completed!

================================================================================
EXECUTION SUMMARY
================================================================================

Objective: Research AI trends in 2024 and write a summary report
Total tasks: 4
Completed: 4
Success rate: 100.0%

Task Results:
✓ Task 1: Research current AI trends in 2024
   Result: Search results for: Research current AI trends in 2024...
✓ Task 2: Identify top 5 most significant trends
   Result: Based on the research, the top 5 AI trends are...
✓ Task 3: Write detailed summary of each trend
   Result: 1. Generative AI: ...
✓ Task 4: Compile into final report
   Result: Written: AI Trends 2024 Report...
================================================================================
```

## Bonus Challenges

1. **Dynamic Replanning**:
   - Detect when plan needs revision
   - Automatically replan on failures
   - Adapt to new information

2. **Parallel Execution**:
   - Execute independent tasks in parallel
   - Optimize execution order
   - Resource allocation

3. **Progress Visualization**:
   - Real-time progress tracking
   - Gantt chart visualization
   - Execution timeline

4. **Plan Optimization**:
   - Minimize total steps
   - Optimize for time/cost
   - Find critical path

5. **Learning from History**:
   - Store successful plans
   - Reuse similar plans
   - Improve planning over time

6. **Interactive Planning**:
   - Human plan approval
   - Manual plan editing
   - Step-by-step execution control

7. **Advanced Error Recovery**:
   - Automatic retry with backoff
   - Alternative approaches
   - Graceful degradation

## Resources

### Documentation
- [Plan-and-Execute Pattern](https://blog.langchain.dev/planning-agents/)
- [LangGraph Planning](https://langchain-ai.github.io/langgraph/tutorials/)

### Research
- [ReWOO: Decoupling Reasoning from Observations](https://arxiv.org/abs/2305.18323)
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601)

## Success Criteria

- [ ] Agent creates logical execution plans
- [ ] Plans are broken into clear subtasks
- [ ] Dependencies are correctly identified
- [ ] Tasks execute in proper order
- [ ] Results are aggregated correctly
- [ ] Failed tasks are handled gracefully
- [ ] Final output achieves objective
- [ ] Progress is tracked accurately
- [ ] Code is modular and extensible

## Testing Checklist

- [ ] Test simple sequential tasks
- [ ] Test tasks with dependencies
- [ ] Test parallel-eligible tasks
- [ ] Test with task failures
- [ ] Test replanning scenarios
- [ ] Verify dependency resolution
- [ ] Test with various objectives
- [ ] Check plan quality
- [ ] Verify execution order
- [ ] Test error handling

## Next Steps

After completing this project:
1. Move on to Project 09: Research Assistant with Citations
2. Implement parallel execution
3. Add interactive plan editing
4. Build progress visualization
5. Create plan templates for common objectives
