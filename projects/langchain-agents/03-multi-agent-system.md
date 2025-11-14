# Project 03: Build a Multi-Agent System with Researcher and Writer Agents

## Overview
Build a **multi-agent system** using LangGraph where specialized agents collaborate to complete complex tasks. This project implements a researcher agent that gathers information and a writer agent that creates comprehensive content based on the research, demonstrating agent orchestration and state management.

## Learning Objectives
- Understand multi-agent architecture patterns
- Implement LangGraph state machines for agent workflows
- Design agent communication and handoffs
- Manage shared state between agents
- Implement conditional routing and decision logic
- Debug complex agent interactions

## Difficulty Level
**Advanced** - Requires understanding of state machines, agent coordination, and LangGraph.

## Technical Stack
- **Framework**: LangChain, LangGraph
- **LLM**: OpenAI GPT-4 or GPT-3.5-turbo
- **State Management**: LangGraph StateGraph
- **Tools**: Web search, document retrieval
- **Visualization**: LangGraph graph visualization
- **Additional**: Pydantic for state validation

## Project Requirements

### Multi-Agent Architecture
- **Researcher Agent**: Searches web, gathers information, validates sources
- **Writer Agent**: Creates structured content from research findings
- **Supervisor/Router**: Coordinates agent handoffs and workflow
- **Shared State**: Maintains context across agent transitions

### Agent Capabilities
1. **Researcher**:
   - Perform web searches
   - Extract key information
   - Validate and score sources
   - Summarize findings

2. **Writer**:
   - Structure content logically
   - Synthesize research into narrative
   - Format output appropriately
   - Cite sources accurately

### Workflow Design
- Linear pipeline or conditional routing
- Error handling and retry logic
- Quality checks between stages
- Final output validation

### Safety and Quality
- Maximum iteration limits
- Content validation
- Source verification
- Output sanitization

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langgraph>=0.0.20
duckduckgo-search>=4.0.0
pydantic>=2.0.0
python-dotenv>=1.0.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Define Agent State

```python
# state.py
from typing import TypedDict, Annotated, Sequence
from langchain.schema import BaseMessage
from pydantic import BaseModel, Field
import operator

class ResearchResult(BaseModel):
    """Research findings from researcher agent."""
    query: str = Field(description="Original research query")
    findings: list[str] = Field(description="List of research findings")
    sources: list[str] = Field(description="List of source URLs")
    summary: str = Field(description="Summary of research")
    confidence: float = Field(description="Confidence score 0-1")

class AgentState(TypedDict):
    """Shared state for multi-agent system."""

    # Input
    task: str  # Original user task
    messages: Annotated[Sequence[BaseMessage], operator.add]  # Message history

    # Research stage
    research_query: str  # Query for researcher
    research_results: ResearchResult  # Findings from researcher
    research_complete: bool  # Flag for research completion

    # Writing stage
    outline: str  # Content outline
    draft: str  # First draft
    final_content: str  # Final polished content
    writing_complete: bool  # Flag for writing completion

    # Metadata
    iterations: int  # Iteration counter
    next_agent: str  # Which agent should run next
    error: str  # Error message if any

def create_initial_state(task: str) -> AgentState:
    """Create initial state for a new task."""
    return AgentState(
        task=task,
        messages=[],
        research_query="",
        research_results=None,
        research_complete=False,
        outline="",
        draft="",
        final_content="",
        writing_complete=False,
        iterations=0,
        next_agent="researcher",
        error=""
    )
```

### Step 3: Build Researcher Agent

```python
# agents/researcher.py
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from duckduckgo_search import DDGS
from state import AgentState, ResearchResult
import os

class ResearcherAgent:
    """Agent responsible for researching topics."""

    def __init__(self, model_name: str = "gpt-3.5-turbo"):
        """Initialize researcher agent."""
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.3,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.research_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are an expert researcher. Your job is to:
1. Understand the research task
2. Break it down into searchable queries
3. Analyze search results
4. Extract key information
5. Provide a comprehensive summary

Be thorough, accurate, and cite your sources."""),
            ("human", """Task: {task}

Please formulate a research query to gather information about this task.""")
        ])

        self.analysis_prompt = ChatPromptTemplate.from_messages([
            ("system", "You are analyzing research findings. Extract key points and summarize."),
            ("human", """Search Results:
{search_results}

Task: {task}

Extract the most relevant information and provide:
1. Key findings (bullet points)
2. A comprehensive summary
3. Confidence score (0-1) in the research quality""")
        ])

    def formulate_query(self, task: str) -> str:
        """Create optimized search query from task."""
        chain = self.research_prompt | self.llm
        response = chain.invoke({"task": task})
        return response.content.strip()

    def search_web(self, query: str, max_results: int = 5) -> list[dict]:
        """Perform web search."""
        try:
            with DDGS() as ddgs:
                results = list(ddgs.text(query, max_results=max_results))
            return results
        except Exception as e:
            print(f"Search error: {e}")
            return []

    def analyze_results(self, search_results: list[dict], task: str) -> dict:
        """Analyze search results and extract insights."""

        # Format search results
        formatted_results = "\n\n".join([
            f"Source {i+1}: {r['title']}\n{r['body']}\nURL: {r['href']}"
            for i, r in enumerate(search_results)
        ])

        # Get analysis from LLM
        chain = self.analysis_prompt | self.llm
        response = chain.invoke({
            "search_results": formatted_results,
            "task": task
        })

        # Parse response (simplified - in production, use structured output)
        content = response.content

        return {
            "findings": content,
            "sources": [r['href'] for r in search_results],
            "summary": content[:500] + "..."
        }

    def research(self, state: AgentState) -> AgentState:
        """
        Main research function - this is called by LangGraph.

        Args:
            state: Current agent state

        Returns:
            Updated state with research results
        """
        print(f"\n🔍 Researcher Agent: Starting research on '{state['task']}'")

        # Formulate search query
        query = self.formulate_query(state["task"])
        print(f"   Search query: {query}")

        # Perform search
        search_results = self.search_web(query, max_results=5)
        print(f"   Found {len(search_results)} results")

        if not search_results:
            state["error"] = "No search results found"
            state["next_agent"] = "end"
            return state

        # Analyze results
        analysis = self.analyze_results(search_results, state["task"])

        # Create research result
        research_result = ResearchResult(
            query=query,
            findings=[analysis["findings"]],
            sources=analysis["sources"],
            summary=analysis["summary"],
            confidence=0.8  # Could be computed based on source quality
        )

        # Update state
        state["research_query"] = query
        state["research_results"] = research_result
        state["research_complete"] = True
        state["next_agent"] = "writer"
        state["iterations"] += 1

        print(f"   ✓ Research complete. Confidence: {research_result.confidence}")

        return state
```

### Step 4: Build Writer Agent

```python
# agents/writer.py
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from state import AgentState
import os

class WriterAgent:
    """Agent responsible for writing content based on research."""

    def __init__(self, model_name: str = "gpt-4"):
        """Initialize writer agent."""
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.7,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.outline_prompt = ChatPromptTemplate.from_messages([
            ("system", "You are an expert content writer creating structured outlines."),
            ("human", """Task: {task}

Research Summary:
{research_summary}

Create a detailed outline for a comprehensive article on this topic.""")
        ])

        self.writing_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are an expert content writer. Create engaging, well-structured,
and informative content. Use clear headings, bullet points where appropriate,
and maintain a professional tone."""),
            ("human", """Task: {task}

Outline:
{outline}

Research Findings:
{research_findings}

Sources:
{sources}

Write a comprehensive article following the outline. Include citations to sources
where appropriate using [1], [2], etc. notation.""")
        ])

    def create_outline(self, state: AgentState) -> str:
        """Create content outline from research."""
        chain = self.outline_prompt | self.llm
        response = chain.invoke({
            "task": state["task"],
            "research_summary": state["research_results"].summary
        })
        return response.content

    def write_content(self, state: AgentState) -> str:
        """Write final content."""

        # Format sources
        sources = "\n".join([
            f"[{i+1}] {url}"
            for i, url in enumerate(state["research_results"].sources)
        ])

        chain = self.writing_prompt | self.llm
        response = chain.invoke({
            "task": state["task"],
            "outline": state["outline"],
            "research_findings": "\n".join(state["research_results"].findings),
            "sources": sources
        })

        return response.content

    def write(self, state: AgentState) -> AgentState:
        """
        Main writing function - called by LangGraph.

        Args:
            state: Current agent state

        Returns:
            Updated state with written content
        """
        print(f"\n✍️  Writer Agent: Creating content for '{state['task']}'")

        # Create outline
        print("   Creating outline...")
        outline = self.create_outline(state)
        state["outline"] = outline

        # Write content
        print("   Writing content...")
        content = self.write_content(state)

        # Update state
        state["draft"] = content
        state["final_content"] = content  # In production, might have editing step
        state["writing_complete"] = True
        state["next_agent"] = "end"
        state["iterations"] += 1

        print(f"   ✓ Writing complete. Content length: {len(content)} chars")

        return state
```

### Step 5: Build LangGraph Workflow

```python
# workflow.py
from langgraph.graph import StateGraph, END
from state import AgentState, create_initial_state
from agents.researcher import ResearcherAgent
from agents.writer import WriterAgent

class MultiAgentWorkflow:
    """Orchestrates multi-agent workflow using LangGraph."""

    def __init__(self):
        """Initialize workflow with agents."""
        self.researcher = ResearcherAgent(model_name="gpt-3.5-turbo")
        self.writer = WriterAgent(model_name="gpt-4")
        self.workflow = self._build_workflow()

    def _build_workflow(self) -> StateGraph:
        """Build the LangGraph workflow."""

        # Create state graph
        workflow = StateGraph(AgentState)

        # Add nodes (agents)
        workflow.add_node("researcher", self.researcher.research)
        workflow.add_node("writer", self.writer.write)

        # Define routing logic
        def route_next(state: AgentState) -> str:
            """Determine which node to execute next."""
            if state.get("error"):
                return "end"

            next_agent = state.get("next_agent", "researcher")

            if next_agent == "end":
                return END
            return next_agent

        # Set entry point
        workflow.set_entry_point("researcher")

        # Add conditional edges
        workflow.add_conditional_edges(
            "researcher",
            route_next,
            {
                "writer": "writer",
                "end": END
            }
        )

        workflow.add_conditional_edges(
            "writer",
            route_next,
            {
                "end": END
            }
        )

        return workflow.compile()

    def run(self, task: str, max_iterations: int = 10) -> AgentState:
        """
        Execute the multi-agent workflow.

        Args:
            task: User task/query
            max_iterations: Maximum iterations to prevent infinite loops

        Returns:
            Final state with results
        """
        print(f"\n{'='*80}")
        print(f"Starting Multi-Agent Workflow: {task}")
        print(f"{'='*80}")

        # Create initial state
        initial_state = create_initial_state(task)

        # Run workflow
        try:
            final_state = self.workflow.invoke(
                initial_state,
                {"recursion_limit": max_iterations}
            )

            print(f"\n{'='*80}")
            print(f"Workflow Complete!")
            print(f"Total iterations: {final_state.get('iterations', 0)}")
            print(f"{'='*80}\n")

            return final_state

        except Exception as e:
            print(f"\n❌ Workflow error: {e}")
            return {"error": str(e)}

    def visualize(self):
        """Visualize the workflow graph."""
        try:
            from IPython.display import Image, display
            display(Image(self.workflow.get_graph().draw_png()))
        except Exception as e:
            print(f"Visualization not available: {e}")
            print("Install pygraphviz for graph visualization")
```

### Step 6: Main Application

```python
# main.py
import os
from dotenv import load_dotenv
from workflow import MultiAgentWorkflow

load_dotenv()

def format_output(state):
    """Format workflow output for display."""

    print("\n" + "="*80)
    print("RESEARCH & WRITING RESULTS")
    print("="*80)

    if state.get("error"):
        print(f"\n❌ Error: {state['error']}")
        return

    # Research section
    if state.get("research_results"):
        research = state["research_results"]
        print(f"\n📊 RESEARCH FINDINGS")
        print(f"{'-'*80}")
        print(f"Query: {research.query}")
        print(f"Confidence: {research.confidence:.2%}")
        print(f"\nSummary:\n{research.summary}")
        print(f"\nSources:")
        for i, source in enumerate(research.sources, 1):
            print(f"  [{i}] {source}")

    # Writing section
    if state.get("final_content"):
        print(f"\n📝 FINAL CONTENT")
        print(f"{'-'*80}")
        print(state["final_content"])

    # Metadata
    print(f"\n📈 WORKFLOW METADATA")
    print(f"{'-'*80}")
    print(f"Total iterations: {state.get('iterations', 0)}")
    print(f"Research complete: {state.get('research_complete', False)}")
    print(f"Writing complete: {state.get('writing_complete', False)}")
    print("="*80)

def main():
    """Interactive CLI for multi-agent system."""

    print("Multi-Agent Research & Writing System")
    print("="*80)
    print("This system uses two specialized agents:")
    print("  1. Researcher: Gathers information from the web")
    print("  2. Writer: Creates comprehensive content")
    print("\nType 'quit' to exit.\n")

    # Initialize workflow
    workflow = MultiAgentWorkflow()

    while True:
        task = input("\nEnter your task/topic: ").strip()

        if task.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if not task:
            continue

        # Run workflow
        result = workflow.run(task)

        # Display results
        format_output(result)

        # Ask if user wants to save
        save = input("\nSave content to file? (y/n): ").strip().lower()
        if save == 'y':
            filename = input("Enter filename (default: output.md): ").strip()
            if not filename:
                filename = "output.md"

            with open(filename, 'w') as f:
                f.write(f"# {task}\n\n")
                f.write(result.get("final_content", ""))

                if result.get("research_results"):
                    f.write("\n\n## Sources\n\n")
                    for i, source in enumerate(result["research_results"].sources, 1):
                        f.write(f"{i}. {source}\n")

            print(f"✓ Content saved to {filename}")

if __name__ == "__main__":
    main()
```

### Step 7: Advanced Features - Supervisor Pattern

```python
# supervisor.py
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from state import AgentState
import os

class SupervisorAgent:
    """Supervisor agent that routes between specialist agents."""

    def __init__(self):
        """Initialize supervisor."""
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.routing_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are a supervisor managing a team of agents:
- researcher: Gathers information from the web
- writer: Creates written content
- reviewer: Reviews and improves content

Based on the current state and task, decide which agent should act next.
Respond with only the agent name or 'FINISH' if the task is complete."""),
            ("human", """Task: {task}

Current State:
- Research complete: {research_complete}
- Writing complete: {writing_complete}
- Iterations: {iterations}

What should happen next?""")
        ])

    def route(self, state: AgentState) -> AgentState:
        """Determine next agent."""

        response = self.llm.invoke(
            self.routing_prompt.format(
                task=state["task"],
                research_complete=state.get("research_complete", False),
                writing_complete=state.get("writing_complete", False),
                iterations=state.get("iterations", 0)
            )
        )

        next_agent = response.content.strip().lower()

        if next_agent == "finish":
            next_agent = "end"

        state["next_agent"] = next_agent
        return state
```

## Expected Outputs

### Example Workflow Output
```
================================================================================
Starting Multi-Agent Workflow: The impact of AI on healthcare
================================================================================

🔍 Researcher Agent: Starting research on 'The impact of AI on healthcare'
   Search query: AI artificial intelligence healthcare medical applications 2024
   Found 5 results
   ✓ Research complete. Confidence: 0.80

✍️  Writer Agent: Creating content for 'The impact of AI on healthcare'
   Creating outline...
   Writing content...
   ✓ Writing complete. Content length: 2547 chars

================================================================================
Workflow Complete!
Total iterations: 2
================================================================================

📊 RESEARCH FINDINGS
--------------------------------------------------------------------------------
Query: AI artificial intelligence healthcare medical applications 2024
Confidence: 80.00%

Summary:
AI is transforming healthcare through diagnostic imaging, personalized medicine,
drug discovery, and patient monitoring...

Sources:
  [1] https://www.example.com/ai-healthcare
  [2] https://www.example.com/medical-ai-2024

📝 FINAL CONTENT
--------------------------------------------------------------------------------
# The Impact of AI on Healthcare

## Introduction
Artificial intelligence is revolutionizing the healthcare industry...

[Full article content with citations]

📈 WORKFLOW METADATA
--------------------------------------------------------------------------------
Total iterations: 2
Research complete: True
Writing complete: True
```

## Bonus Challenges

1. **Add More Specialized Agents**:
   - Fact-checker agent
   - Editor/reviewer agent
   - SEO optimizer agent
   - Citation validator agent

2. **Implement Parallel Execution**:
   - Multiple researchers for different aspects
   - Parallel fact-checking
   - Concurrent content generation

3. **Add Human-in-the-Loop**:
   - Approval checkpoints
   - Feedback incorporation
   - Manual routing decisions

4. **Enhanced State Management**:
   - Persistent state storage
   - State versioning
   - Rollback capabilities

5. **Quality Control**:
   - Automated content scoring
   - Source reliability checking
   - Plagiarism detection

6. **Streaming and Real-time Updates**:
   - Stream agent thoughts
   - Live progress updates
   - Incremental content display

7. **Multi-Modal Agents**:
   - Image generation agent
   - Data visualization agent
   - Audio/video processing

## Resources

### Documentation
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Multi-Agent Systems](https://python.langchain.com/docs/use_cases/more/agents/multi_agent/)
- [State Management](https://langchain-ai.github.io/langgraph/concepts/#state)

### Tutorials
- [Building Multi-Agent Systems](https://blog.langchain.dev/langgraph-multi-agent-workflows/)
- [Agent Orchestration Patterns](https://langchain-ai.github.io/langgraph/tutorials/)

### Research
- [Multi-Agent Collaboration](https://arxiv.org/abs/2308.08155)
- [AutoGen Framework](https://microsoft.github.io/autogen/)

## Success Criteria

- [ ] Researcher agent successfully gathers information
- [ ] Writer agent creates coherent content
- [ ] Agents communicate via shared state
- [ ] Workflow executes in correct order
- [ ] State transitions work properly
- [ ] Error handling prevents crashes
- [ ] Final output includes research and writing
- [ ] Sources are properly cited
- [ ] Workflow visualization works
- [ ] Code is modular and extensible

## Testing Checklist

- [ ] Test with various research topics
- [ ] Test with insufficient search results
- [ ] Test error handling in each agent
- [ ] Test state persistence across steps
- [ ] Verify agent handoffs work correctly
- [ ] Test with max iteration limits
- [ ] Validate final content quality
- [ ] Check source attribution accuracy
- [ ] Test workflow visualization
- [ ] Verify LangSmith tracing (if enabled)

## Next Steps

After completing this project:
1. Move on to Project 04: SQL Database Query Agent
2. Experiment with different agent architectures
3. Add more specialized agents
4. Implement parallel agent execution
5. Build a web interface for the system
