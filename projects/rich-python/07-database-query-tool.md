# Project 7: Database Query Tool with Table Display

## Overview
Build a terminal-based database query tool that connects to SQL databases (SQLite, PostgreSQL, MySQL), executes queries, and displays results in beautifully formatted tables with syntax highlighting, query history, and export capabilities.

## Learning Objectives
- Connect to multiple database types
- Execute SQL queries safely with parameter binding
- Format query results in rich tables
- Implement SQL syntax highlighting
- Build query history and favorites
- Handle database schema inspection
- Create interactive query builders

## Difficulty Level
**Advanced** - Requires database knowledge, SQL parsing, and connection management

## Technical Stack
- **Rich**: Table, Syntax, Panel, Tree, Console, Prompt
- **Python stdlib**: sqlite3, csv, json
- **Database drivers**: psycopg2 (PostgreSQL), pymysql (MySQL)
- **Optional**: sqlparse (SQL formatting), tabulate

## Requirements

### Core Features
1. **Multi-DB Support**: SQLite, PostgreSQL, MySQL
2. **Query Execution**: Run SELECT, INSERT, UPDATE, DELETE
3. **Result Display**: Paginated table results
4. **Schema Browser**: View tables, columns, indexes
5. **Query History**: Save and recall queries
6. **Export Results**: CSV, JSON, SQL formats

### Layout Components
1. **Connection Panel**: Database connection info
2. **Query Editor**: SQL input with syntax highlighting
3. **Results Table**: Formatted query results
4. **Schema Tree**: Database structure browser
5. **History Panel**: Recent queries
6. **Statistics Bar**: Row count, execution time

### Interaction Features
- Connect to databases interactively
- Edit and execute queries
- Browse database schema
- Save favorite queries
- Export query results
- View query execution plans

## Step-by-Step Implementation

### Step 1: Database Connection Manager

```python
# db_query_tool.py
from dataclasses import dataclass
from typing import Optional, List, Dict, Any, Tuple
from pathlib import Path
import sqlite3
import time
from enum import Enum
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.tree import Tree

class DatabaseType(Enum):
    SQLITE = "sqlite"
    POSTGRESQL = "postgresql"
    MYSQL = "mysql"

@dataclass
class DatabaseConfig:
    db_type: DatabaseType
    database: str
    host: str = "localhost"
    port: int = None
    user: str = None
    password: str = None

    def __post_init__(self):
        if self.port is None:
            if self.db_type == DatabaseType.POSTGRESQL:
                self.port = 5432
            elif self.db_type == DatabaseType.MYSQL:
                self.port = 3306

class DatabaseConnection:
    def __init__(self, config: DatabaseConfig):
        self.config = config
        self.connection = None
        self.console = Console()

    def connect(self) -> bool:
        """Establish database connection."""
        try:
            if self.config.db_type == DatabaseType.SQLITE:
                self.connection = sqlite3.connect(self.config.database)
                # Enable row factory for dict-like access
                self.connection.row_factory = sqlite3.Row

            elif self.config.db_type == DatabaseType.POSTGRESQL:
                import psycopg2
                import psycopg2.extras

                self.connection = psycopg2.connect(
                    host=self.config.host,
                    port=self.config.port,
                    database=self.config.database,
                    user=self.config.user,
                    password=self.config.password
                )

            elif self.config.db_type == DatabaseType.MYSQL:
                import pymysql

                self.connection = pymysql.connect(
                    host=self.config.host,
                    port=self.config.port,
                    database=self.config.database,
                    user=self.config.user,
                    password=self.config.password
                )

            return True

        except Exception as e:
            self.console.print(f"[red]Connection failed: {e}[/red]")
            return False

    def disconnect(self):
        """Close database connection."""
        if self.connection:
            self.connection.close()
            self.connection = None

    def execute_query(
        self,
        query: str,
        params: Tuple = None
    ) -> Tuple[bool, Any, float]:
        """
        Execute SQL query and return results.
        Returns: (success, results/error, execution_time)
        """
        if not self.connection:
            return False, "Not connected to database", 0

        start_time = time.time()

        try:
            cursor = self.connection.cursor()

            if params:
                cursor.execute(query, params)
            else:
                cursor.execute(query)

            # Check if query returns results
            if query.strip().upper().startswith('SELECT') or query.strip().upper().startswith('SHOW'):
                results = cursor.fetchall()

                # Get column names
                if cursor.description:
                    columns = [desc[0] for desc in cursor.description]
                    # Convert to list of dicts
                    if self.config.db_type == DatabaseType.SQLITE:
                        results = [dict(row) for row in results]
                    else:
                        results = [dict(zip(columns, row)) for row in results]
                else:
                    results = []

                execution_time = time.time() - start_time
                return True, {'rows': results, 'columns': columns if cursor.description else []}, execution_time

            else:
                # INSERT, UPDATE, DELETE, etc.
                self.connection.commit()
                rows_affected = cursor.rowcount
                execution_time = time.time() - start_time

                return True, {'rows_affected': rows_affected}, execution_time

        except Exception as e:
            execution_time = time.time() - start_time
            return False, str(e), execution_time

        finally:
            if 'cursor' in locals():
                cursor.close()

    def get_tables(self) -> List[str]:
        """Get list of tables in database."""
        if not self.connection:
            return []

        try:
            if self.config.db_type == DatabaseType.SQLITE:
                query = "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
            elif self.config.db_type == DatabaseType.POSTGRESQL:
                query = """
                    SELECT tablename
                    FROM pg_tables
                    WHERE schemaname = 'public'
                    ORDER BY tablename
                """
            elif self.config.db_type == DatabaseType.MYSQL:
                query = "SHOW TABLES"

            success, result, _ = self.execute_query(query)

            if success and 'rows' in result:
                # Extract table names from results
                if self.config.db_type == DatabaseType.SQLITE:
                    return [row['name'] for row in result['rows']]
                elif self.config.db_type == DatabaseType.POSTGRESQL:
                    return [row['tablename'] for row in result['rows']]
                elif self.config.db_type == DatabaseType.MYSQL:
                    # MySQL returns tables as single values
                    return [list(row.values())[0] for row in result['rows']]

        except Exception as e:
            self.console.print(f"[red]Error getting tables: {e}[/red]")

        return []

    def get_table_schema(self, table_name: str) -> List[Dict]:
        """Get schema information for a table."""
        if not self.connection:
            return []

        try:
            if self.config.db_type == DatabaseType.SQLITE:
                query = f"PRAGMA table_info({table_name})"
            elif self.config.db_type == DatabaseType.POSTGRESQL:
                query = f"""
                    SELECT column_name, data_type, is_nullable
                    FROM information_schema.columns
                    WHERE table_name = '{table_name}'
                    ORDER BY ordinal_position
                """
            elif self.config.db_type == DatabaseType.MYSQL:
                query = f"DESCRIBE {table_name}"

            success, result, _ = self.execute_query(query)

            if success and 'rows' in result:
                return result['rows']

        except Exception as e:
            self.console.print(f"[red]Error getting schema: {e}[/red]")

        return []

# Usage
if __name__ == "__main__":
    # SQLite example
    config = DatabaseConfig(
        db_type=DatabaseType.SQLITE,
        database="example.db"
    )

    db = DatabaseConnection(config)
    if db.connect():
        print("Connected successfully!")

        # Get tables
        tables = db.get_tables()
        print(f"Tables: {tables}")

        db.disconnect()
```

### Step 2: Query Result Display

```python
from rich.table import Table
from rich import box

class QueryResultDisplay:
    def __init__(self, console: Console):
        self.console = console

    def display_results(
        self,
        result: Dict,
        execution_time: float,
        max_rows: int = 100
    ):
        """Display query results in a formatted table."""
        if 'rows' in result:
            rows = result['rows']
            columns = result.get('columns', [])

            if not rows:
                self.console.print("[yellow]Query returned no results[/yellow]")
                self.console.print(f"[dim]Execution time: {execution_time:.3f}s[/dim]")
                return

            # Create table
            table = Table(
                title="[bold]Query Results[/bold]",
                show_header=True,
                header_style="bold magenta",
                box=box.ROUNDED,
                show_lines=False
            )

            # Add columns
            for col in columns:
                table.add_column(col, style="cyan", overflow="fold")

            # Add rows (limit to max_rows)
            for row in rows[:max_rows]:
                # Convert all values to strings
                row_values = [self.format_value(row.get(col)) for col in columns]
                table.add_row(*row_values)

            self.console.print(table)

            # Show summary
            total_rows = len(rows)
            if total_rows > max_rows:
                self.console.print(
                    f"\n[dim]Showing {max_rows} of {total_rows} rows[/dim]"
                )
            else:
                self.console.print(f"\n[dim]Total rows: {total_rows}[/dim]")

            self.console.print(f"[dim]Execution time: {execution_time:.3f}s[/dim]")

        elif 'rows_affected' in result:
            # Non-SELECT query
            rows_affected = result['rows_affected']
            self.console.print(
                f"[green]✓[/green] Query executed successfully. "
                f"Rows affected: {rows_affected}"
            )
            self.console.print(f"[dim]Execution time: {execution_time:.3f}s[/dim]")

    def format_value(self, value: Any) -> str:
        """Format value for display."""
        if value is None:
            return "[dim]NULL[/dim]"
        elif isinstance(value, (int, float)):
            return f"[green]{value}[/green]"
        elif isinstance(value, bool):
            return f"[cyan]{value}[/cyan]"
        elif isinstance(value, bytes):
            return f"[dim]<binary data>[/dim]"
        else:
            # String or other types
            value_str = str(value)
            if len(value_str) > 100:
                value_str = value_str[:97] + "..."
            return value_str

    def display_error(self, error: str, execution_time: float):
        """Display query error."""
        panel = Panel(
            f"[red]{error}[/red]",
            title="[bold red]Query Error[/bold red]",
            border_style="red"
        )
        self.console.print(panel)
        self.console.print(f"[dim]Execution time: {execution_time:.3f}s[/dim]")

# Usage
if __name__ == "__main__":
    console = Console()
    display = QueryResultDisplay(console)

    # Sample result
    result = {
        'rows': [
            {'id': 1, 'name': 'Alice', 'age': 30},
            {'id': 2, 'name': 'Bob', 'age': 25},
            {'id': 3, 'name': 'Charlie', 'age': 35},
        ],
        'columns': ['id', 'name', 'age']
    }

    display.display_results(result, 0.042)
```

### Step 3: Schema Browser

```python
from rich.tree import Tree
from rich.panel import Panel

class SchemaExplorer:
    def __init__(self, db_connection: DatabaseConnection, console: Console):
        self.db = db_connection
        self.console = console

    def display_database_tree(self):
        """Display database schema as a tree."""
        tables = self.db.get_tables()

        if not tables:
            self.console.print("[yellow]No tables found in database[/yellow]")
            return

        tree = Tree(
            f"[bold blue]Database: {self.db.config.database}[/bold blue]",
            guide_style="dim"
        )

        for table in tables:
            # Add table branch
            table_branch = tree.add(f"📊 [cyan]{table}[/cyan]")

            # Get schema
            schema = self.db.get_table_schema(table)

            if schema:
                # Add columns
                for col_info in schema:
                    if self.db.config.db_type == DatabaseType.SQLITE:
                        col_name = col_info['name']
                        col_type = col_info['type']
                        nullable = "NULL" if col_info['notnull'] == 0 else "NOT NULL"
                        pk = " 🔑" if col_info['pk'] == 1 else ""

                        table_branch.add(
                            f"[green]{col_name}[/green] "
                            f"[yellow]{col_type}[/yellow] "
                            f"[dim]{nullable}[/dim]{pk}"
                        )

                    elif self.db.config.db_type == DatabaseType.POSTGRESQL:
                        col_name = col_info['column_name']
                        col_type = col_info['data_type']
                        nullable = col_info['is_nullable']

                        table_branch.add(
                            f"[green]{col_name}[/green] "
                            f"[yellow]{col_type}[/yellow] "
                            f"[dim]{nullable}[/dim]"
                        )

                    elif self.db.config.db_type == DatabaseType.MYSQL:
                        col_name = col_info['Field']
                        col_type = col_info['Type']
                        nullable = col_info['Null']
                        key = " 🔑" if col_info['Key'] == 'PRI' else ""

                        table_branch.add(
                            f"[green]{col_name}[/green] "
                            f"[yellow]{col_type}[/yellow] "
                            f"[dim]{nullable}[/dim]{key}"
                        )

        self.console.print(tree)

    def display_table_info(self, table_name: str):
        """Display detailed table information."""
        schema = self.db.get_table_schema(table_name)

        if not schema:
            self.console.print(f"[red]Table '{table_name}' not found[/red]")
            return

        # Create table info panel
        table_info = Table(
            title=f"[bold]Table: {table_name}[/bold]",
            show_header=True,
            header_style="bold cyan",
            box=box.ROUNDED
        )

        table_info.add_column("Column", style="green")
        table_info.add_column("Type", style="yellow")
        table_info.add_column("Nullable", style="blue")
        table_info.add_column("Key", style="magenta")
        table_info.add_column("Default", style="dim")

        for col_info in schema:
            if self.db.config.db_type == DatabaseType.SQLITE:
                table_info.add_row(
                    col_info['name'],
                    col_info['type'],
                    "YES" if col_info['notnull'] == 0 else "NO",
                    "PRIMARY" if col_info['pk'] == 1 else "",
                    str(col_info.get('dflt_value', ''))
                )

            elif self.db.config.db_type == DatabaseType.POSTGRESQL:
                table_info.add_row(
                    col_info['column_name'],
                    col_info['data_type'],
                    col_info['is_nullable'],
                    "",
                    ""
                )

            elif self.db.config.db_type == DatabaseType.MYSQL:
                table_info.add_row(
                    col_info['Field'],
                    col_info['Type'],
                    col_info['Null'],
                    col_info.get('Key', ''),
                    str(col_info.get('Default', ''))
                )

        self.console.print(table_info)

        # Show row count
        success, result, _ = self.db.execute_query(f"SELECT COUNT(*) as count FROM {table_name}")
        if success and 'rows' in result and result['rows']:
            count = list(result['rows'][0].values())[0]
            self.console.print(f"\n[dim]Total rows: {count:,}[/dim]")

    def generate_select_query(self, table_name: str) -> str:
        """Generate a SELECT query template for a table."""
        schema = self.db.get_table_schema(table_name)

        if not schema:
            return ""

        if self.db.config.db_type == DatabaseType.SQLITE:
            columns = [col['name'] for col in schema]
        elif self.db.config.db_type == DatabaseType.POSTGRESQL:
            columns = [col['column_name'] for col in schema]
        elif self.db.config.db_type == DatabaseType.MYSQL:
            columns = [col['Field'] for col in schema]

        columns_str = ",\n    ".join(columns)

        query = f"""SELECT
    {columns_str}
FROM {table_name}
LIMIT 10;"""

        return query

# Usage
if __name__ == "__main__":
    console = Console()
    config = DatabaseConfig(db_type=DatabaseType.SQLITE, database="example.db")
    db = DatabaseConnection(config)

    if db.connect():
        explorer = SchemaExplorer(db, console)

        # Display database tree
        explorer.display_database_tree()

        # Display table info
        explorer.display_table_info("users")

        # Generate query
        query = explorer.generate_select_query("users")
        print(query)

        db.disconnect()
```

### Step 4: Query History and Favorites

```python
import json
from datetime import datetime
from pathlib import Path

@dataclass
class SavedQuery:
    query: str
    name: str
    created_at: datetime
    last_used: Optional[datetime] = None
    use_count: int = 0
    tags: List[str] = None

    def __post_init__(self):
        if self.tags is None:
            self.tags = []

class QueryHistory:
    def __init__(self, history_file: str = "query_history.json"):
        self.history_file = Path(history_file)
        self.queries: List[SavedQuery] = []
        self.recent_queries: List[str] = []
        self.load()

    def load(self):
        """Load query history from file."""
        if self.history_file.exists():
            try:
                with open(self.history_file, 'r') as f:
                    data = json.load(f)

                    # Load saved queries
                    for q in data.get('saved', []):
                        query = SavedQuery(
                            query=q['query'],
                            name=q['name'],
                            created_at=datetime.fromisoformat(q['created_at']),
                            last_used=datetime.fromisoformat(q['last_used']) if q.get('last_used') else None,
                            use_count=q.get('use_count', 0),
                            tags=q.get('tags', [])
                        )
                        self.queries.append(query)

                    # Load recent queries
                    self.recent_queries = data.get('recent', [])

            except Exception as e:
                print(f"Error loading history: {e}")

    def save(self):
        """Save query history to file."""
        try:
            data = {
                'saved': [
                    {
                        'query': q.query,
                        'name': q.name,
                        'created_at': q.created_at.isoformat(),
                        'last_used': q.last_used.isoformat() if q.last_used else None,
                        'use_count': q.use_count,
                        'tags': q.tags
                    }
                    for q in self.queries
                ],
                'recent': self.recent_queries[-50:]  # Keep last 50
            }

            with open(self.history_file, 'w') as f:
                json.dump(data, f, indent=2)

        except Exception as e:
            print(f"Error saving history: {e}")

    def add_recent(self, query: str):
        """Add query to recent history."""
        # Remove if already exists
        if query in self.recent_queries:
            self.recent_queries.remove(query)

        # Add to beginning
        self.recent_queries.insert(0, query)

        # Limit to 50 queries
        self.recent_queries = self.recent_queries[:50]

        self.save()

    def save_query(self, query: str, name: str, tags: List[str] = None):
        """Save a query as favorite."""
        saved = SavedQuery(
            query=query,
            name=name,
            created_at=datetime.now(),
            tags=tags or []
        )

        self.queries.append(saved)
        self.save()

    def get_saved_query(self, name: str) -> Optional[SavedQuery]:
        """Get saved query by name."""
        for query in self.queries:
            if query.name == name:
                query.last_used = datetime.now()
                query.use_count += 1
                self.save()
                return query
        return None

    def delete_saved_query(self, name: str) -> bool:
        """Delete a saved query."""
        for i, query in enumerate(self.queries):
            if query.name == name:
                self.queries.pop(i)
                self.save()
                return True
        return False

    def display_recent(self, console: Console, count: int = 10):
        """Display recent queries."""
        table = Table(
            title="[bold]Recent Queries[/bold]",
            show_header=True,
            header_style="bold cyan",
            box=box.ROUNDED
        )

        table.add_column("#", style="dim", width=4)
        table.add_column("Query", style="white", overflow="fold")

        for i, query in enumerate(self.recent_queries[:count], 1):
            # Truncate long queries
            display_query = query
            if len(display_query) > 100:
                display_query = display_query[:97] + "..."

            table.add_row(str(i), display_query)

        console.print(table)

    def display_saved(self, console: Console):
        """Display saved queries."""
        if not self.queries:
            console.print("[yellow]No saved queries[/yellow]")
            return

        table = Table(
            title="[bold]Saved Queries[/bold]",
            show_header=True,
            header_style="bold cyan",
            box=box.ROUNDED
        )

        table.add_column("Name", style="green", width=20)
        table.add_column("Query", style="white", overflow="fold")
        table.add_column("Tags", style="blue", width=15)
        table.add_column("Used", style="yellow", width=8)

        for query in sorted(self.queries, key=lambda q: q.use_count, reverse=True):
            display_query = query.query
            if len(display_query) > 80:
                display_query = display_query[:77] + "..."

            tags_str = ", ".join(query.tags[:3])

            table.add_row(
                query.name,
                display_query,
                tags_str,
                str(query.use_count)
            )

        console.print(table)

# Usage
if __name__ == "__main__":
    history = QueryHistory()

    # Add recent query
    history.add_recent("SELECT * FROM users WHERE age > 18")

    # Save favorite query
    history.save_query(
        "SELECT * FROM users WHERE active = 1",
        "active_users",
        tags=["users", "common"]
    )

    # Display
    console = Console()
    history.display_recent(console)
    history.display_saved(console)
```

### Step 5: Interactive Query Tool

```python
from rich.syntax import Syntax
from rich.prompt import Prompt, Confirm

class InteractiveDatabaseTool:
    def __init__(self, db: DatabaseConnection):
        self.db = db
        self.console = Console()
        self.display = QueryResultDisplay(self.console)
        self.explorer = SchemaExplorer(db, self.console)
        self.history = QueryHistory()

    def display_connection_info(self):
        """Display current connection information."""
        info_text = f"""[bold]Connection Info[/bold]

Database Type: [cyan]{self.db.config.db_type.value}[/cyan]
Database: [green]{self.db.config.database}[/green]"""

        if self.db.config.host:
            info_text += f"\nHost: [blue]{self.db.config.host}:{self.db.config.port}[/blue]"

        panel = Panel(info_text, border_style="blue")
        self.console.print(panel)

    def execute_interactive_query(self, query: str):
        """Execute query and display results."""
        # Add to history
        self.history.add_recent(query)

        # Display query with syntax highlighting
        syntax = Syntax(query, "sql", theme="monokai", line_numbers=False)
        self.console.print("\n[bold]Executing Query:[/bold]")
        self.console.print(syntax)
        self.console.print()

        # Execute
        success, result, exec_time = self.db.execute_query(query)

        if success:
            self.display.display_results(result, exec_time)
        else:
            self.display.display_error(result, exec_time)

    def run_interactive_mode(self):
        """Run interactive query mode."""
        self.console.clear()
        self.console.print("[bold magenta]Database Query Tool[/bold magenta]\n")

        self.display_connection_info()

        while True:
            self.console.print("\n[bold cyan]Options:[/bold cyan]")
            self.console.print("  1. Execute query")
            self.console.print("  2. Browse schema")
            self.console.print("  3. View recent queries")
            self.console.print("  4. View saved queries")
            self.console.print("  5. Save current query")
            self.console.print("  6. Load saved query")
            self.console.print("  7. Export results")
            self.console.print("  0. Exit")

            choice = Prompt.ask("\nSelect option", choices=["0","1","2","3","4","5","6","7"], default="1")

            if choice == "0":
                self.console.print("[green]Goodbye![/green]")
                break

            elif choice == "1":
                # Execute query
                self.console.print("\n[dim]Enter SQL query (or 'cancel' to abort):[/dim]")

                lines = []
                while True:
                    line = Prompt.ask("", default="")
                    if line.lower() == 'cancel':
                        break
                    if line.endswith(';'):
                        lines.append(line)
                        break
                    lines.append(line)

                query = '\n'.join(lines)

                if query and query.lower() != 'cancel':
                    self.execute_interactive_query(query)

            elif choice == "2":
                # Browse schema
                self.console.print()
                self.explorer.display_database_tree()

                # Ask if user wants table details
                if Confirm.ask("\nView table details?", default=False):
                    table_name = Prompt.ask("Enter table name")
                    self.explorer.display_table_info(table_name)

                    # Offer to generate query
                    if Confirm.ask("\nGenerate SELECT query?", default=True):
                        query = self.explorer.generate_select_query(table_name)
                        syntax = Syntax(query, "sql", theme="monokai")
                        self.console.print(syntax)

                        if Confirm.ask("Execute this query?", default=False):
                            self.execute_interactive_query(query)

            elif choice == "3":
                # Recent queries
                self.console.print()
                self.history.display_recent(self.console)

                if self.history.recent_queries:
                    if Confirm.ask("\nExecute a recent query?", default=False):
                        idx = int(Prompt.ask("Enter query number")) - 1
                        if 0 <= idx < len(self.history.recent_queries):
                            query = self.history.recent_queries[idx]
                            self.execute_interactive_query(query)

            elif choice == "4":
                # Saved queries
                self.console.print()
                self.history.display_saved(self.console)

            elif choice == "5":
                # Save query
                if self.history.recent_queries:
                    query = self.history.recent_queries[0]
                    self.console.print(f"\n[dim]Saving: {query[:50]}...[/dim]")

                    name = Prompt.ask("Query name")
                    tags_input = Prompt.ask("Tags (comma-separated)", default="")
                    tags = [t.strip() for t in tags_input.split(",") if t.strip()]

                    self.history.save_query(query, name, tags)
                    self.console.print("[green]✓[/green] Query saved!")

            elif choice == "6":
                # Load saved query
                self.history.display_saved(self.console)

                if self.history.queries:
                    name = Prompt.ask("\nQuery name to load")
                    saved = self.history.get_saved_query(name)

                    if saved:
                        if Confirm.ask(f"Execute '{name}'?", default=True):
                            self.execute_interactive_query(saved.query)
                    else:
                        self.console.print(f"[red]Query '{name}' not found[/red]")

            elif choice == "7":
                # Export results (placeholder)
                self.console.print("[yellow]Export feature coming soon![/yellow]")

# Usage
if __name__ == "__main__":
    config = DatabaseConfig(
        db_type=DatabaseType.SQLITE,
        database="example.db"
    )

    db = DatabaseConnection(config)

    if db.connect():
        tool = InteractiveDatabaseTool(db)
        tool.run_interactive_mode()
        db.disconnect()
```

## Expected Output

### Query Results
```
╭─ Query Results ──────────────────────────────────────────╮
│ id │ name    │ email              │ age │ active        │
│ 1  │ Alice   │ alice@example.com  │ 30  │ True          │
│ 2  │ Bob     │ bob@example.com    │ 25  │ True          │
│ 3  │ Charlie │ charlie@example.com│ 35  │ False         │
╰──────────────────────────────────────────────────────────╯
Total rows: 3
Execution time: 0.042s
```

### Schema Tree
```
Database: myapp.db
├── 📊 users
│   ├── id INTEGER NOT NULL 🔑
│   ├── name TEXT NOT NULL
│   ├── email TEXT NULL
│   └── created_at TIMESTAMP NULL
├── 📊 products
│   ├── id INTEGER NOT NULL 🔑
│   ├── name TEXT NOT NULL
│   └── price REAL NOT NULL
└── 📊 orders
    ├── id INTEGER NOT NULL 🔑
    ├── user_id INTEGER NOT NULL
    └── total REAL NOT NULL
```

## Bonus Challenges

1. **Query Builder GUI**: Visual query builder interface
2. **Auto-completion**: SQL keyword and table/column completion
3. **Query Optimization**: EXPLAIN PLAN visualization
4. **Data Editing**: Update records interactively
5. **Import Data**: CSV/JSON import functionality
6. **Backup/Restore**: Database backup tools
7. **Multi-DB Queries**: Join across databases
8. **Transaction Support**: BEGIN/COMMIT/ROLLBACK
9. **Stored Procedures**: Execute procedures/functions
10. **Performance Monitoring**: Query performance tracking

## Resources

- [SQLite Documentation](https://www.sqlite.org/docs.html)
- [psycopg2 - PostgreSQL](https://www.psycopg.org/)
- [PyMySQL - MySQL](https://pymysql.readthedocs.io/)
- [sqlparse - SQL Parser](https://sqlparse.readthedocs.io/)
- [Rich Tables](https://rich.readthedocs.io/en/latest/tables.html)

## Success Criteria

- [ ] Connect to multiple database types
- [ ] Execute SQL queries safely
- [ ] Display results in formatted tables
- [ ] Browse database schema
- [ ] Handle NULL values correctly
- [ ] Show execution time
- [ ] Maintain query history
- [ ] Save favorite queries
- [ ] Export results to CSV/JSON
- [ ] Handle errors gracefully

## Testing Checklist

- Test with SQLite, PostgreSQL, MySQL
- Execute various query types (SELECT, INSERT, UPDATE, DELETE)
- Test with large result sets (1000+ rows)
- Verify NULL handling
- Test with special characters in data
- Check error handling for invalid queries
- Test connection failure scenarios
- Verify data type formatting
- Test with empty tables
- Check transaction handling
