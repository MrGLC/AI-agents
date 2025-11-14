# Project 9: Code Snippet Manager with Search

## Overview
Build a powerful terminal-based code snippet manager that stores, organizes, searches, and displays code snippets with syntax highlighting, tags, and categories - your personal developer knowledge base in the terminal.

## Learning Objectives
- Implement full-text search across code snippets
- Create syntax highlighting for multiple languages
- Build tagging and categorization systems
- Design efficient snippet storage and retrieval
- Implement code execution capabilities
- Create import/export functionality

## Difficulty Level
**Intermediate to Advanced** - Requires text search, syntax highlighting, and data management

## Technical Stack
- **Rich**: Syntax, Table, Panel, Tree, Prompt, Console
- **Python stdlib**: json, sqlite3, re, subprocess
- **pygments**: Syntax highlighting for many languages
- **Optional**: whoosh (full-text search), clipboard (copy to clipboard)

## Requirements

### Core Features
1. **Snippet Storage**: Create, read, update, delete snippets
2. **Syntax Highlighting**: Support for 50+ languages
3. **Search**: Full-text search with filters
4. **Tags**: Flexible tagging system
5. **Categories**: Organize by language/topic
6. **Favorites**: Star important snippets

### Layout Components
1. **Snippet List**: Browse all snippets
2. **Snippet View**: Display with syntax highlighting
3. **Search Results**: Filtered snippet list
4. **Category Tree**: Hierarchical organization
5. **Tag Cloud**: Visual tag representation
6. **Edit Form**: Interactive snippet creation

### Interaction Features
- Quick search by language, tags, or content
- Copy snippet to clipboard
- Execute code snippets
- Import from files or GitHub gists
- Export snippets to files
- Batch operations

## Step-by-Step Implementation

### Step 1: Snippet Data Model

```python
# snippet_manager.py
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import List, Optional, Dict
from pathlib import Path
import json
import sqlite3
from rich.console import Console

@dataclass
class CodeSnippet:
    id: int
    title: str
    code: str
    language: str
    description: str = ""
    tags: List[str] = None
    category: str = "General"
    favorite: bool = False
    created_at: datetime = None
    updated_at: datetime = None
    usage_count: int = 0

    def __post_init__(self):
        if self.tags is None:
            self.tags = []
        if self.created_at is None:
            self.created_at = datetime.now()
        if self.updated_at is None:
            self.updated_at = datetime.now()

class SnippetDatabase:
    def __init__(self, db_path: str = "snippets.db"):
        self.db_path = db_path
        self.console = Console()
        self.init_database()

    def init_database(self):
        """Initialize SQLite database."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        # Create snippets table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS snippets (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                code TEXT NOT NULL,
                language TEXT NOT NULL,
                description TEXT,
                category TEXT,
                favorite INTEGER DEFAULT 0,
                created_at TEXT,
                updated_at TEXT,
                usage_count INTEGER DEFAULT 0
            )
        """)

        # Create tags table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS tags (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                snippet_id INTEGER,
                tag TEXT,
                FOREIGN KEY (snippet_id) REFERENCES snippets (id) ON DELETE CASCADE
            )
        """)

        # Create index for faster searches
        cursor.execute("""
            CREATE INDEX IF NOT EXISTS idx_language ON snippets(language)
        """)
        cursor.execute("""
            CREATE INDEX IF NOT EXISTS idx_category ON snippets(category)
        """)
        cursor.execute("""
            CREATE INDEX IF NOT EXISTS idx_tags ON tags(tag)
        """)

        conn.commit()
        conn.close()

    def add_snippet(self, snippet: CodeSnippet) -> int:
        """Add a new snippet."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO snippets (title, code, language, description, category,
                                 favorite, created_at, updated_at, usage_count)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            snippet.title,
            snippet.code,
            snippet.language,
            snippet.description,
            snippet.category,
            1 if snippet.favorite else 0,
            snippet.created_at.isoformat(),
            snippet.updated_at.isoformat(),
            snippet.usage_count
        ))

        snippet_id = cursor.lastrowid

        # Add tags
        for tag in snippet.tags:
            cursor.execute("INSERT INTO tags (snippet_id, tag) VALUES (?, ?)",
                         (snippet_id, tag))

        conn.commit()
        conn.close()

        return snippet_id

    def get_snippet(self, snippet_id: int) -> Optional[CodeSnippet]:
        """Get snippet by ID."""
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()

        cursor.execute("SELECT * FROM snippets WHERE id = ?", (snippet_id,))
        row = cursor.fetchone()

        if not row:
            conn.close()
            return None

        # Get tags
        cursor.execute("SELECT tag FROM tags WHERE snippet_id = ?", (snippet_id,))
        tags = [tag_row['tag'] for tag_row in cursor.fetchall()]

        conn.close()

        return CodeSnippet(
            id=row['id'],
            title=row['title'],
            code=row['code'],
            language=row['language'],
            description=row['description'],
            category=row['category'],
            favorite=bool(row['favorite']),
            created_at=datetime.fromisoformat(row['created_at']),
            updated_at=datetime.fromisoformat(row['updated_at']),
            usage_count=row['usage_count'],
            tags=tags
        )

    def search_snippets(
        self,
        query: str = "",
        language: str = None,
        category: str = None,
        tags: List[str] = None,
        favorite_only: bool = False
    ) -> List[CodeSnippet]:
        """Search snippets with filters."""
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()

        # Build query
        sql = "SELECT DISTINCT s.* FROM snippets s"

        if tags:
            sql += " JOIN tags t ON s.id = t.snippet_id"

        where_clauses = []
        params = []

        if query:
            where_clauses.append("(s.title LIKE ? OR s.description LIKE ? OR s.code LIKE ?)")
            search_term = f"%{query}%"
            params.extend([search_term, search_term, search_term])

        if language:
            where_clauses.append("s.language = ?")
            params.append(language)

        if category:
            where_clauses.append("s.category = ?")
            params.append(category)

        if tags:
            where_clauses.append(f"t.tag IN ({','.join('?' * len(tags))})")
            params.extend(tags)

        if favorite_only:
            where_clauses.append("s.favorite = 1")

        if where_clauses:
            sql += " WHERE " + " AND ".join(where_clauses)

        sql += " ORDER BY s.updated_at DESC"

        cursor.execute(sql, params)
        rows = cursor.fetchall()

        snippets = []
        for row in rows:
            # Get tags for this snippet
            cursor.execute("SELECT tag FROM tags WHERE snippet_id = ?", (row['id'],))
            snippet_tags = [tag_row['tag'] for tag_row in cursor.fetchall()]

            snippets.append(CodeSnippet(
                id=row['id'],
                title=row['title'],
                code=row['code'],
                language=row['language'],
                description=row['description'],
                category=row['category'],
                favorite=bool(row['favorite']),
                created_at=datetime.fromisoformat(row['created_at']),
                updated_at=datetime.fromisoformat(row['updated_at']),
                usage_count=row['usage_count'],
                tags=snippet_tags
            ))

        conn.close()
        return snippets

    def update_snippet(self, snippet: CodeSnippet):
        """Update existing snippet."""
        snippet.updated_at = datetime.now()

        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            UPDATE snippets
            SET title=?, code=?, language=?, description=?, category=?,
                favorite=?, updated_at=?, usage_count=?
            WHERE id=?
        """, (
            snippet.title,
            snippet.code,
            snippet.language,
            snippet.description,
            snippet.category,
            1 if snippet.favorite else 0,
            snippet.updated_at.isoformat(),
            snippet.usage_count,
            snippet.id
        ))

        # Update tags
        cursor.execute("DELETE FROM tags WHERE snippet_id = ?", (snippet.id,))
        for tag in snippet.tags:
            cursor.execute("INSERT INTO tags (snippet_id, tag) VALUES (?, ?)",
                         (snippet.id, tag))

        conn.commit()
        conn.close()

    def delete_snippet(self, snippet_id: int):
        """Delete snippet."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("DELETE FROM snippets WHERE id = ?", (snippet_id,))
        cursor.execute("DELETE FROM tags WHERE snippet_id = ?", (snippet_id,))

        conn.commit()
        conn.close()

    def get_all_languages(self) -> List[str]:
        """Get list of all languages."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("SELECT DISTINCT language FROM snippets ORDER BY language")
        languages = [row[0] for row in cursor.fetchall()]

        conn.close()
        return languages

    def get_all_categories(self) -> List[str]:
        """Get list of all categories."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("SELECT DISTINCT category FROM snippets ORDER BY category")
        categories = [row[0] for row in cursor.fetchall()]

        conn.close()
        return categories

    def get_all_tags(self) -> Dict[str, int]:
        """Get all tags with usage counts."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT tag, COUNT(*) as count
            FROM tags
            GROUP BY tag
            ORDER BY count DESC
        """)

        tags = {row[0]: row[1] for row in cursor.fetchall()}

        conn.close()
        return tags

# Usage
if __name__ == "__main__":
    db = SnippetDatabase()

    # Add a snippet
    snippet = CodeSnippet(
        id=0,
        title="Quick Sort Implementation",
        code="""def quicksort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[len(arr) // 2]
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return quicksort(left) + middle + quicksort(right)""",
        language="python",
        description="Efficient quicksort algorithm implementation",
        tags=["algorithm", "sorting", "recursion"],
        category="Algorithms"
    )

    snippet_id = db.add_snippet(snippet)
    print(f"Added snippet with ID: {snippet_id}")
```

### Step 2: Snippet Display with Syntax Highlighting

```python
from rich.syntax import Syntax
from rich.panel import Panel
from rich.table import Table
from rich import box
from rich.text import Text

class SnippetDisplay:
    def __init__(self, console: Console):
        self.console = console

    def display_snippet(self, snippet: CodeSnippet, show_code: bool = True):
        """Display a single snippet with syntax highlighting."""
        # Create header with metadata
        header = Text()
        header.append(snippet.title, style="bold cyan")
        header.append(f" ({snippet.language})", style="dim")

        if snippet.favorite:
            header.append(" ⭐", style="yellow")

        # Description
        content = []

        if snippet.description:
            content.append(Text(snippet.description, style="white"))
            content.append(Text())

        # Metadata
        meta = Text()
        meta.append(f"Category: ", style="dim")
        meta.append(snippet.category, style="blue")
        meta.append(f" | Tags: ", style="dim")
        meta.append(", ".join(snippet.tags), style="green")
        meta.append(f" | Used: ", style="dim")
        meta.append(f"{snippet.usage_count} times", style="yellow")

        content.append(meta)

        # Code with syntax highlighting
        if show_code:
            content.append(Text())
            syntax = Syntax(
                snippet.code,
                snippet.language,
                theme="monokai",
                line_numbers=True,
                word_wrap=True
            )
            content.append(syntax)

        # Combine content
        panel_content = Text("\n").join(content) if isinstance(content[0], Text) else content[-1]

        # Create panel
        panel = Panel(
            panel_content,
            title=header,
            subtitle=f"[dim]ID: {snippet.id} | Updated: {snippet.updated_at.strftime('%Y-%m-%d %H:%M')}[/dim]",
            border_style="cyan"
        )

        self.console.print(panel)

    def display_snippet_list(self, snippets: List[CodeSnippet], max_results: int = 50):
        """Display list of snippets in a table."""
        if not snippets:
            self.console.print("[yellow]No snippets found[/yellow]")
            return

        table = Table(
            title=f"[bold]Code Snippets ({len(snippets)} found)[/bold]",
            show_header=True,
            header_style="bold magenta",
            box=box.ROUNDED
        )

        table.add_column("ID", style="cyan", width=5)
        table.add_column("⭐", width=3)
        table.add_column("Title", style="white", overflow="ellipsis", width=30)
        table.add_column("Language", style="green", width=12)
        table.add_column("Category", style="blue", width=15)
        table.add_column("Tags", style="yellow", width=20, overflow="ellipsis")
        table.add_column("Updated", style="dim", width=12)

        for snippet in snippets[:max_results]:
            star = "⭐" if snippet.favorite else ""
            tags_str = ", ".join(snippet.tags[:3])
            if len(snippet.tags) > 3:
                tags_str += "..."

            table.add_row(
                str(snippet.id),
                star,
                snippet.title,
                snippet.language,
                snippet.category,
                tags_str,
                snippet.updated_at.strftime("%Y-%m-%d")
            )

        self.console.print(table)

        if len(snippets) > max_results:
            self.console.print(f"\n[dim]Showing {max_results} of {len(snippets)} snippets[/dim]")

    def display_tag_cloud(self, tags: Dict[str, int]):
        """Display tag cloud with usage counts."""
        if not tags:
            self.console.print("[yellow]No tags found[/yellow]")
            return

        # Sort by count
        sorted_tags = sorted(tags.items(), key=lambda x: x[1], reverse=True)

        table = Table(
            title="[bold]Tag Cloud[/bold]",
            show_header=True,
            header_style="bold cyan",
            box=box.SIMPLE
        )

        table.add_column("Tag", style="green")
        table.add_column("Count", justify="right", style="yellow")
        table.add_column("Bar", style="cyan")

        max_count = max(tags.values())

        for tag, count in sorted_tags[:30]:  # Show top 30
            # Create visual bar
            bar_length = int((count / max_count) * 30)
            bar = "█" * bar_length

            table.add_row(tag, str(count), bar)

        self.console.print(table)

    def display_category_tree(self, db: SnippetDatabase):
        """Display snippets organized by category."""
        from rich.tree import Tree

        categories = db.get_all_categories()

        tree = Tree(
            "[bold blue]Categories[/bold blue]",
            guide_style="dim"
        )

        for category in categories:
            snippets = db.search_snippets(category=category)
            category_branch = tree.add(
                f"[cyan]{category}[/cyan] [dim]({len(snippets)})[/dim]"
            )

            # Group by language
            by_language = {}
            for snippet in snippets:
                if snippet.language not in by_language:
                    by_language[snippet.language] = []
                by_language[snippet.language].append(snippet)

            for language, lang_snippets in sorted(by_language.items()):
                lang_branch = category_branch.add(
                    f"[green]{language}[/green] [dim]({len(lang_snippets)})[/dim]"
                )

                for snippet in lang_snippets[:5]:  # Show first 5
                    star = "⭐ " if snippet.favorite else ""
                    lang_branch.add(f"{star}{snippet.title} [dim](#{snippet.id})[/dim]")

        self.console.print(tree)

# Usage
if __name__ == "__main__":
    console = Console()
    db = SnippetDatabase()
    display = SnippetDisplay(console)

    # Display all snippets
    snippets = db.search_snippets()
    display.display_snippet_list(snippets)

    # Display single snippet
    if snippets:
        display.display_snippet(snippets[0])

    # Display tag cloud
    tags = db.get_all_tags()
    display.display_tag_cloud(tags)
```

### Step 3: Interactive Snippet Creation and Editing

```python
from rich.prompt import Prompt, Confirm
import tempfile
import subprocess
import os

class SnippetEditor:
    def __init__(self, db: SnippetDatabase, console: Console):
        self.db = db
        self.console = console

    def create_snippet_interactive(self) -> Optional[CodeSnippet]:
        """Interactively create a new snippet."""
        self.console.print(Panel("[bold cyan]Create New Snippet[/bold cyan]", border_style="cyan"))

        # Title
        title = Prompt.ask("\n[bold]Title[/bold]")

        # Language
        languages = self.db.get_all_languages()
        if languages:
            self.console.print("\n[dim]Existing languages:[/dim]", ", ".join(languages[:10]))

        language = Prompt.ask("[bold]Language[/bold]", default="python")

        # Category
        categories = self.db.get_all_categories()
        if categories:
            self.console.print("\n[dim]Existing categories:[/dim]", ", ".join(categories))

        category = Prompt.ask("[bold]Category[/bold]", default="General")

        # Description
        description = Prompt.ask("[bold]Description[/bold] (optional)", default="")

        # Tags
        tags_input = Prompt.ask("[bold]Tags[/bold] (comma-separated)", default="")
        tags = [tag.strip() for tag in tags_input.split(",") if tag.strip()]

        # Code input
        self.console.print("\n[bold]Code:[/bold]")
        self.console.print("[dim]Enter code (type 'END' on a new line when done):[/dim]\n")

        code_lines = []
        while True:
            line = input()
            if line.strip() == "END":
                break
            code_lines.append(line)

        code = "\n".join(code_lines)

        if not code.strip():
            self.console.print("[red]Code cannot be empty![/red]")
            return None

        # Favorite
        favorite = Confirm.ask("\n[bold]Mark as favorite?[/bold]", default=False)

        # Create snippet
        snippet = CodeSnippet(
            id=0,
            title=title,
            code=code,
            language=language,
            description=description,
            category=category,
            tags=tags,
            favorite=favorite
        )

        # Preview
        self.console.print("\n[bold]Preview:[/bold]")
        display = SnippetDisplay(self.console)
        display.display_snippet(snippet)

        # Confirm
        if Confirm.ask("\n[bold]Save this snippet?[/bold]", default=True):
            snippet_id = self.db.add_snippet(snippet)
            snippet.id = snippet_id
            self.console.print(f"\n[green]✓[/green] Snippet saved with ID: {snippet_id}")
            return snippet
        else:
            self.console.print("[dim]Snippet not saved[/dim]")
            return None

    def edit_snippet_interactive(self, snippet_id: int):
        """Interactively edit an existing snippet."""
        snippet = self.db.get_snippet(snippet_id)

        if not snippet:
            self.console.print(f"[red]Snippet {snippet_id} not found[/red]")
            return

        self.console.print(f"\n[bold]Editing Snippet #{snippet_id}[/bold]\n")

        # Show current values and allow editing
        snippet.title = Prompt.ask("Title", default=snippet.title)
        snippet.language = Prompt.ask("Language", default=snippet.language)
        snippet.category = Prompt.ask("Category", default=snippet.category)
        snippet.description = Prompt.ask("Description", default=snippet.description)

        tags_str = ", ".join(snippet.tags)
        tags_input = Prompt.ask("Tags", default=tags_str)
        snippet.tags = [tag.strip() for tag in tags_input.split(",") if tag.strip()]

        # Edit code
        if Confirm.ask("Edit code?", default=False):
            # Open in external editor
            editor = os.environ.get('EDITOR', 'nano')

            with tempfile.NamedTemporaryFile(mode='w+', suffix=f'.{snippet.language}', delete=False) as tf:
                tf.write(snippet.code)
                tf.flush()
                temp_path = tf.name

            try:
                subprocess.call([editor, temp_path])

                with open(temp_path, 'r') as f:
                    snippet.code = f.read()

            finally:
                os.unlink(temp_path)

        snippet.favorite = Confirm.ask("Favorite?", default=snippet.favorite)

        # Save
        self.db.update_snippet(snippet)
        self.console.print(f"\n[green]✓[/green] Snippet updated!")

    def execute_snippet(self, snippet: CodeSnippet):
        """Execute a code snippet (with caution!)."""
        self.console.print(f"\n[yellow]⚠️  About to execute snippet: {snippet.title}[/yellow]")

        # Show code
        syntax = Syntax(snippet.code, snippet.language, theme="monokai", line_numbers=True)
        self.console.print(syntax)

        if not Confirm.ask("\n[bold red]Execute this code?[/bold red]", default=False):
            self.console.print("[dim]Execution cancelled[/dim]")
            return

        # Execute based on language
        try:
            if snippet.language.lower() == 'python':
                exec(snippet.code)

            elif snippet.language.lower() == 'bash':
                result = subprocess.run(
                    snippet.code,
                    shell=True,
                    capture_output=True,
                    text=True
                )
                self.console.print(result.stdout)
                if result.stderr:
                    self.console.print(f"[red]{result.stderr}[/red]")

            else:
                self.console.print(f"[yellow]Execution not supported for {snippet.language}[/yellow]")

            # Increment usage count
            snippet.usage_count += 1
            self.db.update_snippet(snippet)

        except Exception as e:
            self.console.print(f"[red]Execution error: {e}[/red]")

# Usage
if __name__ == "__main__":
    console = Console()
    db = SnippetDatabase()
    editor = SnippetEditor(db, console)

    # Create new snippet
    editor.create_snippet_interactive()
```

### Step 4: Main Interactive Application

```python
from enum import Enum

class MenuOption(Enum):
    LIST = "1"
    SEARCH = "2"
    VIEW = "3"
    CREATE = "4"
    EDIT = "5"
    DELETE = "6"
    EXECUTE = "7"
    TAGS = "8"
    CATEGORIES = "9"
    FAVORITES = "10"
    EXPORT = "11"
    EXIT = "0"

class SnippetManagerApp:
    def __init__(self):
        self.console = Console()
        self.db = SnippetDatabase()
        self.display = SnippetDisplay(self.console)
        self.editor = SnippetEditor(self.db, self.console)

    def display_menu(self):
        """Display main menu."""
        menu_text = """
[bold cyan]Code Snippet Manager[/bold cyan]

[1]  📋 List all snippets
[2]  🔍 Search snippets
[3]  👁️  View snippet
[4]  ➕ Create new snippet
[5]  ✏️  Edit snippet
[6]  🗑️  Delete snippet
[7]  ▶️  Execute snippet
[8]  🏷️  View tags
[9]  📁 View categories
[10] ⭐ View favorites
[11] 💾 Export snippet
[0]  🚪 Exit
"""
        self.console.print(Panel(menu_text, border_style="blue"))

    def run(self):
        """Run the main application loop."""
        self.console.clear()

        while True:
            self.display_menu()

            choice = Prompt.ask(
                "Select option",
                choices=["0","1","2","3","4","5","6","7","8","9","10","11"],
                default="1"
            )

            self.console.print()

            if choice == MenuOption.EXIT.value:
                self.console.print("[green]Goodbye![/green]")
                break

            elif choice == MenuOption.LIST.value:
                snippets = self.db.search_snippets()
                self.display.display_snippet_list(snippets)

            elif choice == MenuOption.SEARCH.value:
                query = Prompt.ask("Search query (title/description/code)")
                language = Prompt.ask("Filter by language (optional)", default="")
                category = Prompt.ask("Filter by category (optional)", default="")

                snippets = self.db.search_snippets(
                    query=query,
                    language=language if language else None,
                    category=category if category else None
                )

                self.display.display_snippet_list(snippets)

            elif choice == MenuOption.VIEW.value:
                snippet_id = int(Prompt.ask("Enter snippet ID"))
                snippet = self.db.get_snippet(snippet_id)

                if snippet:
                    self.display.display_snippet(snippet, show_code=True)
                else:
                    self.console.print(f"[red]Snippet {snippet_id} not found[/red]")

            elif choice == MenuOption.CREATE.value:
                self.editor.create_snippet_interactive()

            elif choice == MenuOption.EDIT.value:
                snippet_id = int(Prompt.ask("Enter snippet ID"))
                self.editor.edit_snippet_interactive(snippet_id)

            elif choice == MenuOption.DELETE.value:
                snippet_id = int(Prompt.ask("Enter snippet ID"))
                snippet = self.db.get_snippet(snippet_id)

                if snippet:
                    self.console.print(f"\n[yellow]Delete '{snippet.title}'?[/yellow]")
                    if Confirm.ask("Are you sure?", default=False):
                        self.db.delete_snippet(snippet_id)
                        self.console.print("[green]✓[/green] Snippet deleted")
                else:
                    self.console.print(f"[red]Snippet {snippet_id} not found[/red]")

            elif choice == MenuOption.EXECUTE.value:
                snippet_id = int(Prompt.ask("Enter snippet ID"))
                snippet = self.db.get_snippet(snippet_id)

                if snippet:
                    self.editor.execute_snippet(snippet)
                else:
                    self.console.print(f"[red]Snippet {snippet_id} not found[/red]")

            elif choice == MenuOption.TAGS.value:
                tags = self.db.get_all_tags()
                self.display.display_tag_cloud(tags)

            elif choice == MenuOption.CATEGORIES.value:
                self.display.display_category_tree(self.db)

            elif choice == MenuOption.FAVORITES.value:
                snippets = self.db.search_snippets(favorite_only=True)
                self.display.display_snippet_list(snippets)

            elif choice == MenuOption.EXPORT.value:
                snippet_id = int(Prompt.ask("Enter snippet ID"))
                snippet = self.db.get_snippet(snippet_id)

                if snippet:
                    filename = Prompt.ask(
                        "Output filename",
                        default=f"{snippet.title.replace(' ', '_')}.{snippet.language}"
                    )

                    with open(filename, 'w') as f:
                        f.write(snippet.code)

                    self.console.print(f"[green]✓[/green] Exported to {filename}")
                else:
                    self.console.print(f"[red]Snippet {snippet_id} not found[/red]")

            # Pause before showing menu again
            if choice != MenuOption.EXIT.value:
                Prompt.ask("\n[dim]Press Enter to continue[/dim]", default="")
                self.console.clear()

# Usage
if __name__ == "__main__":
    app = SnippetManagerApp()
    app.run()
```

## Expected Output

### Snippet List
```
╭─ Code Snippets (42 found) ──────────────────────────────╮
│ ID  ⭐ Title                 Language  Category  Tags   │
│ 12  ⭐ Quick Sort           python    Algorithm sort... │
│ 23     Binary Search        python    Algorithm search │
│ 34     Flask Route Example  python    Web       flask  │
│ 45  ⭐ Docker Compose       yaml      DevOps    docker │
╰──────────────────────────────────────────────────────────╯
```

### Snippet Display with Syntax Highlighting
```
╭─ Quick Sort Implementation (python) ⭐ ─────────────────╮
│ Efficient quicksort algorithm implementation            │
│                                                          │
│ Category: Algorithms | Tags: sorting, recursion         │
│                                                          │
│  1  def quicksort(arr):                                 │
│  2      if len(arr) <= 1:                               │
│  3          return arr                                  │
│  4      pivot = arr[len(arr) // 2]                      │
│  5      ...                                             │
╰────────────────────────────ID: 12 | Updated: 2024-01-15─╯
```

## Bonus Challenges

1. **GitHub Gist Integration**: Import/export from GitHub Gists
2. **Snippet Templates**: Predefined templates for common patterns
3. **Version Control**: Track snippet history
4. **Sharing**: Generate shareable links
5. **Syntax Validation**: Validate code before saving
6. **Auto-tagging**: AI-powered automatic tag suggestions
7. **Snippet Collections**: Group related snippets
8. **Markdown Support**: Include documentation snippets
9. **Multi-file Snippets**: Store related files together
10. **Cloud Sync**: Synchronize across devices

## Resources

- [pygments Documentation](https://pygments.org/)
- [SQLite in Python](https://docs.python.org/3/library/sqlite3.html)
- [Rich Syntax Highlighting](https://rich.readthedocs.io/en/latest/syntax.html)

## Success Criteria

- [ ] Create, read, update, delete snippets
- [ ] Store snippets in SQLite database
- [ ] Display code with syntax highlighting
- [ ] Search by title, description, code content
- [ ] Filter by language, category, tags
- [ ] Tag-based organization
- [ ] Favorite snippets
- [ ] Execute Python/Bash snippets
- [ ] Export snippets to files
- [ ] View usage statistics

## Testing Checklist

- Add snippets in various languages
- Test search functionality
- Verify syntax highlighting for different languages
- Test tag filtering
- Check favorite toggling
- Test snippet execution (safely!)
- Verify export functionality
- Test with empty database
- Add snippets with special characters
- Test database persistence across sessions
