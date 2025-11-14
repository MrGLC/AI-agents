# Project 01: Build a ReAct Agent with Web Search and Calculator Tools

## Overview
Build a **ReAct (Reasoning + Acting) agent** that can perform web searches and mathematical calculations. This agent uses iterative reasoning to determine which tools to use and when, making it ideal for answering questions that require both factual information and computational analysis.

## Learning Objectives
- Understand the ReAct agent architecture and reasoning pattern
- Create and integrate custom tools with LangChain
- Configure agent executors with proper error handling
- Debug agent reasoning with verbose output and tracing
- Implement safety guardrails for tool usage

## Difficulty Level
**Intermediate** - Requires understanding of LangChain basics, tool creation, and agent patterns.

## Technical Stack
- **Framework**: LangChain
- **LLM**: OpenAI GPT-4 or GPT-3.5-turbo
- **Tools**:
  - DuckDuckGo Search API (or SerpAPI)
  - Custom Calculator tool
- **Additional**: Python 3.9+, python-dotenv

## Project Requirements

### Agent Design
- Implement ReAct reasoning pattern (Thought -> Action -> Observation loop)
- Support dynamic tool selection based on user query
- Include max iteration limits to prevent infinite loops
- Add verbose logging for debugging reasoning chains

### Tools Required
1. **Web Search Tool**:
   - Search current information on the internet
   - Return top 3-5 relevant results
   - Handle rate limiting and API errors

2. **Calculator Tool**:
   - Perform mathematical operations
   - Support basic arithmetic and advanced functions
   - Return formatted numerical results

### Memory
- Stateless single-turn agent (no conversation memory required)
- Can be extended with ConversationBufferMemory for multi-turn

### Safety Considerations
- Set max iterations (default: 15)
- Validate calculator inputs to prevent code injection
- Filter search results for inappropriate content
- Add timeout limits for tool execution

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
duckduckgo-search>=4.0.0
python-dotenv>=1.0.0
numexpr>=2.8.0
```

```bash
pip install -r requirements.txt
```

```python
# .env file
OPENAI_API_KEY=your_openai_api_key_here
```

### Step 2: Create Custom Calculator Tool

```python
# tools/calculator.py
from langchain.tools import Tool
from langchain.pydantic_v1 import BaseModel, Field
import numexpr

class CalculatorInput(BaseModel):
    """Input for calculator tool."""
    expression: str = Field(description="Mathematical expression to evaluate")

def calculate(expression: str) -> str:
    """
    Safely evaluate mathematical expressions.

    Args:
        expression: Mathematical expression as a string

    Returns:
        Result of the calculation
    """
    try:
        # Use numexpr for safe evaluation
        result = numexpr.evaluate(expression).item()
        return f"The result is: {result}"
    except Exception as e:
        return f"Error calculating expression: {str(e)}"

calculator_tool = Tool(
    name="Calculator",
    func=calculate,
    description="""
    Useful for performing mathematical calculations.
    Input should be a valid mathematical expression.
    Examples: "2 + 2", "sqrt(16)", "(10 * 5) / 2", "sin(3.14159/2)"
    Supports: +, -, *, /, **, sqrt, sin, cos, tan, log, exp
    """,
    args_schema=CalculatorInput
)
```

### Step 3: Create Web Search Tool

```python
# tools/search.py
from langchain.tools import Tool
from langchain.pydantic_v1 import BaseModel, Field
from duckduckgo_search import DDGS

class SearchInput(BaseModel):
    """Input for search tool."""
    query: str = Field(description="Search query to look up")

def web_search(query: str) -> str:
    """
    Search the web for current information.

    Args:
        query: Search query string

    Returns:
        Formatted search results
    """
    try:
        with DDGS() as ddgs:
            results = list(ddgs.text(query, max_results=5))

        if not results:
            return "No results found."

        formatted_results = []
        for i, result in enumerate(results, 1):
            formatted_results.append(
                f"{i}. {result['title']}\n"
                f"   {result['body']}\n"
                f"   URL: {result['href']}"
            )

        return "\n\n".join(formatted_results)
    except Exception as e:
        return f"Error performing search: {str(e)}"

search_tool = Tool(
    name="WebSearch",
    func=web_search,
    description="""
    Useful for finding current information on the internet.
    Input should be a search query as a string.
    Use this when you need to answer questions about current events,
    recent information, or anything you don't know.
    """,
    args_schema=SearchInput
)
```

### Step 4: Build the ReAct Agent

```python
# agent.py
import os
from dotenv import load_dotenv
from langchain.agents import create_react_agent, AgentExecutor
from langchain_openai import ChatOpenAI
from langchain.prompts import PromptTemplate
from tools.calculator import calculator_tool
from tools.search import search_tool

# Load environment variables
load_dotenv()

# Define the ReAct prompt template
REACT_PROMPT = """Answer the following questions as best you can. You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought: {agent_scratchpad}
"""

def create_react_search_agent():
    """Create and configure the ReAct agent with search and calculator tools."""

    # Initialize LLM
    llm = ChatOpenAI(
        model="gpt-3.5-turbo",
        temperature=0,
        openai_api_key=os.getenv("OPENAI_API_KEY")
    )

    # Define tools
    tools = [search_tool, calculator_tool]

    # Create prompt
    prompt = PromptTemplate.from_template(REACT_PROMPT)

    # Create ReAct agent
    agent = create_react_agent(
        llm=llm,
        tools=tools,
        prompt=prompt
    )

    # Create agent executor with configuration
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,  # Enable verbose output for debugging
        max_iterations=15,  # Prevent infinite loops
        handle_parsing_errors=True,  # Gracefully handle parsing errors
        return_intermediate_steps=True  # Return reasoning steps
    )

    return agent_executor

def run_agent(question: str):
    """
    Run the ReAct agent with a given question.

    Args:
        question: User question to answer

    Returns:
        Agent response with answer and intermediate steps
    """
    agent_executor = create_react_search_agent()

    try:
        result = agent_executor.invoke({"input": question})
        return result
    except Exception as e:
        return {"error": str(e)}

if __name__ == "__main__":
    # Example usage
    questions = [
        "What is the current population of Tokyo and what is that number divided by 2?",
        "Who won the latest Nobel Prize in Physics and what is 2^10?",
        "Search for the GDP of USA in 2023 and calculate 5% of that number"
    ]

    for question in questions:
        print(f"\n{'='*80}")
        print(f"Question: {question}")
        print(f"{'='*80}\n")

        result = run_agent(question)

        if "error" in result:
            print(f"Error: {result['error']}")
        else:
            print(f"Final Answer: {result['output']}")
            print(f"\nIntermediate Steps: {len(result.get('intermediate_steps', []))}")
```

### Step 5: Add Error Handling and Callbacks

```python
# callbacks.py
from langchain.callbacks import StdOutCallbackHandler
from typing import Any, Dict, List
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class CustomCallbackHandler(StdOutCallbackHandler):
    """Custom callback handler for agent monitoring."""

    def on_tool_start(
        self,
        serialized: Dict[str, Any],
        input_str: str,
        **kwargs
    ) -> None:
        """Log when a tool starts."""
        tool_name = serialized.get("name", "Unknown")
        logger.info(f"🔧 Using tool: {tool_name}")
        logger.info(f"   Input: {input_str}")

    def on_tool_end(self, output: str, **kwargs) -> None:
        """Log when a tool completes."""
        logger.info(f"✅ Tool output: {output[:100]}...")

    def on_tool_error(self, error: Exception, **kwargs) -> None:
        """Handle tool errors."""
        logger.error(f"❌ Tool error: {str(error)}")

    def on_agent_action(self, action, **kwargs) -> None:
        """Log agent actions."""
        logger.info(f"🤔 Agent thinking: {action.log}")

    def on_agent_finish(self, finish, **kwargs) -> None:
        """Log when agent finishes."""
        logger.info(f"🎯 Agent finished: {finish.log}")

# Usage in agent.py:
# agent_executor = AgentExecutor(
#     agent=agent,
#     tools=tools,
#     callbacks=[CustomCallbackHandler()],
#     ...
# )
```

### Step 6: Create Main Application

```python
# main.py
import sys
from agent import run_agent
from callbacks import CustomCallbackHandler

def main():
    """Interactive CLI for the ReAct agent."""

    print("ReAct Agent with Web Search and Calculator")
    print("=" * 50)
    print("Ask questions that require web search or calculations.")
    print("Type 'quit' to exit.\n")

    while True:
        question = input("\nYour question: ").strip()

        if question.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if not question:
            continue

        print("\n" + "=" * 50)
        result = run_agent(question)
        print("=" * 50)

        if "error" in result:
            print(f"\n❌ Error: {result['error']}")
        else:
            print(f"\n📝 Answer: {result['output']}")

            # Show reasoning steps
            steps = result.get('intermediate_steps', [])
            if steps:
                print(f"\n🔍 Reasoning steps: {len(steps)}")
                for i, (action, observation) in enumerate(steps, 1):
                    print(f"\n  Step {i}:")
                    print(f"    Action: {action.tool}")
                    print(f"    Input: {action.tool_input}")
                    print(f"    Result: {observation[:100]}...")

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example 1: Simple Search + Calculation
```
Question: What is the current population of Tokyo and what is that number divided by 2?

Thought: I need to find the current population of Tokyo
Action: WebSearch
Action Input: current population of Tokyo 2024

Observation: Tokyo has a population of approximately 14 million people...

Thought: Now I need to divide this by 2
Action: Calculator
Action Input: 14000000 / 2

Observation: The result is: 7000000.0

Thought: I now know the final answer
Final Answer: Tokyo has a population of approximately 14 million people.
Half of that number is 7 million.
```

### Example 2: Multiple Tool Uses
```
Question: Who won the 2023 Nobel Prize in Chemistry and calculate the square root of 144?

Steps: 3
Tools Used: WebSearch (1x), Calculator (1x)
Final Answer: The 2023 Nobel Prize in Chemistry was awarded to Moungi Bawendi,
Louis Brus, and Alexei Ekimov. The square root of 144 is 12.
```

## Bonus Challenges

1. **Add More Tools**:
   - Wikipedia tool for detailed information
   - Weather API tool
   - Stock price lookup tool

2. **Implement Streaming**:
   - Stream agent thoughts and actions in real-time
   - Use async/await for better performance

3. **Add Conversation Memory**:
   - Store conversation history
   - Allow follow-up questions
   - Implement context-aware responses

4. **Build a Web Interface**:
   - Create a FastAPI backend
   - Build a React frontend
   - Display reasoning steps visually

5. **Advanced Error Handling**:
   - Implement retry logic for failed tool calls
   - Add fallback tools
   - Create custom error recovery strategies

6. **Performance Optimization**:
   - Cache search results
   - Use parallel tool execution where possible
   - Implement rate limiting

7. **LangSmith Integration**:
   - Add tracing and monitoring
   - Analyze agent performance
   - Debug failing queries

## Resources

### Documentation
- [LangChain Agents Documentation](https://python.langchain.com/docs/modules/agents/)
- [ReAct Paper](https://arxiv.org/abs/2210.03629)
- [LangChain Tools](https://python.langchain.com/docs/modules/agents/tools/)

### Tutorials
- [Building ReAct Agents from Scratch](https://python.langchain.com/docs/modules/agents/agent_types/react)
- [Custom Tool Creation](https://python.langchain.com/docs/modules/agents/tools/custom_tools)

### Related Projects
- [AutoGPT](https://github.com/Significant-Gravitas/AutoGPT)
- [BabyAGI](https://github.com/yoheinakajima/babyagi)

## Success Criteria

- [ ] Agent successfully performs web searches
- [ ] Agent correctly calculates mathematical expressions
- [ ] Agent uses reasoning to determine which tool to use
- [ ] Agent handles errors gracefully
- [ ] Agent respects max iteration limits
- [ ] Verbose output shows clear reasoning chain
- [ ] Tool inputs are validated and sanitized
- [ ] Agent provides accurate final answers
- [ ] Code is well-documented and modular
- [ ] All example questions run successfully

## Testing Checklist

- [ ] Test with search-only questions
- [ ] Test with calculation-only questions
- [ ] Test with combined search + calculation questions
- [ ] Test with invalid inputs
- [ ] Test with ambiguous questions
- [ ] Test max iteration limit
- [ ] Test error recovery
- [ ] Test with different LLM models
- [ ] Verify token usage is reasonable
- [ ] Check response time performance

## Next Steps

After completing this project:
1. Move on to Project 02: RAG Chatbot with document retrieval
2. Experiment with different agent prompts
3. Add custom tools for your specific use case
4. Deploy as an API service
5. Integrate with LangSmith for production monitoring
