# Project 3: Package Manager TUI (Terminal User Interface)

## Overview
Build a terminal-based package manager interface similar to npm, pip, or apt, featuring package search, installation tracking, dependency visualization, and version management with Rich's beautiful formatting.

## Learning Objectives
- Create multi-step workflows in terminal UIs
- Implement progress tracking for long-running operations
- Design table-based interfaces for data browsing
- Handle subprocess execution with real-time output
- Manage application state across different views
- Parse and display dependency trees

## Difficulty Level
**Advanced** - Requires understanding of package management, subprocess handling, and complex state management

## Technical Stack
- **Rich**: Table, Progress, Tree, Panel, Prompt, Live, Console
- **Python stdlib**: subprocess, json, urllib, packaging
- **pip API**: pip._internal (for pip wrapper)
- **Optional**: requests (for PyPI API), toml (for config files)

## Requirements

### Core Features
1. **Package Search**: Search PyPI/npm with pagination
2. **Install Tracker**: Real-time installation progress
3. **Package List**: View installed packages with versions
4. **Dependency Tree**: Visualize package dependencies
5. **Update Check**: Find outdated packages
6. **Uninstall Manager**: Safe package removal

### Layout Components
1. **Main Menu**: Navigation to different operations
2. **Search Results Table**: Paginated package listings
3. **Installation Progress**: Multi-stage progress bars
4. **Package Details**: Version, dependencies, description
5. **Dependency Tree**: Hierarchical dependency view
6. **Status Messages**: Success/error notifications

### Interaction Features
- Navigate search results with arrow keys
- Select packages for installation/removal
- Bulk operations (install/update multiple packages)
- Filter and sort package lists
- View package details before installing
- Confirmation prompts for destructive operations

## Step-by-Step Implementation

### Step 1: Basic Package Manager Framework

```python
# package_manager.py
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.prompt import Prompt, Confirm
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn
import subprocess
import json
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class Package:
    name: str
    version: str
    summary: str = ""
    author: str = ""
    home_page: str = ""
    requires: List[str] = None

    def __post_init__(self):
        if self.requires is None:
            self.requires = []

class PackageManager:
    def __init__(self):
        self.console = Console()

    def run_command(self, cmd: List[str]) -> tuple:
        """Execute command and return output."""
        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                check=True
            )
            return True, result.stdout
        except subprocess.CalledProcessError as e:
            return False, e.stderr

    def get_installed_packages(self) -> List[Package]:
        """Get list of installed packages."""
        success, output = self.run_command(["pip", "list", "--format=json"])

        if not success:
            return []

        packages = []
        for pkg_data in json.loads(output):
            packages.append(Package(
                name=pkg_data['name'],
                version=pkg_data['version']
            ))

        return packages

    def display_installed_packages(self):
        """Display table of installed packages."""
        packages = self.get_installed_packages()

        table = Table(title="Installed Packages", show_header=True, header_style="bold magenta")
        table.add_column("Package", style="cyan", width=30)
        table.add_column("Version", style="green", width=15)
        table.add_column("Location", style="yellow")

        for pkg in sorted(packages, key=lambda p: p.name.lower()):
            # Get package location
            success, location = self.run_command(["pip", "show", pkg.name])
            loc = "N/A"
            if success:
                for line in location.split('\n'):
                    if line.startswith('Location:'):
                        loc = line.split(':', 1)[1].strip()
                        break

            table.add_row(pkg.name, pkg.version, loc)

        self.console.print(table)
        self.console.print(f"\n[dim]Total packages: {len(packages)}[/dim]")

# Usage
if __name__ == "__main__":
    pm = PackageManager()
    pm.display_installed_packages()
```

### Step 2: Package Search with PyPI API

```python
import urllib.request
import urllib.parse
from typing import Optional

class PackageManager:
    # ... previous code ...

    def search_pypi(self, query: str, limit: int = 20) -> List[Dict]:
        """Search PyPI for packages."""
        # PyPI's JSON API endpoint
        url = f"https://pypi.org/search/?q={urllib.parse.quote(query)}"

        # Alternative: Use PyPI's JSON API for specific package
        # This is a simplified version - actual implementation would use PyPI's API
        packages = []

        try:
            # For demonstration, we'll show installed packages matching query
            installed = self.get_installed_packages()
            matches = [
                pkg for pkg in installed
                if query.lower() in pkg.name.lower()
            ]

            for pkg in matches[:limit]:
                # Get detailed info
                success, info = self.run_command(["pip", "show", pkg.name])
                if success:
                    pkg_info = self.parse_pip_show(info)
                    packages.append(pkg_info)

        except Exception as e:
            self.console.print(f"[red]Search failed: {e}[/red]")

        return packages

    def parse_pip_show(self, output: str) -> Dict:
        """Parse output from 'pip show' command."""
        info = {}
        for line in output.split('\n'):
            if ':' in line:
                key, value = line.split(':', 1)
                info[key.strip()] = value.strip()

        return {
            'name': info.get('Name', ''),
            'version': info.get('Version', ''),
            'summary': info.get('Summary', ''),
            'author': info.get('Author', ''),
            'home_page': info.get('Home-page', ''),
            'requires': info.get('Requires', '').split(', ') if info.get('Requires') else [],
            'location': info.get('Location', '')
        }

    def display_search_results(self, query: str):
        """Display search results in a table."""
        with self.console.status(f"[bold green]Searching for '{query}'..."):
            results = self.search_pypi(query)

        if not results:
            self.console.print(f"[yellow]No packages found matching '{query}'[/yellow]")
            return

        table = Table(title=f"Search Results for '{query}'", show_header=True)
        table.add_column("#", style="dim", width=4)
        table.add_column("Package", style="cyan", width=25)
        table.add_column("Version", style="green", width=12)
        table.add_column("Summary", style="white", overflow="fold")

        for idx, pkg in enumerate(results, 1):
            table.add_row(
                str(idx),
                pkg['name'],
                pkg['version'],
                pkg['summary'][:100] + "..." if len(pkg['summary']) > 100 else pkg['summary']
            )

        self.console.print(table)

    def display_package_details(self, package_name: str):
        """Display detailed information about a package."""
        success, output = self.run_command(["pip", "show", package_name])

        if not success:
            self.console.print(f"[red]Package '{package_name}' not found[/red]")
            return

        info = self.parse_pip_show(output)

        # Create details panel
        details_table = Table(show_header=False, box=None, padding=(0, 2))
        details_table.add_column("Field", style="cyan bold")
        details_table.add_column("Value", style="white")

        details_table.add_row("Name", info['name'])
        details_table.add_row("Version", info['version'])
        details_table.add_row("Summary", info['summary'])
        details_table.add_row("Author", info['author'])
        details_table.add_row("Home Page", info['home_page'])
        details_table.add_row("Location", info['location'])

        if info['requires']:
            details_table.add_row("Requires", ", ".join(info['requires']))

        panel = Panel(
            details_table,
            title=f"[bold]{package_name}[/bold]",
            border_style="blue"
        )

        self.console.print(panel)

# Usage
if __name__ == "__main__":
    pm = PackageManager()

    # Search for packages
    pm.display_search_results("requests")

    # Show package details
    pm.display_package_details("pip")
```

### Step 3: Installation Progress Tracking

```python
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn, TimeRemainingColumn
import threading
import queue

class PackageManager:
    # ... previous code ...

    def install_package(self, package_name: str, upgrade: bool = False):
        """Install a package with progress tracking."""
        cmd = ["pip", "install"]
        if upgrade:
            cmd.append("--upgrade")
        cmd.append(package_name)

        self.console.print(f"\n[bold cyan]Installing {package_name}...[/bold cyan]\n")

        with Progress(
            SpinnerColumn(),
            TextColumn("[progress.description]{task.description}"),
            BarColumn(),
            TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
            TimeRemainingColumn(),
            console=self.console
        ) as progress:

            # Create tasks for different stages
            download_task = progress.add_task("[cyan]Downloading...", total=100)
            install_task = progress.add_task("[green]Installing...", total=100)

            # Run pip install
            try:
                process = subprocess.Popen(
                    cmd,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.STDOUT,
                    text=True,
                    bufsize=1
                )

                # Simulate progress based on output
                for line in process.stdout:
                    if "Downloading" in line or "Collecting" in line:
                        progress.update(download_task, advance=10)
                    elif "Installing" in line:
                        progress.update(download_task, completed=100)
                        progress.update(install_task, advance=20)
                    elif "Successfully installed" in line:
                        progress.update(install_task, completed=100)

                    # Print output
                    self.console.print(f"[dim]{line.strip()}[/dim]")

                process.wait()

                # Ensure tasks complete
                progress.update(download_task, completed=100)
                progress.update(install_task, completed=100)

                if process.returncode == 0:
                    self.console.print(f"\n[bold green]✓[/bold green] Successfully installed {package_name}")
                else:
                    self.console.print(f"\n[bold red]✗[/bold red] Failed to install {package_name}")

            except Exception as e:
                self.console.print(f"\n[bold red]Error:[/bold red] {e}")

    def install_multiple_packages(self, packages: List[str]):
        """Install multiple packages with overall progress."""
        self.console.print(f"\n[bold]Installing {len(packages)} packages...[/bold]\n")

        with Progress(
            SpinnerColumn(),
            TextColumn("[progress.description]{task.description}"),
            BarColumn(),
            TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
            console=self.console
        ) as progress:

            overall = progress.add_task("[cyan]Overall Progress", total=len(packages))

            for package in packages:
                task = progress.add_task(f"[green]Installing {package}", total=100)

                cmd = ["pip", "install", package, "-q"]  # Quiet mode
                result = subprocess.run(cmd, capture_output=True, text=True)

                progress.update(task, completed=100)
                progress.update(overall, advance=1)

                if result.returncode == 0:
                    self.console.print(f"  [green]✓[/green] {package}")
                else:
                    self.console.print(f"  [red]✗[/red] {package} - {result.stderr.strip()[:50]}")

        self.console.print("\n[bold green]Installation complete![/bold green]")

    def uninstall_package(self, package_name: str, auto_confirm: bool = False):
        """Uninstall a package with confirmation."""
        if not auto_confirm:
            confirm = Confirm.ask(
                f"[yellow]Are you sure you want to uninstall {package_name}?[/yellow]"
            )
            if not confirm:
                self.console.print("[dim]Uninstall cancelled[/dim]")
                return

        with self.console.status(f"[bold red]Uninstalling {package_name}..."):
            success, output = self.run_command(["pip", "uninstall", package_name, "-y"])

        if success:
            self.console.print(f"[bold green]✓[/bold green] Successfully uninstalled {package_name}")
        else:
            self.console.print(f"[bold red]✗[/bold red] Failed to uninstall {package_name}")
            self.console.print(f"[red]{output}[/red]")

# Usage
if __name__ == "__main__":
    pm = PackageManager()

    # Install single package
    pm.install_package("requests")

    # Install multiple packages
    pm.install_multiple_packages(["requests", "rich", "click"])

    # Uninstall package
    pm.uninstall_package("old-package")
```

### Step 4: Dependency Tree Visualization

```python
from rich.tree import Tree
from typing import Set

class PackageManager:
    # ... previous code ...

    def get_package_dependencies(self, package_name: str) -> List[str]:
        """Get direct dependencies of a package."""
        success, output = self.run_command(["pip", "show", package_name])

        if not success:
            return []

        info = self.parse_pip_show(output)
        return info.get('requires', [])

    def build_dependency_tree(
        self,
        package_name: str,
        tree: Tree,
        visited: Set[str] = None,
        max_depth: int = 3,
        current_depth: int = 0
    ):
        """Recursively build dependency tree."""
        if visited is None:
            visited = set()

        if current_depth >= max_depth or package_name in visited:
            return

        visited.add(package_name)

        dependencies = self.get_package_dependencies(package_name)

        for dep in dependencies:
            # Get version info
            success, output = self.run_command(["pip", "show", dep])

            if success:
                info = self.parse_pip_show(output)
                version = info.get('version', 'unknown')

                # Create branch
                branch = tree.add(f"[cyan]{dep}[/cyan] [dim]({version})[/dim]")

                # Recursively add dependencies
                self.build_dependency_tree(
                    dep,
                    branch,
                    visited,
                    max_depth,
                    current_depth + 1
                )
            else:
                tree.add(f"[red]{dep}[/red] [dim](not installed)[/dim]")

    def display_dependency_tree(self, package_name: str, max_depth: int = 3):
        """Display dependency tree for a package."""
        # Get package version
        success, output = self.run_command(["pip", "show", package_name])

        if not success:
            self.console.print(f"[red]Package '{package_name}' not found[/red]")
            return

        info = self.parse_pip_show(output)
        version = info.get('version', 'unknown')

        # Create tree
        tree = Tree(
            f"[bold blue]{package_name}[/bold blue] [dim]({version})[/dim]",
            guide_style="dim"
        )

        self.build_dependency_tree(package_name, tree, max_depth=max_depth)

        self.console.print(Panel(
            tree,
            title="[bold]Dependency Tree[/bold]",
            border_style="blue"
        ))

    def get_outdated_packages(self) -> List[Dict]:
        """Get list of outdated packages."""
        success, output = self.run_command(["pip", "list", "--outdated", "--format=json"])

        if not success:
            return []

        return json.loads(output)

    def display_outdated_packages(self):
        """Display table of outdated packages."""
        with self.console.status("[bold green]Checking for outdated packages..."):
            outdated = self.get_outdated_packages()

        if not outdated:
            self.console.print("[green]All packages are up to date![/green]")
            return

        table = Table(title="Outdated Packages", show_header=True, header_style="bold red")
        table.add_column("Package", style="cyan", width=25)
        table.add_column("Current", style="yellow", width=15)
        table.add_column("Latest", style="green", width=15)
        table.add_column("Type", style="magenta", width=10)

        for pkg in outdated:
            table.add_row(
                pkg['name'],
                pkg['version'],
                pkg['latest_version'],
                pkg.get('latest_filetype', 'wheel')
            )

        self.console.print(table)
        self.console.print(f"\n[dim]Total outdated: {len(outdated)}[/dim]")

        # Offer to update all
        if Confirm.ask("\n[yellow]Update all packages?[/yellow]"):
            package_names = [pkg['name'] for pkg in outdated]
            self.install_multiple_packages(package_names)

# Usage
if __name__ == "__main__":
    pm = PackageManager()

    # Show dependency tree
    pm.display_dependency_tree("flask", max_depth=2)

    # Check for outdated packages
    pm.display_outdated_packages()
```

### Step 5: Interactive Menu System

```python
from rich.prompt import Prompt
from rich.panel import Panel
from enum import Enum

class MenuOption(Enum):
    SEARCH = "1"
    INSTALL = "2"
    UNINSTALL = "3"
    LIST = "4"
    OUTDATED = "5"
    DEPENDENCIES = "6"
    DETAILS = "7"
    EXIT = "8"

class InteractivePackageManager(PackageManager):
    def __init__(self):
        super().__init__()

    def display_menu(self):
        """Display main menu."""
        menu_text = """
[bold cyan]Package Manager Menu[/bold cyan]

[1] 🔍 Search packages
[2] ⬇️  Install package
[3] 🗑️  Uninstall package
[4] 📦 List installed packages
[5] 🔄 Check for updates
[6] 🌳 Show dependency tree
[7] ℹ️  Package details
[8] 🚪 Exit

"""
        self.console.print(Panel(menu_text, border_style="blue"))

    def run(self):
        """Run interactive menu loop."""
        self.console.clear()
        self.console.print("[bold magenta]Welcome to Package Manager TUI[/bold magenta]\n")

        while True:
            self.display_menu()

            choice = Prompt.ask(
                "Select an option",
                choices=["1", "2", "3", "4", "5", "6", "7", "8"],
                default="8"
            )

            self.console.print()

            if choice == MenuOption.SEARCH.value:
                query = Prompt.ask("Enter search query")
                self.display_search_results(query)

            elif choice == MenuOption.INSTALL.value:
                package = Prompt.ask("Enter package name")
                upgrade = Confirm.ask("Upgrade if already installed?", default=False)
                self.install_package(package, upgrade=upgrade)

            elif choice == MenuOption.UNINSTALL.value:
                package = Prompt.ask("Enter package name")
                self.uninstall_package(package)

            elif choice == MenuOption.LIST.value:
                self.display_installed_packages()

            elif choice == MenuOption.OUTDATED.value:
                self.display_outdated_packages()

            elif choice == MenuOption.DEPENDENCIES.value:
                package = Prompt.ask("Enter package name")
                depth = int(Prompt.ask("Max depth", default="2"))
                self.display_dependency_tree(package, max_depth=depth)

            elif choice == MenuOption.DETAILS.value:
                package = Prompt.ask("Enter package name")
                self.display_package_details(package)

            elif choice == MenuOption.EXIT.value:
                self.console.print("[bold green]Goodbye![/bold green]")
                break

            # Pause before showing menu again
            if choice != MenuOption.EXIT.value:
                Prompt.ask("\n[dim]Press Enter to continue[/dim]", default="")
                self.console.clear()

# Usage
if __name__ == "__main__":
    pm = InteractivePackageManager()
    pm.run()
```

## Expected Output

### Main Menu
```
╭────────────────────────────────────╮
│ Package Manager Menu               │
│                                    │
│ [1] 🔍 Search packages             │
│ [2] ⬇️  Install package            │
│ [3] 🗑️  Uninstall package          │
│ [4] 📦 List installed packages     │
│ [5] 🔄 Check for updates           │
│ [6] 🌳 Show dependency tree        │
│ [7] ℹ️  Package details            │
│ [8] 🚪 Exit                        │
╰────────────────────────────────────╯
```

### Installation Progress
```
⠋ Overall Progress ━━━━━━━━━━━━━━━━━━━━ 40% 0:00:15
⠙ Installing requests ━━━━━━━━━━━━━━━━━━ 75%
  ✓ numpy
  ✓ pandas
  ⠋ requests
```

### Dependency Tree
```
╭─ Dependency Tree ────────────────────╮
│ flask (2.0.1)                        │
│ ├── click (8.0.1)                    │
│ ├── itsdangerous (2.0.1)             │
│ ├── jinja2 (3.0.1)                   │
│ │   └── markupsafe (2.0.1)           │
│ └── werkzeug (2.0.1)                 │
╰──────────────────────────────────────╯
```

## Bonus Challenges

1. **Virtual Environment Manager**: Create/manage virtual environments
2. **Requirements File Manager**: Import/export requirements.txt
3. **Package Comparison**: Compare versions across environments
4. **Security Audit**: Check for vulnerabilities using safety
5. **License Checker**: Display licenses of all dependencies
6. **Download Statistics**: Show download counts from PyPI
7. **Alternative Repos**: Support for custom package indexes
8. **Package Size**: Show disk space used by packages
9. **Conflict Resolution**: Detect and resolve dependency conflicts
10. **Build from Source**: Options for source/wheel installation

## Resources

- [pip Documentation](https://pip.pypa.io/)
- [PyPI JSON API](https://warehouse.pypa.io/api-reference/)
- [packaging library](https://packaging.pypa.io/)
- [Rich Progress Bars](https://rich.readthedocs.io/en/latest/progress.html)
- [Rich Prompts](https://rich.readthedocs.io/en/latest/prompt.html)

## Success Criteria

- [ ] Search and display package information
- [ ] Install packages with progress tracking
- [ ] Uninstall packages with confirmation
- [ ] List all installed packages
- [ ] Check for outdated packages
- [ ] Display dependency trees
- [ ] Show detailed package information
- [ ] Handle errors gracefully
- [ ] Interactive menu navigation
- [ ] Batch operations support
- [ ] Clear visual feedback for all operations

## Testing Checklist

- Test package installation in various scenarios
- Verify progress bars update correctly
- Test dependency tree with complex packages (like Django)
- Verify outdated package detection
- Test uninstall with confirmation
- Check error handling for non-existent packages
- Test with packages that have no dependencies
- Verify menu navigation works smoothly
- Test bulk installation operations
- Check terminal display on different screen sizes
