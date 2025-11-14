# Project 1: File System Explorer with Tree View

## Overview
Build a terminal-based file system explorer that displays directory structures in an interactive tree view with rich formatting, file metadata, and navigation capabilities.

## Learning Objectives
- Master Rich's Tree widget for hierarchical data display
- Implement file system traversal and metadata extraction
- Create interactive terminal UIs with keyboard navigation
- Handle file permissions and error cases gracefully
- Apply color coding and icons for different file types
- Implement search and filter functionality

## Difficulty Level
**Intermediate** - Requires understanding of file systems, event handling, and UI state management

## Technical Stack
- **Rich**: Tree, Panel, Table, Syntax, Live, Layout
- **Python stdlib**: os, pathlib, stat, datetime
- **Optional**: watchdog (for file system monitoring)

## Requirements

### Layout Components
1. **Header Panel**: Display current path, total files/folders, disk usage
2. **Tree View**: Main file/folder hierarchy display
3. **Details Panel**: Selected item metadata (size, modified date, permissions)
4. **Status Bar**: Current selection, keyboard shortcuts
5. **Search Box**: Filter files by name/pattern

### Interaction Features
- Navigate directories with keyboard (↑/↓/←/→)
- Expand/collapse folders
- Display hidden files (toggle with 'h')
- Sort by name, size, or date
- Copy file paths to clipboard
- Quick search/filter

### Styling Requirements
- Different colors for files vs directories
- Icons for common file types (.py, .md, .json, etc.)
- Highlight executables and symlinks
- Size formatting (KB, MB, GB)
- Relative timestamps (2 hours ago, yesterday)

## Step-by-Step Implementation

### Step 1: Basic File Tree Display

```python
# file_explorer.py
from pathlib import Path
from rich.tree import Tree
from rich.console import Console
from rich.filesize import decimal
from datetime import datetime
import os

class FileExplorer:
    def __init__(self, root_path: str = "."):
        self.root_path = Path(root_path).resolve()
        self.console = Console()

    def get_file_icon(self, path: Path) -> str:
        """Return an icon based on file type."""
        if path.is_dir():
            return "📁"

        suffix = path.suffix.lower()
        icons = {
            '.py': '🐍',
            '.md': '📝',
            '.json': '⚙️',
            '.yaml': '⚙️',
            '.yml': '⚙️',
            '.txt': '📄',
            '.pdf': '📕',
            '.jpg': '🖼️',
            '.png': '🖼️',
            '.gif': '🖼️',
            '.js': '📜',
            '.html': '🌐',
            '.css': '🎨',
            '.zip': '📦',
            '.tar': '📦',
            '.gz': '📦',
        }
        return icons.get(suffix, '📄')

    def get_file_color(self, path: Path) -> str:
        """Return color based on file type."""
        if path.is_dir():
            return "bold blue"
        elif path.is_symlink():
            return "cyan"
        elif os.access(path, os.X_OK):
            return "bold green"
        elif path.suffix in ['.py', '.js', '.sh']:
            return "green"
        elif path.suffix in ['.md', '.txt', '.doc']:
            return "yellow"
        else:
            return "white"

    def build_tree(self, path: Path, tree: Tree, max_depth: int = 3,
                   current_depth: int = 0, show_hidden: bool = False):
        """Recursively build file tree."""
        if current_depth >= max_depth:
            return

        try:
            paths = sorted(path.iterdir(),
                          key=lambda p: (not p.is_dir(), p.name.lower()))

            for item in paths:
                # Skip hidden files unless requested
                if not show_hidden and item.name.startswith('.'):
                    continue

                # Get metadata
                try:
                    stat = item.stat()
                    size = decimal(stat.st_size) if item.is_file() else ""

                    # Format display name
                    icon = self.get_file_icon(item)
                    color = self.get_file_color(item)
                    name = item.name

                    label = f"{icon} [{color}]{name}[/{color}]"
                    if size:
                        label += f" [dim]({size})[/dim]"

                    # Add to tree
                    if item.is_dir():
                        branch = tree.add(label)
                        self.build_tree(item, branch, max_depth,
                                      current_depth + 1, show_hidden)
                    else:
                        tree.add(label)

                except PermissionError:
                    tree.add(f"🔒 [red]{item.name} (Permission Denied)[/red]")

        except PermissionError:
            tree.add("[red]Permission Denied[/red]")

    def display(self, max_depth: int = 3, show_hidden: bool = False):
        """Display the file tree."""
        tree = Tree(
            f"📂 [bold blue]{self.root_path}[/bold blue]",
            guide_style="dim"
        )

        self.build_tree(self.root_path, tree, max_depth, show_hidden=show_hidden)
        self.console.print(tree)

# Usage
if __name__ == "__main__":
    explorer = FileExplorer(".")
    explorer.display(max_depth=2)
```

### Step 2: Add Metadata Panel

```python
from rich.panel import Panel
from rich.table import Table
from rich.layout import Layout
from rich import box
from datetime import datetime

class FileExplorer:
    # ... previous code ...

    def get_file_metadata(self, path: Path) -> Table:
        """Create metadata table for a file/directory."""
        table = Table(show_header=False, box=box.SIMPLE, padding=(0, 1))
        table.add_column("Property", style="cyan")
        table.add_column("Value", style="white")

        try:
            stat = path.stat()

            # Basic info
            table.add_row("Name", path.name)
            table.add_row("Type", "Directory" if path.is_dir() else "File")
            table.add_row("Path", str(path.absolute()))

            # Size
            if path.is_file():
                size = decimal(stat.st_size)
                table.add_row("Size", size)
            else:
                # Count items in directory
                try:
                    items = list(path.iterdir())
                    table.add_row("Items", str(len(items)))
                except PermissionError:
                    table.add_row("Items", "N/A (Permission Denied)")

            # Timestamps
            modified = datetime.fromtimestamp(stat.st_mtime)
            created = datetime.fromtimestamp(stat.st_ctime)
            table.add_row("Modified", modified.strftime("%Y-%m-%d %H:%M:%S"))
            table.add_row("Created", created.strftime("%Y-%m-%d %H:%M:%S"))

            # Permissions
            mode = oct(stat.st_mode)[-3:]
            table.add_row("Permissions", mode)

            # Owner info (Unix/Linux)
            if hasattr(stat, 'st_uid'):
                table.add_row("Owner UID", str(stat.st_uid))

        except Exception as e:
            table.add_row("Error", str(e))

        return table

    def display_with_details(self, selected_path: Path = None):
        """Display file tree with details panel."""
        layout = Layout()
        layout.split_column(
            Layout(name="header", size=3),
            Layout(name="main"),
            Layout(name="footer", size=3)
        )

        layout["main"].split_row(
            Layout(name="tree", ratio=2),
            Layout(name="details", ratio=1)
        )

        # Header
        header = Panel(
            f"[bold]File Explorer[/bold] - {self.root_path}",
            style="bold white on blue"
        )
        layout["header"].update(header)

        # Tree
        tree = Tree(f"📂 [bold blue]{self.root_path}[/bold blue]")
        self.build_tree(self.root_path, tree, max_depth=2)
        layout["tree"].update(Panel(tree, title="Directory Tree", border_style="blue"))

        # Details
        if selected_path:
            details = self.get_file_metadata(selected_path)
            layout["details"].update(Panel(details, title="Details", border_style="green"))
        else:
            layout["details"].update(Panel("Select a file to view details", title="Details"))

        # Footer
        footer = Panel(
            "[b]↑/↓[/b] Navigate | [b]→[/b] Expand | [b]←[/b] Collapse | [b]h[/b] Toggle Hidden | [b]q[/b] Quit",
            style="white on dark_blue"
        )
        layout["footer"].update(footer)

        self.console.print(layout)
```

### Step 3: Interactive Navigation

```python
from rich.live import Live
from pynput import keyboard
import threading

class InteractiveFileExplorer(FileExplorer):
    def __init__(self, root_path: str = "."):
        super().__init__(root_path)
        self.current_index = 0
        self.file_list = []
        self.show_hidden = False
        self.running = True

    def scan_directory(self, path: Path) -> list:
        """Get list of files/directories."""
        try:
            items = sorted(path.iterdir(),
                         key=lambda p: (not p.is_dir(), p.name.lower()))
            if not self.show_hidden:
                items = [i for i in items if not i.name.startswith('.')]
            return items
        except PermissionError:
            return []

    def build_interactive_tree(self, path: Path, tree: Tree, max_depth: int = 2,
                              current_depth: int = 0):
        """Build tree with current selection highlighted."""
        if current_depth >= max_depth:
            return

        items = self.scan_directory(path)

        for idx, item in enumerate(items):
            try:
                stat = item.stat()
                size = decimal(stat.st_size) if item.is_file() else ""

                icon = self.get_file_icon(item)
                color = self.get_file_color(item)
                name = item.name

                # Highlight if this is current selection
                is_selected = (len(self.file_list) > self.current_index and
                             item == self.file_list[self.current_index])

                if is_selected:
                    label = f"▶ {icon} [reverse {color}]{name}[/reverse {color}]"
                else:
                    label = f"  {icon} [{color}]{name}[/{color}]"

                if size:
                    label += f" [dim]({size})[/dim]"

                if item.is_dir():
                    branch = tree.add(label)
                    if current_depth < max_depth - 1:
                        self.build_interactive_tree(item, branch, max_depth,
                                                   current_depth + 1)
                else:
                    tree.add(label)

            except PermissionError:
                tree.add(f"🔒 [red]{item.name} (Permission Denied)[/red]")

    def render(self) -> Layout:
        """Render the current UI state."""
        layout = Layout()
        layout.split_column(
            Layout(name="header", size=3),
            Layout(name="main"),
            Layout(name="footer", size=3)
        )

        layout["main"].split_row(
            Layout(name="tree", ratio=2),
            Layout(name="details", ratio=1)
        )

        # Update file list
        self.file_list = self.scan_directory(self.root_path)

        # Header
        total_size = sum(f.stat().st_size for f in self.file_list if f.is_file())
        header_text = (
            f"[bold]File Explorer[/bold] - {self.root_path}\n"
            f"Files: {len([f for f in self.file_list if f.is_file()])} | "
            f"Folders: {len([f for f in self.file_list if f.is_dir()])} | "
            f"Total Size: {decimal(total_size)}"
        )
        layout["header"].update(Panel(header_text, style="bold white on blue"))

        # Tree
        tree = Tree(f"📂 [bold blue]{self.root_path}[/bold blue]")
        self.build_interactive_tree(self.root_path, tree, max_depth=3)
        layout["tree"].update(Panel(tree, title="Directory Tree", border_style="blue"))

        # Details
        if self.file_list and 0 <= self.current_index < len(self.file_list):
            selected = self.file_list[self.current_index]
            details = self.get_file_metadata(selected)
            layout["details"].update(Panel(details, title="Details", border_style="green"))
        else:
            layout["details"].update(Panel("No selection", title="Details"))

        # Footer
        footer = Panel(
            "[b]↑/↓[/b] Navigate | [b]Enter[/b] Open | [b]h[/b] Toggle Hidden | "
            "[b]s[/b] Sort | [b]q[/b] Quit",
            style="white on dark_blue"
        )
        layout["footer"].update(footer)

        return layout

    def on_key_press(self, key):
        """Handle keyboard input."""
        try:
            if hasattr(key, 'char'):
                if key.char == 'q':
                    self.running = False
                elif key.char == 'h':
                    self.show_hidden = not self.show_hidden
            elif key == keyboard.Key.up:
                self.current_index = max(0, self.current_index - 1)
            elif key == keyboard.Key.down:
                self.current_index = min(len(self.file_list) - 1,
                                       self.current_index + 1)
            elif key == keyboard.Key.enter:
                if self.file_list and self.current_index < len(self.file_list):
                    selected = self.file_list[self.current_index]
                    if selected.is_dir():
                        self.root_path = selected
                        self.current_index = 0
        except AttributeError:
            pass

    def run(self):
        """Run the interactive explorer."""
        listener = keyboard.Listener(on_press=self.on_key_press)
        listener.start()

        with Live(self.render(), refresh_per_second=4, screen=True) as live:
            while self.running:
                live.update(self.render())
                threading.Event().wait(0.1)

        listener.stop()

# Usage
if __name__ == "__main__":
    explorer = InteractiveFileExplorer(".")
    explorer.run()
```

### Step 4: Search and Filter

```python
from rich.prompt import Prompt
import fnmatch

class FileExplorer:
    # ... previous code ...

    def search_files(self, pattern: str, max_results: int = 50) -> list:
        """Search for files matching pattern."""
        results = []

        def walk_directory(path: Path, depth: int = 0):
            if depth > 5 or len(results) >= max_results:
                return

            try:
                for item in path.iterdir():
                    if fnmatch.fnmatch(item.name.lower(), pattern.lower()):
                        results.append(item)

                    if item.is_dir() and not item.name.startswith('.'):
                        walk_directory(item, depth + 1)
            except PermissionError:
                pass

        walk_directory(self.root_path)
        return results

    def display_search_results(self, pattern: str):
        """Display search results in a table."""
        from rich.table import Table

        results = self.search_files(pattern)

        table = Table(title=f"Search Results for '{pattern}'",
                     show_header=True, header_style="bold magenta")
        table.add_column("Icon", style="white", width=4)
        table.add_column("Name", style="cyan")
        table.add_column("Path", style="dim")
        table.add_column("Size", justify="right", style="green")
        table.add_column("Modified", style="yellow")

        for item in results[:50]:
            try:
                stat = item.stat()
                icon = self.get_file_icon(item)
                size = decimal(stat.st_size) if item.is_file() else "-"
                modified = datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M")

                table.add_row(
                    icon,
                    item.name,
                    str(item.parent),
                    size,
                    modified
                )
            except Exception:
                continue

        self.console.print(table)
        self.console.print(f"\n[dim]Found {len(results)} matches[/dim]")

# Usage for search
if __name__ == "__main__":
    explorer = FileExplorer("/home/user")

    # Search for Python files
    explorer.display_search_results("*.py")

    # Search for config files
    explorer.display_search_results("*.json")
```

## Expected Output

### Basic Tree View
```
📂 /home/user/projects
├── 📁 src
│   ├── 🐍 main.py (2.4 KB)
│   ├── 🐍 utils.py (1.8 KB)
│   └── 📁 models
│       ├── 🐍 user.py (3.2 KB)
│       └── 🐍 database.py (4.1 KB)
├── 📁 tests
│   ├── 🐍 test_main.py (1.2 KB)
│   └── 🐍 test_utils.py (980 B)
├── 📝 README.md (1.5 KB)
└── ⚙️ requirements.txt (234 B)
```

### Interactive View with Details
```
┌─────────────────────────────────────────────────────────────┐
│ File Explorer - /home/user/projects                         │
│ Files: 45 | Folders: 12 | Total Size: 2.4 MB                │
└─────────────────────────────────────────────────────────────┘
┌──────────────────────────────┬──────────────────────────────┐
│ Directory Tree               │ Details                       │
│                              │                               │
│ 📂 /home/user/projects       │ Name        main.py          │
│   ▶ 🐍 main.py (2.4 KB)      │ Type        File             │
│     🐍 utils.py (1.8 KB)     │ Path        /home/.../src    │
│     📁 models                │ Size        2.4 KB           │
│     📁 tests                 │ Modified    2024-01-15 14:30 │
│     📝 README.md             │ Created     2024-01-10 09:15 │
│                              │ Permissions 644              │
└──────────────────────────────┴──────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ ↑/↓ Navigate | Enter Open | h Toggle Hidden | q Quit        │
└─────────────────────────────────────────────────────────────┘
```

## Bonus Challenges

1. **File Operations**: Add ability to copy, move, delete files
2. **Multiple Panes**: Implement dual-pane explorer like Midnight Commander
3. **File Preview**: Show file contents in preview pane for text files
4. **Disk Usage Visualization**: Add progress bars for directory sizes
5. **Bookmarks**: Allow users to bookmark frequently accessed directories
6. **Git Integration**: Show git status indicators for files
7. **Compression Support**: Browse inside ZIP/TAR archives
8. **Network Drives**: Support for remote file systems (SSH/FTP)
9. **Thumbnail Preview**: Show ASCII art thumbnails for images
10. **Watch Mode**: Auto-refresh when files change using watchdog

## Resources

- [Rich Documentation](https://rich.readthedocs.io/)
- [Rich Tree Widget](https://rich.readthedocs.io/en/latest/tree.html)
- [Python pathlib](https://docs.python.org/3/library/pathlib.html)
- [Python os.stat](https://docs.python.org/3/library/os.html#os.stat)
- [pynput for keyboard input](https://pynput.readthedocs.io/)

## Success Criteria

- [ ] Display hierarchical directory structure with proper formatting
- [ ] Show file metadata (size, date, permissions)
- [ ] Implement keyboard navigation (up/down/enter)
- [ ] Toggle hidden files visibility
- [ ] Color code different file types appropriately
- [ ] Handle permission errors gracefully
- [ ] Display file counts and total sizes
- [ ] Implement search/filter functionality
- [ ] Responsive layout that adapts to terminal size
- [ ] Clear and intuitive keyboard shortcuts

## Testing Checklist

- Test with various directory structures (deep, wide, mixed)
- Test with permission-restricted directories
- Test with symbolic links
- Test with hidden files
- Test navigation with empty directories
- Test with directories containing special characters
- Verify performance with large directories (1000+ files)
- Test terminal resizing behavior
- Verify color rendering in different terminal emulators
