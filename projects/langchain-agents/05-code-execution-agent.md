# Project 05: Build a Code Generation and Execution Agent

## Overview
Build a **code execution agent** that can generate, test, and execute Python code to solve problems. This agent combines code generation capabilities with safe code execution in a sandboxed environment, making it perfect for data analysis, automation tasks, and computational problem-solving.

## Learning Objectives
- Implement safe code execution in sandboxed environments
- Generate code using LLMs with proper error handling
- Create code validation and testing mechanisms
- Build iterative code improvement workflows
- Handle code execution errors and debugging
- Implement security best practices for code execution

## Difficulty Level
**Advanced** - Requires understanding of code execution security, sandboxing, and error handling.

## Technical Stack
- **Framework**: LangChain
- **LLM**: OpenAI GPT-4 (recommended for code quality)
- **Sandbox**: Docker containers or RestrictedPython
- **Code Execution**: Python REPL, subprocess
- **Validation**: AST parsing, static analysis
- **Testing**: pytest, unittest
- **Additional**: black (formatting), pylint (linting)

## Project Requirements

### Agent Capabilities
- Generate Python code from natural language
- Execute code in isolated sandbox
- Capture stdout, stderr, and return values
- Handle execution errors gracefully
- Iterate and improve code based on errors
- Support data analysis and visualization
- Provide code explanations

### Safety Features
- **Sandboxed Execution**: Isolated environment
- **Resource Limits**: CPU time, memory, file access
- **Forbidden Operations**: Network access, file system (configurable)
- **Import Restrictions**: Whitelist allowed modules
- **Timeout Protection**: Maximum execution time
- **Output Sanitization**: Clean dangerous output

### Code Quality
- Syntax validation before execution
- Style checking (PEP 8)
- Error recovery and retry
- Code optimization suggestions
- Unit test generation

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langchain-experimental>=0.0.50
python-dotenv>=1.0.0
RestrictedPython>=6.0
black>=23.0.0
pylint>=3.0.0
matplotlib>=3.7.0
pandas>=2.0.0
numpy>=1.24.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Create Safe Code Executor

```python
# code_executor.py
import sys
import io
import traceback
import ast
import signal
from typing import Dict, Any, Optional
from contextlib import redirect_stdout, redirect_stderr
import RestrictedPython

class TimeoutError(Exception):
    """Raised when code execution times out."""
    pass

def timeout_handler(signum, frame):
    """Handler for execution timeout."""
    raise TimeoutError("Code execution timed out")

class SafeCodeExecutor:
    """Execute Python code in a restricted environment."""

    def __init__(
        self,
        timeout: int = 30,
        allowed_imports: Optional[list] = None
    ):
        """
        Initialize safe code executor.

        Args:
            timeout: Maximum execution time in seconds
            allowed_imports: List of allowed module names
        """
        self.timeout = timeout
        self.allowed_imports = allowed_imports or [
            'math', 'random', 'datetime', 'json', 'collections',
            'itertools', 'functools', 're', 'statistics',
            'numpy', 'pandas', 'matplotlib'
        ]

    def validate_syntax(self, code: str) -> tuple[bool, str]:
        """
        Validate Python syntax.

        Returns:
            (is_valid, error_message)
        """
        try:
            ast.parse(code)
            return True, ""
        except SyntaxError as e:
            return False, f"Syntax error at line {e.lineno}: {e.msg}"

    def validate_imports(self, code: str) -> tuple[bool, str]:
        """
        Validate that only allowed imports are used.

        Returns:
            (is_valid, error_message)
        """
        try:
            tree = ast.parse(code)

            for node in ast.walk(tree):
                if isinstance(node, ast.Import):
                    for alias in node.names:
                        module = alias.name.split('.')[0]
                        if module not in self.allowed_imports:
                            return False, f"Import not allowed: {module}"

                elif isinstance(node, ast.ImportFrom):
                    module = node.module.split('.')[0] if node.module else ''
                    if module and module not in self.allowed_imports:
                        return False, f"Import not allowed: {module}"

            return True, ""

        except Exception as e:
            return False, f"Error validating imports: {str(e)}"

    def execute(self, code: str) -> Dict[str, Any]:
        """
        Execute Python code safely.

        Args:
            code: Python code string

        Returns:
            Dictionary with execution results
        """
        # Validate syntax
        is_valid, error = self.validate_syntax(code)
        if not is_valid:
            return {
                "success": False,
                "error": error,
                "error_type": "SyntaxError"
            }

        # Validate imports
        is_valid, error = self.validate_imports(code)
        if not is_valid:
            return {
                "success": False,
                "error": error,
                "error_type": "ImportError"
            }

        # Setup execution environment
        stdout_capture = io.StringIO()
        stderr_capture = io.StringIO()

        # Create restricted globals
        safe_globals = {
            '__builtins__': RestrictedPython.safe_builtins,
            '_print_': RestrictedPython.PrintCollector,
            '_getattr_': RestrictedPython.safe_globals['_getattr_'],
        }

        # Add allowed imports
        for module_name in self.allowed_imports:
            try:
                safe_globals[module_name] = __import__(module_name)
            except ImportError:
                pass  # Module not installed

        local_vars = {}

        try:
            # Set timeout
            if hasattr(signal, 'SIGALRM'):
                signal.signal(signal.SIGALRM, timeout_handler)
                signal.alarm(self.timeout)

            # Execute code
            with redirect_stdout(stdout_capture), redirect_stderr(stderr_capture):
                exec(code, safe_globals, local_vars)

            # Cancel timeout
            if hasattr(signal, 'SIGALRM'):
                signal.alarm(0)

            # Get output
            stdout_output = stdout_capture.getvalue()
            stderr_output = stderr_capture.getvalue()

            # Extract return value if present
            result_value = local_vars.get('result', None)

            return {
                "success": True,
                "output": stdout_output,
                "error": stderr_output,
                "result": result_value,
                "variables": {k: str(v)[:100] for k, v in local_vars.items() if not k.startswith('_')}
            }

        except TimeoutError:
            return {
                "success": False,
                "error": f"Code execution timed out after {self.timeout} seconds",
                "error_type": "TimeoutError"
            }

        except Exception as e:
            return {
                "success": False,
                "error": str(e),
                "error_type": type(e).__name__,
                "traceback": traceback.format_exc()
            }

        finally:
            # Ensure timeout is cancelled
            if hasattr(signal, 'SIGALRM'):
                signal.alarm(0)

class PythonREPL:
    """Simple Python REPL for code execution."""

    def __init__(self):
        """Initialize REPL."""
        self.executor = SafeCodeExecutor()
        self.history = []

    def run(self, code: str) -> str:
        """
        Run code and return formatted output.

        Args:
            code: Python code to execute

        Returns:
            Formatted execution result
        """
        result = self.executor.execute(code)
        self.history.append({"code": code, "result": result})

        if result["success"]:
            output_parts = []

            if result["output"]:
                output_parts.append(f"Output:\n{result['output']}")

            if result["result"] is not None:
                output_parts.append(f"Result: {result['result']}")

            return "\n\n".join(output_parts) if output_parts else "Code executed successfully (no output)"

        else:
            return f"Error ({result['error_type']}): {result['error']}"
```

### Step 3: Build Code Generation Agent

```python
# code_agent.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.agents import Tool, create_react_agent, AgentExecutor
from langchain.prompts import PromptTemplate
from code_executor import PythonREPL

load_dotenv()

class CodeGenerationAgent:
    """Agent that generates and executes Python code."""

    def __init__(self, model_name: str = "gpt-4"):
        """Initialize code generation agent."""

        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0.2,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.repl = PythonREPL()

        # Code generation prompt
        self.code_gen_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are an expert Python programmer. Generate clean,
efficient, and well-commented Python code to solve the given task.

Guidelines:
- Write clear, readable code
- Include error handling
- Add comments for complex logic
- Use appropriate data structures
- Follow PEP 8 style guidelines
- Only use standard library and: numpy, pandas, matplotlib
- Print results or store in 'result' variable
"""),
            ("human", "{task}")
        ])

        # Create tools
        self.tools = self._create_tools()

        # Create agent
        self.agent = self._create_agent()

    def _create_tools(self):
        """Create tools for the agent."""

        execute_tool = Tool(
            name="PythonREPL",
            func=self.repl.run,
            description="""
            Execute Python code and get the output.
            Input: Valid Python code as a string
            Use this to run code and see results.
            If there's an error, you'll see the error message.
            """
        )

        generate_code_tool = Tool(
            name="GenerateCode",
            func=self._generate_code,
            description="""
            Generate Python code for a given task.
            Input: Description of what the code should do
            Output: Python code as a string
            """
        )

        return [generate_code_tool, execute_tool]

    def _generate_code(self, task: str) -> str:
        """Generate Python code for a task."""
        chain = self.code_gen_prompt | self.llm
        response = chain.invoke({"task": task})

        # Extract code from response
        code = response.content

        # Remove markdown code blocks if present
        if "```python" in code:
            code = code.split("```python")[1].split("```")[0].strip()
        elif "```" in code:
            code = code.split("```")[1].split("```")[0].strip()

        return code

    def _create_agent(self):
        """Create the ReAct agent."""

        template = """You are a Python coding assistant. You can generate and execute Python code to solve problems.

You have access to the following tools:

{tools}

Use the following format:

Question: the task you need to complete
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original task

IMPORTANT:
1. First, generate code using GenerateCode
2. Then execute it using PythonREPL
3. If there's an error, analyze it and generate improved code
4. Iterate until the code works correctly
5. Provide the final working code and results

Begin!

Question: {input}
Thought: {agent_scratchpad}
"""

        prompt = PromptTemplate.from_template(template)

        agent = create_react_agent(
            llm=self.llm,
            tools=self.tools,
            prompt=prompt
        )

        return AgentExecutor(
            agent=agent,
            tools=self.tools,
            verbose=True,
            max_iterations=10,
            handle_parsing_errors=True,
            return_intermediate_steps=True
        )

    def solve(self, task: str) -> Dict[str, Any]:
        """
        Solve a task using code generation and execution.

        Args:
            task: Description of the task

        Returns:
            Dictionary with solution and code
        """
        try:
            result = self.agent.invoke({"input": task})

            return {
                "success": True,
                "task": task,
                "answer": result["output"],
                "steps": len(result.get("intermediate_steps", [])),
                "history": self.repl.history
            }

        except Exception as e:
            return {
                "success": False,
                "task": task,
                "error": str(e)
            }
```

### Step 4: Enhanced Agent with Self-Correction

```python
# self_correcting_agent.py
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from code_executor import SafeCodeExecutor
import os

class SelfCorrectingCodeAgent:
    """Code agent that can debug and improve its own code."""

    def __init__(self, max_iterations: int = 5):
        """Initialize self-correcting agent."""

        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0.2,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.executor = SafeCodeExecutor()
        self.max_iterations = max_iterations

        self.generation_prompt = ChatPromptTemplate.from_messages([
            ("system", "You are an expert Python programmer. Generate clean, correct code."),
            ("human", "Task: {task}\n\nGenerate Python code to solve this task.")
        ])

        self.correction_prompt = ChatPromptTemplate.from_messages([
            ("system", "You are debugging Python code. Fix the error and improve the code."),
            ("human", """
Task: {task}

Previous Code:
```python
{code}
```

Error:
{error}

Generate corrected code that fixes this error.
""")
        ])

    def solve(self, task: str) -> Dict[str, Any]:
        """
        Solve task with iterative code improvement.

        Args:
            task: Task description

        Returns:
            Final solution with code and results
        """
        print(f"\n{'='*80}")
        print(f"Task: {task}")
        print(f"{'='*80}\n")

        # Generate initial code
        print("🤖 Generating initial code...")
        chain = self.generation_prompt | self.llm
        response = chain.invoke({"task": task})
        code = self._extract_code(response.content)

        iterations = []

        for i in range(self.max_iterations):
            print(f"\n📝 Iteration {i+1}/{self.max_iterations}")
            print(f"Code:\n{code}\n")

            # Execute code
            print("⚙️  Executing code...")
            result = self.executor.execute(code)

            iterations.append({
                "iteration": i + 1,
                "code": code,
                "result": result
            })

            if result["success"]:
                print("✓ Code executed successfully!")
                return {
                    "success": True,
                    "task": task,
                    "code": code,
                    "output": result["output"],
                    "result": result.get("result"),
                    "iterations": iterations
                }
            else:
                print(f"✗ Error: {result['error']}")

                if i < self.max_iterations - 1:
                    print("🔧 Attempting to fix...")

                    # Generate corrected code
                    correction_chain = self.correction_prompt | self.llm
                    correction_response = correction_chain.invoke({
                        "task": task,
                        "code": code,
                        "error": result["error"]
                    })

                    code = self._extract_code(correction_response.content)

        # Max iterations reached
        return {
            "success": False,
            "task": task,
            "error": "Max iterations reached without successful execution",
            "iterations": iterations
        }

    def _extract_code(self, text: str) -> str:
        """Extract Python code from LLM response."""
        if "```python" in text:
            return text.split("```python")[1].split("```")[0].strip()
        elif "```" in text:
            return text.split("```")[1].split("```")[0].strip()
        return text.strip()
```

### Step 5: Data Analysis Agent

```python
# data_analysis_agent.py
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from code_executor import SafeCodeExecutor
import pandas as pd
import os

class DataAnalysisAgent:
    """Specialized agent for data analysis tasks."""

    def __init__(self):
        """Initialize data analysis agent."""

        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        self.executor = SafeCodeExecutor()

        self.analysis_prompt = ChatPromptTemplate.from_messages([
            ("system", """You are a data analysis expert. Generate Python code
using pandas, numpy, and matplotlib to analyze data and answer questions.

Always:
- Import required libraries
- Handle missing data
- Create clear visualizations
- Print summary statistics
- Store final answer in 'result' variable
"""),
            ("human", """
Data: {data_info}

Question: {question}

Generate Python code to analyze the data and answer the question.
""")
        ])

    def analyze(self, df: pd.DataFrame, question: str) -> Dict[str, Any]:
        """
        Analyze dataframe to answer a question.

        Args:
            df: Pandas DataFrame
            question: Analysis question

        Returns:
            Analysis results
        """
        # Get data info
        data_info = f"""
DataFrame shape: {df.shape}
Columns: {list(df.columns)}
Data types:
{df.dtypes.to_string()}

First few rows:
{df.head().to_string()}
"""

        # Generate analysis code
        chain = self.analysis_prompt | self.llm
        response = chain.invoke({
            "data_info": data_info,
            "question": question
        })

        # Extract code
        code = response.content
        if "```python" in code:
            code = code.split("```python")[1].split("```")[0].strip()

        # Inject dataframe into execution environment
        # (In production, would need safer approach)
        exec_globals = {"df": df}

        # Modified executor to accept custom globals
        result = self._execute_with_data(code, exec_globals)

        return {
            "question": question,
            "code": code,
            "result": result
        }

    def _execute_with_data(self, code: str, globals_dict: dict) -> dict:
        """Execute code with pre-loaded data."""
        # Simplified version - in production use proper sandbox
        import io
        from contextlib import redirect_stdout

        stdout_capture = io.StringIO()

        try:
            with redirect_stdout(stdout_capture):
                exec(code, globals_dict)

            return {
                "success": True,
                "output": stdout_capture.getvalue(),
                "result": globals_dict.get("result")
            }
        except Exception as e:
            return {
                "success": False,
                "error": str(e)
            }
```

### Step 6: Main Application

```python
# main.py
from code_agent import CodeGenerationAgent
from self_correcting_agent import SelfCorrectingCodeAgent
from data_analysis_agent import DataAnalysisAgent
import pandas as pd

def display_result(result):
    """Display code execution result."""
    print("\n" + "="*80)

    if result["success"]:
        print("✓ SUCCESS")
        print(f"\n📝 Final Answer:\n{result['answer']}")

        if result.get("history"):
            print(f"\n🔍 Code History ({len(result['history'])} executions):")
            for i, item in enumerate(result["history"], 1):
                print(f"\n  Execution {i}:")
                print(f"  Code: {item['code'][:100]}...")
                print(f"  Success: {item['result']['success']}")

    else:
        print(f"✗ ERROR: {result['error']}")

    print("="*80)

def main():
    """Interactive code generation CLI."""

    print("Code Generation & Execution Agent")
    print("="*80)
    print("This agent can generate and execute Python code to solve problems.")
    print("\nCommands:")
    print("  - Type a task and the agent will generate and run code")
    print("  - 'mode': Switch between basic/self-correcting mode")
    print("  - 'quit': Exit")
    print()

    mode = "basic"  # or "self-correcting"
    agent = None
    correcting_agent = None

    while True:
        command = input("\nTask or command: ").strip()

        if command.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if command.lower() == 'mode':
            mode = "self-correcting" if mode == "basic" else "basic"
            print(f"Switched to {mode} mode")
            continue

        if not command:
            continue

        try:
            if mode == "basic":
                if agent is None:
                    print("Initializing agent...")
                    agent = CodeGenerationAgent()

                result = agent.solve(command)
                display_result(result)

            else:  # self-correcting mode
                if correcting_agent is None:
                    print("Initializing self-correcting agent...")
                    correcting_agent = SelfCorrectingCodeAgent()

                result = correcting_agent.solve(command)

                if result["success"]:
                    print(f"\n✓ Solution found in {len(result['iterations'])} iterations")
                    print(f"\nFinal Code:\n{result['code']}")
                    print(f"\nOutput:\n{result['output']}")
                else:
                    print(f"\n✗ Failed after {len(result['iterations'])} iterations")
                    print(f"Error: {result['error']}")

        except Exception as e:
            print(f"\n✗ Error: {e}")

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example 1: Mathematical Calculation
```
Task: Calculate the first 10 Fibonacci numbers

🤖 Generating initial code...

📝 Iteration 1/5
Code:
def fibonacci(n):
    fib = [0, 1]
    for i in range(2, n):
        fib.append(fib[i-1] + fib[i-2])
    return fib

result = fibonacci(10)
print(f"First 10 Fibonacci numbers: {result}")

⚙️  Executing code...
✓ Code executed successfully!

Output:
First 10 Fibonacci numbers: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### Example 2: Data Analysis
```
Task: Analyze sales data and find the top 3 performing products

Code generated and executed:
- Loaded data successfully
- Calculated total sales per product
- Top 3 products:
  1. Product A: $125,430
  2. Product C: $98,750
  3. Product B: $87,320
```

## Bonus Challenges

1. **Advanced Sandbox**:
   - Docker-based code execution
   - Support for multiple languages
   - GPU access for ML tasks

2. **Code Testing**:
   - Automatic unit test generation
   - Test-driven development mode
   - Code coverage analysis

3. **Code Optimization**:
   - Performance profiling
   - Suggest optimizations
   - Compare implementations

4. **Package Management**:
   - Dynamic package installation
   - Dependency resolution
   - Version management

5. **Debugging Tools**:
   - Step-by-step execution
   - Variable inspection
   - Breakpoint support

6. **Collaboration Features**:
   - Code sharing and versioning
   - Collaborative editing
   - Code review integration

7. **Web Interface**:
   - Jupyter-like notebooks
   - Real-time code execution
   - Visualization rendering

## Resources

### Documentation
- [LangChain Experimental](https://python.langchain.com/docs/guides/safety/)
- [RestrictedPython](https://restrictedpython.readthedocs.io/)
- [Python AST Module](https://docs.python.org/3/library/ast.html)

### Security
- [Sandboxing Python](https://docs.python.org/3/library/sandbox.html)
- [Code Execution Security](https://owasp.org/www-community/vulnerabilities/Code_Injection)

### Tools
- [PyPy Sandbox](https://pypy.org/features.html)
- [Jupyter](https://jupyter.org/)
- [Codeium](https://codeium.com/)

## Success Criteria

- [ ] Agent generates syntactically correct code
- [ ] Code executes safely in sandbox
- [ ] Execution errors are caught and handled
- [ ] Agent can debug and fix errors
- [ ] Resource limits are enforced
- [ ] Forbidden operations are blocked
- [ ] Output is captured correctly
- [ ] Code quality is good (readable, efficient)
- [ ] Import restrictions work properly
- [ ] Timeout protection functions

## Testing Checklist

- [ ] Test simple calculations
- [ ] Test data processing tasks
- [ ] Test code with syntax errors
- [ ] Test code with runtime errors
- [ ] Test forbidden imports
- [ ] Test infinite loops (timeout)
- [ ] Test memory-intensive operations
- [ ] Test file system access (should fail)
- [ ] Test network access (should fail)
- [ ] Test malicious code attempts
- [ ] Test code correction iterations
- [ ] Test various Python libraries

## Next Steps

After completing this project:
1. Move on to Project 06: Human-in-the-Loop Agent
2. Implement Docker-based sandboxing
3. Add support for more languages
4. Build web-based code editor
5. Integrate with GitHub for code storage
