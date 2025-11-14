# Project 04: Build a SQL Database Query Agent

## Overview
Build an **intelligent SQL agent** that can understand natural language questions and automatically generate, execute, and explain SQL queries. This agent provides a conversational interface to databases, making data accessible to non-technical users while maintaining security and accuracy.

## Learning Objectives
- Implement natural language to SQL translation
- Create secure database query tools
- Handle complex multi-table queries
- Implement query validation and safety checks
- Format and explain query results
- Build error recovery mechanisms
- Understand SQL agent patterns in LangChain

## Difficulty Level
**Intermediate to Advanced** - Requires SQL knowledge, security awareness, and prompt engineering skills.

## Technical Stack
- **Framework**: LangChain, LangChain SQL Database
- **LLM**: OpenAI GPT-4 (recommended for accuracy)
- **Database**: SQLite (development), PostgreSQL/MySQL (production)
- **Tools**: SQLAlchemy, SQL Database toolkit
- **Validation**: SQL query parser, schema validation
- **Additional**: pandas for result formatting

## Project Requirements

### Agent Capabilities
- Translate natural language to SQL
- Execute queries safely on database
- Validate queries before execution
- Format results in human-readable format
- Explain query logic to users
- Handle multi-step queries
- Support multiple database tables

### Database Operations
- **SELECT queries**: Read data (allowed)
- **INSERT/UPDATE/DELETE**: Restricted by default (optional with confirmation)
- **Schema inspection**: View tables, columns, relationships
- **Aggregations**: COUNT, SUM, AVG, etc.
- **Joins**: Multi-table queries
- **Filtering**: WHERE clauses, complex conditions

### Safety Features
- Query validation before execution
- Read-only mode option
- Query result limits
- SQL injection prevention
- Sensitive data filtering
- Query timeouts

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langchain-community>=0.0.20
sqlalchemy>=2.0.0
pandas>=2.0.0
python-dotenv>=1.0.0
tabulate>=0.9.0
```

```bash
pip install -r requirements.txt
```

### Step 2: Create Sample Database

```python
# setup_database.py
import sqlite3
from datetime import datetime, timedelta
import random

def create_sample_database(db_path: str = "company.db"):
    """Create a sample database with employees, departments, and sales."""

    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()

    # Create tables
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS departments (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            budget REAL,
            location TEXT
        )
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            department_id INTEGER,
            salary REAL,
            hire_date TEXT,
            email TEXT,
            FOREIGN KEY (department_id) REFERENCES departments(id)
        )
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS sales (
            id INTEGER PRIMARY KEY,
            employee_id INTEGER,
            amount REAL,
            sale_date TEXT,
            product TEXT,
            FOREIGN KEY (employee_id) REFERENCES employees(id)
        )
    """)

    # Insert sample data
    departments = [
        (1, 'Engineering', 500000, 'San Francisco'),
        (2, 'Sales', 300000, 'New York'),
        (3, 'Marketing', 250000, 'Los Angeles'),
        (4, 'HR', 150000, 'Chicago')
    ]

    employees = [
        (1, 'Alice Johnson', 1, 120000, '2020-01-15', 'alice@company.com'),
        (2, 'Bob Smith', 1, 110000, '2020-03-20', 'bob@company.com'),
        (3, 'Carol White', 2, 95000, '2019-06-10', 'carol@company.com'),
        (4, 'David Brown', 2, 105000, '2019-08-15', 'david@company.com'),
        (5, 'Eve Davis', 3, 85000, '2021-02-01', 'eve@company.com'),
        (6, 'Frank Miller', 3, 90000, '2021-04-12', 'frank@company.com'),
        (7, 'Grace Lee', 4, 75000, '2022-01-05', 'grace@company.com'),
    ]

    # Generate sales data
    sales = []
    products = ['Product A', 'Product B', 'Product C', 'Product D']
    base_date = datetime(2024, 1, 1)

    for i in range(100):
        sale_date = base_date + timedelta(days=random.randint(0, 300))
        sales.append((
            i + 1,
            random.randint(3, 6),  # Sales employees only
            round(random.uniform(1000, 50000), 2),
            sale_date.strftime('%Y-%m-%d'),
            random.choice(products)
        ))

    # Insert data
    cursor.executemany("INSERT OR REPLACE INTO departments VALUES (?, ?, ?, ?)", departments)
    cursor.executemany("INSERT OR REPLACE INTO employees VALUES (?, ?, ?, ?, ?, ?)", employees)
    cursor.executemany("INSERT OR REPLACE INTO sales VALUES (?, ?, ?, ?, ?)", sales)

    conn.commit()
    conn.close()

    print(f"✓ Sample database created at {db_path}")
    print(f"  - {len(departments)} departments")
    print(f"  - {len(employees)} employees")
    print(f"  - {len(sales)} sales records")

if __name__ == "__main__":
    create_sample_database()
```

### Step 3: Build SQL Database Tool

```python
# sql_tool.py
from langchain_community.utilities import SQLDatabase
from sqlalchemy import create_engine, text
from typing import List, Dict, Any
import pandas as pd

class SQLDatabaseTool:
    """Tool for safe SQL database operations."""

    def __init__(self, db_path: str, read_only: bool = True):
        """
        Initialize SQL database tool.

        Args:
            db_path: Path to database or connection string
            read_only: If True, only allow SELECT queries
        """
        self.db_path = db_path
        self.read_only = read_only

        # Create SQLAlchemy engine
        self.engine = create_engine(f"sqlite:///{db_path}")

        # Initialize LangChain SQL Database
        self.db = SQLDatabase(self.engine)

    def get_schema(self) -> str:
        """Get database schema information."""
        return self.db.get_table_info()

    def list_tables(self) -> List[str]:
        """List all tables in database."""
        return self.db.get_usable_table_names()

    def validate_query(self, query: str) -> tuple[bool, str]:
        """
        Validate SQL query for safety.

        Returns:
            (is_valid, message)
        """
        query_upper = query.upper().strip()

        # Check for read-only violations
        if self.read_only:
            forbidden_keywords = ['INSERT', 'UPDATE', 'DELETE', 'DROP', 'ALTER', 'CREATE']
            for keyword in forbidden_keywords:
                if keyword in query_upper:
                    return False, f"Query contains forbidden keyword: {keyword}"

        # Check for dangerous patterns
        if '--' in query or ';' in query[:-1]:  # Allow single semicolon at end
            return False, "Query contains potentially dangerous patterns"

        return True, "Query is valid"

    def execute_query(self, query: str, limit: int = 100) -> Dict[str, Any]:
        """
        Execute SQL query safely.

        Args:
            query: SQL query string
            limit: Maximum number of rows to return

        Returns:
            Dictionary with results, columns, and metadata
        """
        # Validate query
        is_valid, message = self.validate_query(query)
        if not is_valid:
            return {
                "success": False,
                "error": message,
                "query": query
            }

        try:
            # Add LIMIT if not present in SELECT queries
            query_upper = query.upper().strip()
            if query_upper.startswith('SELECT') and 'LIMIT' not in query_upper:
                query = f"{query.rstrip(';')} LIMIT {limit};"

            # Execute query
            with self.engine.connect() as conn:
                result = conn.execute(text(query))

                # For SELECT queries, fetch results
                if query_upper.startswith('SELECT'):
                    rows = result.fetchall()
                    columns = result.keys()

                    # Convert to list of dictionaries
                    data = [dict(zip(columns, row)) for row in rows]

                    return {
                        "success": True,
                        "data": data,
                        "columns": list(columns),
                        "row_count": len(data),
                        "query": query
                    }
                else:
                    conn.commit()
                    return {
                        "success": True,
                        "message": f"Query executed successfully. Rows affected: {result.rowcount}",
                        "query": query
                    }

        except Exception as e:
            return {
                "success": False,
                "error": str(e),
                "query": query
            }

    def execute_query_to_df(self, query: str) -> pd.DataFrame:
        """Execute query and return pandas DataFrame."""
        result = self.execute_query(query)

        if result["success"] and "data" in result:
            return pd.DataFrame(result["data"])
        else:
            return pd.DataFrame()

    def format_results(self, result: Dict[str, Any]) -> str:
        """Format query results for display."""
        if not result["success"]:
            return f"❌ Error: {result['error']}"

        if "data" in result:
            df = pd.DataFrame(result["data"])
            if df.empty:
                return "No results found."

            from tabulate import tabulate
            return tabulate(df, headers='keys', tablefmt='grid', showindex=False)
        else:
            return result.get("message", "Query executed successfully")
```

### Step 4: Create SQL Agent

```python
# sql_agent.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.agents import create_sql_agent
from langchain.agents.agent_toolkits import SQLDatabaseToolkit
from langchain.agents.agent_types import AgentType
from langchain_community.utilities import SQLDatabase
from sql_tool import SQLDatabaseTool

load_dotenv()

class SQLQueryAgent:
    """Natural language SQL query agent."""

    def __init__(self, db_path: str, model_name: str = "gpt-4"):
        """
        Initialize SQL agent.

        Args:
            db_path: Path to SQLite database
            model_name: OpenAI model to use
        """
        self.db_tool = SQLDatabaseTool(db_path, read_only=True)
        self.db = self.db_tool.db

        # Initialize LLM
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create SQL toolkit
        self.toolkit = SQLDatabaseToolkit(
            db=self.db,
            llm=self.llm
        )

        # Create agent
        self.agent = create_sql_agent(
            llm=self.llm,
            toolkit=self.toolkit,
            agent_type=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
            verbose=True,
            max_iterations=15,
            max_execution_time=60,
            handle_parsing_errors=True
        )

    def query(self, question: str) -> Dict[str, Any]:
        """
        Answer a natural language question about the database.

        Args:
            question: Natural language question

        Returns:
            Dictionary with answer and metadata
        """
        try:
            print(f"\n🤔 Processing question: {question}")

            result = self.agent.invoke({"input": question})

            return {
                "success": True,
                "question": question,
                "answer": result["output"],
                "intermediate_steps": result.get("intermediate_steps", [])
            }

        except Exception as e:
            return {
                "success": False,
                "question": question,
                "error": str(e)
            }

    def get_schema_summary(self) -> str:
        """Get a human-readable schema summary."""
        schema = self.db_tool.get_schema()
        return schema

    def explain_query(self, query: str) -> str:
        """
        Get LLM explanation of a SQL query.

        Args:
            query: SQL query string

        Returns:
            Human-readable explanation
        """
        from langchain.prompts import ChatPromptTemplate

        prompt = ChatPromptTemplate.from_messages([
            ("system", "You are a SQL expert. Explain SQL queries in simple terms."),
            ("human", "Explain this SQL query:\n\n{query}\n\nProvide a clear, step-by-step explanation.")
        ])

        chain = prompt | self.llm
        response = chain.invoke({"query": query})

        return response.content
```

### Step 5: Advanced Agent with Custom Tools

```python
# advanced_sql_agent.py
from langchain.tools import Tool
from langchain.agents import create_react_agent, AgentExecutor
from langchain.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from sql_tool import SQLDatabaseTool
import os

class AdvancedSQLAgent:
    """SQL agent with custom tools and enhanced capabilities."""

    def __init__(self, db_path: str):
        """Initialize advanced SQL agent."""
        self.db_tool = SQLDatabaseTool(db_path)
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Create custom tools
        self.tools = self._create_tools()

        # Create agent
        self.agent = self._create_agent()

    def _create_tools(self) -> list:
        """Create custom tools for SQL operations."""

        schema_tool = Tool(
            name="GetSchema",
            func=lambda _: self.db_tool.get_schema(),
            description="""
            Get the database schema showing all tables, columns, and relationships.
            Use this FIRST to understand what data is available.
            Input: empty string
            """
        )

        list_tables_tool = Tool(
            name="ListTables",
            func=lambda _: ", ".join(self.db_tool.list_tables()),
            description="""
            List all table names in the database.
            Input: empty string
            """
        )

        query_tool = Tool(
            name="ExecuteSQL",
            func=self._execute_and_format,
            description="""
            Execute a SQL SELECT query and get formatted results.
            Input: A valid SQL SELECT query
            IMPORTANT: Only use SELECT queries. Always include LIMIT.
            Example: SELECT * FROM employees LIMIT 5
            """
        )

        return [schema_tool, list_tables_tool, query_tool]

    def _execute_and_format(self, query: str) -> str:
        """Execute query and format results."""
        result = self.db_tool.execute_query(query)
        return self.db_tool.format_results(result)

    def _create_agent(self):
        """Create ReAct agent with SQL tools."""

        template = """Answer the user's question about the database as best you can.
You have access to the following tools:

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

IMPORTANT GUIDELINES:
1. ALWAYS start by using GetSchema to understand the database structure
2. Write clear, optimized SQL queries
3. Use LIMIT to avoid returning too many rows
4. Explain your answer clearly
5. If data isn't available, say so

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

    def query(self, question: str):
        """Query the database with natural language."""
        return self.agent.invoke({"input": question})
```

### Step 6: Main Application

```python
# main.py
import os
from setup_database import create_sample_database
from sql_agent import SQLQueryAgent
from sql_tool import SQLDatabaseTool

def display_result(result):
    """Display query result."""
    print("\n" + "="*80)

    if result["success"]:
        print(f"✓ Answer:\n{result['answer']}")

        # Show intermediate steps if verbose
        steps = result.get("intermediate_steps", [])
        if steps:
            print(f"\n📊 Query Steps: {len(steps)}")
    else:
        print(f"✗ Error: {result['error']}")

    print("="*80)

def main():
    """Interactive SQL agent CLI."""

    print("SQL Query Agent - Natural Language Database Interface")
    print("="*80)

    # Setup database
    db_path = "company.db"
    if not os.path.exists(db_path):
        print("Creating sample database...")
        create_sample_database(db_path)

    # Initialize agent
    print("\nInitializing SQL agent...")
    agent = SQLQueryAgent(db_path, model_name="gpt-4")

    # Show schema
    print("\n📋 Database Schema:")
    print("-"*80)
    print(agent.get_schema_summary())
    print("-"*80)

    print("\nYou can now ask questions about the database in natural language.")
    print("Examples:")
    print("  - How many employees are in each department?")
    print("  - What is the average salary by department?")
    print("  - Who are the top 5 salespeople by total sales?")
    print("  - Show me employees hired in 2020")
    print("\nType 'quit' to exit, 'schema' to see schema again.\n")

    while True:
        question = input("\nYour question: ").strip()

        if question.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if question.lower() == 'schema':
            print("\n" + agent.get_schema_summary())
            continue

        if not question:
            continue

        # Process question
        result = agent.query(question)
        display_result(result)

if __name__ == "__main__":
    main()
```

## Expected Outputs

### Example 1: Simple Query
```
Your question: How many employees are in each department?

🤔 Processing question: How many employees are in each department?

Thought: I need to count employees grouped by department
Action: ExecuteSQL
Action Input: SELECT d.name, COUNT(e.id) as employee_count
              FROM departments d
              LEFT JOIN employees e ON d.id = e.department_id
              GROUP BY d.name

Observation: [Query results table]

✓ Answer:
Here are the employee counts by department:
- Engineering: 2 employees
- Sales: 2 employees
- Marketing: 2 employees
- HR: 1 employee
```

### Example 2: Complex Query
```
Your question: What is the total sales amount for each product in 2024?

✓ Answer:
Here are the total sales by product in 2024:
- Product A: $245,320.50
- Product B: $189,450.75
- Product C: $312,890.25
- Product D: $201,567.00

This data is based on 100 sales transactions recorded in the database.
```

## Bonus Challenges

1. **Query Optimization**:
   - Analyze query performance
   - Suggest indexes
   - Rewrite inefficient queries

2. **Natural Language Enhancements**:
   - Handle ambiguous questions
   - Support follow-up questions
   - Implement query clarification

3. **Visualization**:
   - Generate charts from query results
   - Create dashboard summaries
   - Export results to CSV/Excel

4. **Advanced Security**:
   - Row-level security
   - Column-level access control
   - Query audit logging
   - PII detection and masking

5. **Multi-Database Support**:
   - Connect to PostgreSQL, MySQL
   - Query multiple databases
   - Cross-database joins

6. **Query History**:
   - Store query history
   - Learn from past queries
   - Suggest similar queries

7. **Error Recovery**:
   - Auto-fix common SQL errors
   - Suggest corrections
   - Fallback strategies

## Resources

### Documentation
- [LangChain SQL Database](https://python.langchain.com/docs/use_cases/sql/)
- [SQL Agent Tutorial](https://python.langchain.com/docs/use_cases/sql/agents)
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)

### Best Practices
- [SQL Injection Prevention](https://owasp.org/www-community/attacks/SQL_Injection)
- [Database Security](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)

### Tools
- [pgvector](https://github.com/pgvector/pgvector) - Vector similarity in PostgreSQL
- [DBeaver](https://dbeaver.io/) - Database GUI
- [SQLFluff](https://www.sqlfluff.com/) - SQL linter

## Success Criteria

- [ ] Agent translates natural language to SQL accurately
- [ ] Queries execute safely with validation
- [ ] Results are formatted clearly
- [ ] Schema inspection works correctly
- [ ] Multi-table joins work properly
- [ ] Aggregations produce correct results
- [ ] Error handling prevents crashes
- [ ] Read-only mode prevents modifications
- [ ] Query limits prevent excessive data retrieval
- [ ] Code is secure and well-documented

## Testing Checklist

- [ ] Test simple SELECT queries
- [ ] Test queries with WHERE clauses
- [ ] Test JOIN operations
- [ ] Test aggregations (COUNT, SUM, AVG)
- [ ] Test GROUP BY and HAVING
- [ ] Test ORDER BY and LIMIT
- [ ] Test invalid queries
- [ ] Test SQL injection attempts
- [ ] Test with missing tables/columns
- [ ] Test query timeout handling
- [ ] Test result formatting
- [ ] Test with large result sets

## Next Steps

After completing this project:
1. Move on to Project 05: Code Generation and Execution Agent
2. Add support for other databases (PostgreSQL, MySQL)
3. Implement query caching
4. Build a web interface
5. Add data visualization capabilities
