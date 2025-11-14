# Project 5: Task/Todo Manager in the Terminal

## Overview
Build a feature-rich terminal-based task manager with categories, priorities, due dates, and progress tracking. Think of it as a combination of Todoist and Trello, but beautifully rendered in your terminal with Rich.

## Learning Objectives
- Implement persistent data storage (JSON/SQLite)
- Create CRUD operations with rich UI feedback
- Design category and tag systems
- Build filtering and sorting mechanisms
- Implement due date tracking and reminders
- Create progress visualizations

## Difficulty Level
**Intermediate** - Requires data modeling, file I/O, and state management

## Technical Stack
- **Rich**: Table, Panel, Tree, Progress, Prompt, Console
- **Python stdlib**: json, datetime, pathlib, enum
- **Optional**: sqlite3 (for database storage), typer (for CLI)

## Requirements

### Core Features
1. **Task Management**: Create, read, update, delete tasks
2. **Categories/Projects**: Organize tasks into projects
3. **Priorities**: Low, medium, high, urgent
4. **Due Dates**: Track deadlines with visual indicators
5. **Status Tracking**: Todo, in progress, completed
6. **Tags**: Flexible tagging system

### Layout Components
1. **Task List View**: Main task table with filtering
2. **Task Details**: Expanded view of single task
3. **Project View**: Tasks grouped by project
4. **Statistics Dashboard**: Overview of tasks by status
5. **Calendar View**: Tasks by due date
6. **Quick Add Form**: Interactive task creation

### Interaction Features
- Add tasks with rich prompts
- Mark tasks as complete with checkboxes
- Filter by status, priority, project, tags
- Sort by due date, priority, creation date
- Archive completed tasks
- Search tasks by keyword

## Step-by-Step Implementation

### Step 1: Task Data Model and Storage

```python
# todo_manager.py
from dataclasses import dataclass, asdict
from datetime import datetime, date
from enum import Enum
from typing import List, Optional
import json
from pathlib import Path
from rich.console import Console

class Priority(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    URGENT = 4

class Status(Enum):
    TODO = "todo"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    ARCHIVED = "archived"

@dataclass
class Task:
    id: int
    title: str
    description: str = ""
    priority: Priority = Priority.MEDIUM
    status: Status = Status.TODO
    project: str = "Inbox"
    tags: List[str] = None
    due_date: Optional[date] = None
    created_at: datetime = None
    completed_at: Optional[datetime] = None

    def __post_init__(self):
        if self.tags is None:
            self.tags = []
        if self.created_at is None:
            self.created_at = datetime.now()

    def to_dict(self) -> dict:
        """Convert task to dictionary for JSON serialization."""
        data = asdict(self)
        data['priority'] = self.priority.value
        data['status'] = self.status.value
        data['created_at'] = self.created_at.isoformat()
        data['due_date'] = self.due_date.isoformat() if self.due_date else None
        data['completed_at'] = self.completed_at.isoformat() if self.completed_at else None
        return data

    @classmethod
    def from_dict(cls, data: dict) -> 'Task':
        """Create task from dictionary."""
        data = data.copy()
        data['priority'] = Priority(data['priority'])
        data['status'] = Status(data['status'])
        data['created_at'] = datetime.fromisoformat(data['created_at'])

        if data.get('due_date'):
            data['due_date'] = date.fromisoformat(data['due_date'])
        if data.get('completed_at'):
            data['completed_at'] = datetime.fromisoformat(data['completed_at'])

        return cls(**data)

class TaskManager:
    def __init__(self, storage_file: str = "tasks.json"):
        self.storage_file = Path(storage_file)
        self.tasks: List[Task] = []
        self.next_id = 1
        self.console = Console()
        self.load_tasks()

    def load_tasks(self):
        """Load tasks from JSON file."""
        if self.storage_file.exists():
            try:
                with open(self.storage_file, 'r') as f:
                    data = json.load(f)
                    self.tasks = [Task.from_dict(task) for task in data['tasks']]
                    self.next_id = data.get('next_id', 1)
            except Exception as e:
                self.console.print(f"[red]Error loading tasks: {e}[/red]")
                self.tasks = []

    def save_tasks(self):
        """Save tasks to JSON file."""
        try:
            data = {
                'tasks': [task.to_dict() for task in self.tasks],
                'next_id': self.next_id
            }
            with open(self.storage_file, 'w') as f:
                json.dump(data, f, indent=2)
        except Exception as e:
            self.console.print(f"[red]Error saving tasks: {e}[/red]")

    def add_task(self, task: Task) -> Task:
        """Add a new task."""
        task.id = self.next_id
        self.next_id += 1
        self.tasks.append(task)
        self.save_tasks()
        return task

    def get_task(self, task_id: int) -> Optional[Task]:
        """Get task by ID."""
        for task in self.tasks:
            if task.id == task_id:
                return task
        return None

    def update_task(self, task_id: int, **kwargs) -> bool:
        """Update task fields."""
        task = self.get_task(task_id)
        if not task:
            return False

        for key, value in kwargs.items():
            if hasattr(task, key):
                setattr(task, key, value)

        self.save_tasks()
        return True

    def delete_task(self, task_id: int) -> bool:
        """Delete a task."""
        task = self.get_task(task_id)
        if task:
            self.tasks.remove(task)
            self.save_tasks()
            return True
        return False

    def complete_task(self, task_id: int) -> bool:
        """Mark task as completed."""
        return self.update_task(
            task_id,
            status=Status.COMPLETED,
            completed_at=datetime.now()
        )

    def get_projects(self) -> List[str]:
        """Get list of all projects."""
        projects = set(task.project for task in self.tasks)
        return sorted(projects)

    def get_tags(self) -> List[str]:
        """Get list of all tags."""
        tags = set()
        for task in self.tasks:
            tags.update(task.tags)
        return sorted(tags)

# Usage
if __name__ == "__main__":
    tm = TaskManager()

    # Add a task
    task = Task(
        id=0,  # Will be set by add_task
        title="Build todo manager",
        description="Create a Rich-based todo app",
        priority=Priority.HIGH,
        project="Side Projects",
        tags=["coding", "python"],
        due_date=date(2024, 12, 31)
    )

    tm.add_task(task)
    print(f"Added task: {task.title}")
```

### Step 2: Task Display and Formatting

```python
from rich.table import Table
from rich.panel import Panel
from rich import box

class TaskManager:
    # ... previous code ...

    def get_priority_icon(self, priority: Priority) -> str:
        """Get icon for priority level."""
        icons = {
            Priority.LOW: "🟢",
            Priority.MEDIUM: "🟡",
            Priority.HIGH: "🟠",
            Priority.URGENT: "🔴"
        }
        return icons.get(priority, "⚪")

    def get_priority_color(self, priority: Priority) -> str:
        """Get color for priority level."""
        colors = {
            Priority.LOW: "green",
            Priority.MEDIUM: "yellow",
            Priority.HIGH: "orange1",
            Priority.URGENT: "red"
        }
        return colors.get(priority, "white")

    def get_status_icon(self, status: Status) -> str:
        """Get icon for status."""
        icons = {
            Status.TODO: "⭕",
            Status.IN_PROGRESS: "🔄",
            Status.COMPLETED: "✅",
            Status.ARCHIVED: "📦"
        }
        return icons.get(status, "❓")

    def format_due_date(self, due_date: Optional[date]) -> tuple:
        """Format due date with color based on urgency."""
        if not due_date:
            return "", "dim"

        today = date.today()
        delta = (due_date - today).days

        if delta < 0:
            return f"{due_date} (overdue!)", "red bold"
        elif delta == 0:
            return f"{due_date} (today!)", "yellow bold"
        elif delta == 1:
            return f"{due_date} (tomorrow)", "yellow"
        elif delta <= 7:
            return f"{due_date} ({delta}d)", "orange1"
        else:
            return str(due_date), "green"

    def display_tasks(
        self,
        filter_status: Optional[Status] = None,
        filter_project: Optional[str] = None,
        filter_priority: Optional[Priority] = None
    ):
        """Display tasks in a formatted table."""
        # Filter tasks
        filtered_tasks = self.tasks

        if filter_status:
            filtered_tasks = [t for t in filtered_tasks if t.status == filter_status]
        if filter_project:
            filtered_tasks = [t for t in filtered_tasks if t.project == filter_project]
        if filter_priority:
            filtered_tasks = [t for t in filtered_tasks if t.priority == filter_priority]

        # Create table
        table = Table(
            title="[bold]Task List[/bold]",
            show_header=True,
            header_style="bold magenta",
            box=box.ROUNDED
        )

        table.add_column("ID", style="cyan", width=5)
        table.add_column("Status", width=4)
        table.add_column("Priority", width=4)
        table.add_column("Task", style="white", overflow="fold")
        table.add_column("Project", style="blue", width=15)
        table.add_column("Due Date", width=20)
        table.add_column("Tags", style="dim", width=15)

        # Sort tasks by priority (desc) and due date
        sorted_tasks = sorted(
            filtered_tasks,
            key=lambda t: (
                t.status.value != "completed",  # Completed last
                -t.priority.value,  # High priority first
                t.due_date if t.due_date else date.max  # Soonest first
            )
        )

        for task in sorted_tasks:
            status_icon = self.get_status_icon(task.status)
            priority_icon = self.get_priority_icon(task.priority)
            due_str, due_color = self.format_due_date(task.due_date)

            # Strike through completed tasks
            task_title = task.title
            if task.status == Status.COMPLETED:
                task_title = f"[dim strikethrough]{task_title}[/dim strikethrough]"

            table.add_row(
                str(task.id),
                status_icon,
                priority_icon,
                task_title,
                task.project,
                f"[{due_color}]{due_str}[/{due_color}]",
                ", ".join(task.tags[:2]) if task.tags else ""
            )

        self.console.print(table)
        self.console.print(f"\n[dim]Total tasks: {len(filtered_tasks)}[/dim]")

    def display_task_details(self, task_id: int):
        """Display detailed view of a single task."""
        task = self.get_task(task_id)
        if not task:
            self.console.print(f"[red]Task {task_id} not found[/red]")
            return

        # Create details table
        details = Table(show_header=False, box=None, padding=(0, 2))
        details.add_column("Field", style="cyan bold", width=15)
        details.add_column("Value", style="white")

        # Add task details
        details.add_row("ID", str(task.id))
        details.add_row("Title", task.title)

        if task.description:
            details.add_row("Description", task.description)

        details.add_row(
            "Priority",
            f"{self.get_priority_icon(task.priority)} {task.priority.name}"
        )
        details.add_row(
            "Status",
            f"{self.get_status_icon(task.status)} {task.status.value.replace('_', ' ').title()}"
        )
        details.add_row("Project", task.project)

        if task.tags:
            details.add_row("Tags", ", ".join(task.tags))

        if task.due_date:
            due_str, due_color = self.format_due_date(task.due_date)
            details.add_row("Due Date", f"[{due_color}]{due_str}[/{due_color}]")

        details.add_row("Created", task.created_at.strftime("%Y-%m-%d %H:%M"))

        if task.completed_at:
            details.add_row("Completed", task.completed_at.strftime("%Y-%m-%d %H:%M"))

        # Display in panel
        panel = Panel(
            details,
            title=f"[bold]Task #{task.id}[/bold]",
            border_style="blue"
        )

        self.console.print(panel)

# Usage
if __name__ == "__main__":
    tm = TaskManager()

    # Display all tasks
    tm.display_tasks()

    # Display tasks by status
    tm.display_tasks(filter_status=Status.TODO)

    # Display task details
    tm.display_task_details(1)
```

### Step 3: Interactive Task Creation

```python
from rich.prompt import Prompt, Confirm
from rich.panel import Panel

class TaskManager:
    # ... previous code ...

    def interactive_add_task(self):
        """Interactively create a new task."""
        self.console.print(Panel(
            "[bold cyan]Create New Task[/bold cyan]",
            border_style="cyan"
        ))

        # Title (required)
        title = Prompt.ask("\n[bold]Task title[/bold]")

        # Description (optional)
        description = Prompt.ask(
            "[bold]Description[/bold] (optional)",
            default=""
        )

        # Priority
        priority_map = {
            "1": Priority.LOW,
            "2": Priority.MEDIUM,
            "3": Priority.HIGH,
            "4": Priority.URGENT
        }

        self.console.print("\n[bold]Priority:[/bold]")
        self.console.print("  1. 🟢 Low")
        self.console.print("  2. 🟡 Medium")
        self.console.print("  3. 🟠 High")
        self.console.print("  4. 🔴 Urgent")

        priority_choice = Prompt.ask(
            "Choose priority",
            choices=["1", "2", "3", "4"],
            default="2"
        )
        priority = priority_map[priority_choice]

        # Project
        projects = self.get_projects()
        if projects:
            self.console.print("\n[bold]Existing projects:[/bold]")
            for i, proj in enumerate(projects, 1):
                self.console.print(f"  {i}. {proj}")

        project = Prompt.ask(
            "\n[bold]Project[/bold]",
            default="Inbox"
        )

        # Tags
        tags_input = Prompt.ask(
            "[bold]Tags[/bold] (comma-separated, optional)",
            default=""
        )
        tags = [tag.strip() for tag in tags_input.split(",") if tag.strip()]

        # Due date
        has_due_date = Confirm.ask("\n[bold]Set due date?[/bold]", default=False)

        due_date = None
        if has_due_date:
            while True:
                date_str = Prompt.ask(
                    "Due date (YYYY-MM-DD)",
                    default=str(date.today())
                )
                try:
                    due_date = date.fromisoformat(date_str)
                    break
                except ValueError:
                    self.console.print("[red]Invalid date format. Use YYYY-MM-DD[/red]")

        # Create task
        task = Task(
            id=0,  # Will be set by add_task
            title=title,
            description=description,
            priority=priority,
            project=project,
            tags=tags,
            due_date=due_date
        )

        # Confirm and add
        self.console.print("\n[bold]Task Preview:[/bold]")
        self.console.print(f"  Title: {task.title}")
        self.console.print(f"  Priority: {self.get_priority_icon(priority)} {priority.name}")
        self.console.print(f"  Project: {task.project}")
        if task.tags:
            self.console.print(f"  Tags: {', '.join(task.tags)}")
        if task.due_date:
            self.console.print(f"  Due: {task.due_date}")

        if Confirm.ask("\n[bold]Add this task?[/bold]", default=True):
            added_task = self.add_task(task)
            self.console.print(f"\n[green]✓[/green] Task #{added_task.id} created successfully!")
        else:
            self.console.print("\n[dim]Task creation cancelled[/dim]")

    def interactive_update_task(self, task_id: int):
        """Interactively update a task."""
        task = self.get_task(task_id)
        if not task:
            self.console.print(f"[red]Task {task_id} not found[/red]")
            return

        self.console.print(f"\n[bold]Updating Task #{task_id}: {task.title}[/bold]\n")

        # What to update
        self.console.print("What would you like to update?")
        self.console.print("  1. Title")
        self.console.print("  2. Description")
        self.console.print("  3. Priority")
        self.console.print("  4. Status")
        self.console.print("  5. Project")
        self.console.print("  6. Tags")
        self.console.print("  7. Due Date")

        choice = Prompt.ask(
            "Choose option",
            choices=["1", "2", "3", "4", "5", "6", "7"]
        )

        if choice == "1":
            title = Prompt.ask("New title", default=task.title)
            self.update_task(task_id, title=title)

        elif choice == "2":
            description = Prompt.ask("New description", default=task.description)
            self.update_task(task_id, description=description)

        elif choice == "3":
            priority_map = {
                "1": Priority.LOW,
                "2": Priority.MEDIUM,
                "3": Priority.HIGH,
                "4": Priority.URGENT
            }
            priority_choice = Prompt.ask("Priority (1-4)", choices=["1", "2", "3", "4"])
            self.update_task(task_id, priority=priority_map[priority_choice])

        elif choice == "4":
            status_map = {
                "1": Status.TODO,
                "2": Status.IN_PROGRESS,
                "3": Status.COMPLETED
            }
            self.console.print("  1. Todo")
            self.console.print("  2. In Progress")
            self.console.print("  3. Completed")
            status_choice = Prompt.ask("Status", choices=["1", "2", "3"])
            new_status = status_map[status_choice]

            if new_status == Status.COMPLETED:
                self.complete_task(task_id)
            else:
                self.update_task(task_id, status=new_status)

        elif choice == "5":
            project = Prompt.ask("New project", default=task.project)
            self.update_task(task_id, project=project)

        elif choice == "6":
            tags_input = Prompt.ask(
                "Tags (comma-separated)",
                default=", ".join(task.tags)
            )
            tags = [tag.strip() for tag in tags_input.split(",") if tag.strip()]
            self.update_task(task_id, tags=tags)

        elif choice == "7":
            date_str = Prompt.ask("Due date (YYYY-MM-DD)", default=str(task.due_date or ""))
            if date_str:
                try:
                    due_date = date.fromisoformat(date_str)
                    self.update_task(task_id, due_date=due_date)
                except ValueError:
                    self.console.print("[red]Invalid date format[/red]")
            else:
                self.update_task(task_id, due_date=None)

        self.console.print(f"\n[green]✓[/green] Task updated successfully!")

# Usage
if __name__ == "__main__":
    tm = TaskManager()

    # Interactive task creation
    tm.interactive_add_task()

    # Interactive task update
    tm.interactive_update_task(1)
```

### Step 4: Project and Statistics Views

```python
from rich.tree import Tree
from rich.progress import Progress, BarColumn, TextColumn
from collections import Counter

class TaskManager:
    # ... previous code ...

    def display_by_project(self):
        """Display tasks grouped by project."""
        tree = Tree(
            "[bold blue]Projects[/bold blue]",
            guide_style="dim"
        )

        # Group tasks by project
        projects = {}
        for task in self.tasks:
            if task.status != Status.ARCHIVED:
                if task.project not in projects:
                    projects[task.project] = []
                projects[task.project].append(task)

        # Add to tree
        for project_name in sorted(projects.keys()):
            tasks = projects[project_name]
            completed = len([t for t in tasks if t.status == Status.COMPLETED])
            total = len(tasks)

            project_branch = tree.add(
                f"[cyan]{project_name}[/cyan] [dim]({completed}/{total})[/dim]"
            )

            for task in sorted(tasks, key=lambda t: t.priority.value, reverse=True):
                status_icon = self.get_status_icon(task.status)
                priority_icon = self.get_priority_icon(task.priority)

                task_text = f"{status_icon} {priority_icon} {task.title}"

                if task.due_date:
                    due_str, due_color = self.format_due_date(task.due_date)
                    task_text += f" [{due_color}]({due_str})[/{due_color}]"

                project_branch.add(task_text)

        self.console.print(tree)

    def display_statistics(self):
        """Display task statistics dashboard."""
        # Count tasks by status
        status_counts = Counter(task.status for task in self.tasks)

        # Count tasks by priority
        priority_counts = Counter(
            task.priority for task in self.tasks
            if task.status != Status.COMPLETED
        )

        # Overdue tasks
        today = date.today()
        overdue = [
            task for task in self.tasks
            if task.due_date and task.due_date < today
            and task.status != Status.COMPLETED
        ]

        # Create statistics panel
        stats_text = f"""[bold]Task Statistics[/bold]

[cyan]By Status:[/cyan]
  Todo:        {status_counts[Status.TODO]}
  In Progress: {status_counts[Status.IN_PROGRESS]}
  Completed:   {status_counts[Status.COMPLETED]}
  Archived:    {status_counts[Status.ARCHIVED]}

[cyan]Active Tasks by Priority:[/cyan]
  🔴 Urgent:  {priority_counts[Priority.URGENT]}
  🟠 High:    {priority_counts[Priority.HIGH]}
  🟡 Medium:  {priority_counts[Priority.MEDIUM]}
  🟢 Low:     {priority_counts[Priority.LOW]}

[cyan]Other:[/cyan]
  Total Tasks:    {len(self.tasks)}
  Overdue Tasks:  [red]{len(overdue)}[/red]
  Projects:       {len(self.get_projects())}
  Tags:           {len(self.get_tags())}
"""

        panel = Panel(stats_text, border_style="blue")
        self.console.print(panel)

        # Show completion progress
        if status_counts[Status.TODO] + status_counts[Status.IN_PROGRESS] > 0:
            self.console.print("\n[bold]Overall Progress[/bold]\n")

            total_active = (
                status_counts[Status.TODO] +
                status_counts[Status.IN_PROGRESS] +
                status_counts[Status.COMPLETED]
            )

            with Progress(
                TextColumn("[progress.description]{task.description}"),
                BarColumn(bar_width=40),
                TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
                console=self.console
            ) as progress:
                task = progress.add_task(
                    "Completion",
                    total=total_active,
                    completed=status_counts[Status.COMPLETED]
                )

    def display_upcoming(self, days: int = 7):
        """Display tasks due in the next N days."""
        today = date.today()
        upcoming_date = today + timedelta(days=days)

        upcoming_tasks = [
            task for task in self.tasks
            if task.due_date
            and today <= task.due_date <= upcoming_date
            and task.status != Status.COMPLETED
        ]

        if not upcoming_tasks:
            self.console.print(f"[dim]No tasks due in the next {days} days[/dim]")
            return

        # Sort by due date
        upcoming_tasks.sort(key=lambda t: t.due_date)

        table = Table(
            title=f"[bold]Tasks Due in Next {days} Days[/bold]",
            show_header=True,
            header_style="bold yellow",
            box=box.ROUNDED
        )

        table.add_column("Due", style="yellow", width=12)
        table.add_column("Priority", width=4)
        table.add_column("Task", style="white")
        table.add_column("Project", style="blue", width=15)

        for task in upcoming_tasks:
            days_until = (task.due_date - today).days
            if days_until == 0:
                due_display = "Today!"
                due_style = "red bold"
            elif days_until == 1:
                due_display = "Tomorrow"
                due_style = "yellow bold"
            else:
                due_display = f"in {days_until}d"
                due_style = "yellow"

            priority_icon = self.get_priority_icon(task.priority)

            table.add_row(
                f"[{due_style}]{due_display}[/{due_style}]",
                priority_icon,
                task.title,
                task.project
            )

        self.console.print(table)

# Usage
from datetime import timedelta

if __name__ == "__main__":
    tm = TaskManager()

    # Display by project
    tm.display_by_project()

    # Display statistics
    tm.display_statistics()

    # Display upcoming tasks
    tm.display_upcoming(days=7)
```

### Step 5: Main Interactive Interface

```python
from enum import Enum

class MenuOption(Enum):
    LIST = "1"
    ADD = "2"
    UPDATE = "3"
    COMPLETE = "4"
    DELETE = "5"
    DETAILS = "6"
    PROJECTS = "7"
    STATS = "8"
    UPCOMING = "9"
    EXIT = "0"

class InteractiveTaskManager(TaskManager):
    def __init__(self, storage_file: str = "tasks.json"):
        super().__init__(storage_file)

    def display_menu(self):
        """Display main menu."""
        menu_text = """
[bold cyan]Task Manager Menu[/bold cyan]

[1] 📋 List all tasks
[2] ➕ Add new task
[3] ✏️  Update task
[4] ✅ Complete task
[5] 🗑️  Delete task
[6] ℹ️  View task details
[7] 📁 View by project
[8] 📊 Statistics
[9] 📅 Upcoming tasks
[0] 🚪 Exit
"""
        self.console.print(Panel(menu_text, border_style="blue"))

    def run(self):
        """Run interactive menu loop."""
        self.console.clear()
        self.console.print("[bold magenta]Welcome to Task Manager[/bold magenta]\n")

        while True:
            self.display_menu()

            choice = Prompt.ask(
                "Select an option",
                choices=["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"],
                default="1"
            )

            self.console.print()

            if choice == MenuOption.LIST.value:
                # Filter options
                self.console.print("[bold]Filter by:[/bold]")
                self.console.print("  1. All tasks")
                self.console.print("  2. Todo only")
                self.console.print("  3. In Progress only")
                self.console.print("  4. Completed only")

                filter_choice = Prompt.ask(
                    "Choose filter",
                    choices=["1", "2", "3", "4"],
                    default="1"
                )

                filter_map = {
                    "1": None,
                    "2": Status.TODO,
                    "3": Status.IN_PROGRESS,
                    "4": Status.COMPLETED
                }

                self.display_tasks(filter_status=filter_map[filter_choice])

            elif choice == MenuOption.ADD.value:
                self.interactive_add_task()

            elif choice == MenuOption.UPDATE.value:
                task_id = int(Prompt.ask("Enter task ID"))
                self.interactive_update_task(task_id)

            elif choice == MenuOption.COMPLETE.value:
                task_id = int(Prompt.ask("Enter task ID to complete"))
                if self.complete_task(task_id):
                    self.console.print(f"[green]✓[/green] Task #{task_id} marked as complete!")
                else:
                    self.console.print(f"[red]✗[/red] Task {task_id} not found")

            elif choice == MenuOption.DELETE.value:
                task_id = int(Prompt.ask("Enter task ID to delete"))
                task = self.get_task(task_id)

                if task:
                    if Confirm.ask(f"[yellow]Delete '{task.title}'?[/yellow]"):
                        self.delete_task(task_id)
                        self.console.print(f"[green]✓[/green] Task deleted")
                    else:
                        self.console.print("[dim]Cancelled[/dim]")
                else:
                    self.console.print(f"[red]Task {task_id} not found[/red]")

            elif choice == MenuOption.DETAILS.value:
                task_id = int(Prompt.ask("Enter task ID"))
                self.display_task_details(task_id)

            elif choice == MenuOption.PROJECTS.value:
                self.display_by_project()

            elif choice == MenuOption.STATS.value:
                self.display_statistics()

            elif choice == MenuOption.UPCOMING.value:
                days = int(Prompt.ask("Show tasks due in next N days", default="7"))
                self.display_upcoming(days)

            elif choice == MenuOption.EXIT.value:
                self.console.print("[bold green]Goodbye![/bold green]")
                break

            # Pause
            if choice != MenuOption.EXIT.value:
                Prompt.ask("\n[dim]Press Enter to continue[/dim]", default="")
                self.console.clear()

# Usage
if __name__ == "__main__":
    tm = InteractiveTaskManager()
    tm.run()
```

## Expected Output

### Task List
```
╭─ Task List ───────────────────────────────────────────────╮
│ ID  Status Priority  Task              Project  Due Date  │
│ 1   ⭕   🔴      Build website      Work     2024-01-20   │
│ 2   🔄   🟠      Write docs         Work     2024-01-25   │
│ 3   ⭕   🟡      Buy groceries      Personal today!       │
│ 4   ✅   🟢      Update resume      Career   completed    │
╰───────────────────────────────────────────────────────────╯
```

### Statistics Dashboard
```
╭─ Task Statistics ─────────────────╮
│ By Status:                        │
│   Todo:        5                  │
│   In Progress: 3                  │
│   Completed:   12                 │
│                                   │
│ Active Tasks by Priority:         │
│   🔴 Urgent:  2                   │
│   🟠 High:    3                   │
│   🟡 Medium:  2                   │
│   🟢 Low:     1                   │
│                                   │
│ Overall Progress                  │
│ Completion ████████████░░ 60%     │
╰───────────────────────────────────╯
```

## Bonus Challenges

1. **Recurring Tasks**: Support for daily/weekly/monthly tasks
2. **Subtasks**: Break tasks into smaller subtasks
3. **Time Tracking**: Track time spent on tasks
4. **Reports**: Generate weekly/monthly productivity reports
5. **Search**: Full-text search across all tasks
6. **Export**: Export to CSV, JSON, Markdown
7. **Sync**: Cloud sync with REST API
8. **Notifications**: Desktop notifications for due tasks
9. **Calendar Integration**: Export to iCal format
10. **Themes**: Customizable color schemes

## Resources

- [Rich Tables](https://rich.readthedocs.io/en/latest/tables.html)
- [Rich Prompts](https://rich.readthedocs.io/en/latest/prompt.html)
- [Python datetime](https://docs.python.org/3/library/datetime.html)
- [JSON in Python](https://docs.python.org/3/library/json.html)

## Success Criteria

- [ ] Create, read, update, delete tasks
- [ ] Persist tasks to JSON file
- [ ] Display tasks in formatted table
- [ ] Filter tasks by status, priority, project
- [ ] Track task priorities and due dates
- [ ] Visual indicators for overdue tasks
- [ ] Group tasks by projects
- [ ] Display statistics dashboard
- [ ] Interactive task creation
- [ ] Clean and intuitive UI

## Testing Checklist

- Create tasks with various priorities
- Test due date tracking (past, today, future)
- Verify task persistence across restarts
- Test filtering and sorting
- Create tasks in multiple projects
- Test task completion workflow
- Verify statistics calculations
- Test with empty task list
- Test with large number of tasks (100+)
- Verify data integrity after operations
