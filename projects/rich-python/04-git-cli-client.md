# Project 4: Git CLI Client with Rich Formatting

## Overview
Create a beautiful Git command-line interface that enhances the standard Git experience with rich formatting, interactive commit browsing, visual diff displays, and branch management - making Git more accessible and visually appealing.

## Learning Objectives
- Parse and format Git command output
- Create syntax-highlighted diff displays
- Build interactive commit browsers
- Implement branch visualization
- Handle Git workflows programmatically
- Design intuitive CLI tools for developers

## Difficulty Level
**Advanced** - Requires understanding of Git internals, parsing complex outputs, and subprocess management

## Technical Stack
- **Rich**: Console, Syntax, Table, Tree, Panel, Prompt, Live
- **Python stdlib**: subprocess, re, datetime, pathlib
- **GitPython**: Optional for advanced Git operations
- **pygments**: For syntax highlighting diffs

## Requirements

### Core Features
1. **Enhanced Status**: Beautiful git status with file categories
2. **Commit Browser**: Interactive commit history viewer
3. **Diff Viewer**: Syntax-highlighted diff display
4. **Branch Manager**: Visual branch listing and switching
5. **Stash Manager**: Interactive stash operations
6. **Remote Viewer**: Display remote repositories

### Layout Components
1. **Status Dashboard**: Working tree state overview
2. **Commit Table**: Paginated commit history
3. **Diff Panel**: Side-by-side or unified diff view
4. **Branch Tree**: Visual branch hierarchy
5. **File Tree**: Changed files with status indicators
6. **Command Output**: Formatted Git command results

### Interaction Features
- Browse commits with arrow keys
- View commit details and diffs
- Stage/unstage files interactively
- Create commits with formatted prompts
- Switch branches safely
- Search commit history

## Step-by-Step Implementation

### Step 1: Basic Git Wrapper

```python
# git_client.py
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.syntax import Syntax
from rich import box
import subprocess
import re
from datetime import datetime
from typing import List, Dict, Tuple
from dataclasses import dataclass

@dataclass
class GitCommit:
    hash: str
    short_hash: str
    author: str
    email: str
    date: datetime
    message: str
    refs: List[str] = None

    def __post_init__(self):
        if self.refs is None:
            self.refs = []

@dataclass
class GitFileStatus:
    status: str  # M, A, D, R, etc.
    file_path: str
    old_path: str = None  # For renames

class GitClient:
    def __init__(self, repo_path: str = "."):
        self.repo_path = repo_path
        self.console = Console()

    def run_git(self, *args) -> Tuple[bool, str]:
        """Execute git command and return output."""
        try:
            result = subprocess.run(
                ["git", "-C", self.repo_path] + list(args),
                capture_output=True,
                text=True,
                check=True
            )
            return True, result.stdout
        except subprocess.CalledProcessError as e:
            return False, e.stderr

    def get_repo_name(self) -> str:
        """Get repository name from remote URL."""
        success, output = self.run_git("remote", "get-url", "origin")
        if success and output:
            # Extract repo name from URL
            match = re.search(r'/([^/]+?)(\.git)?$', output.strip())
            if match:
                return match.group(1)
        return "Unknown Repository"

    def get_current_branch(self) -> str:
        """Get current branch name."""
        success, output = self.run_git("rev-parse", "--abbrev-ref", "HEAD")
        return output.strip() if success else "unknown"

    def is_git_repo(self) -> bool:
        """Check if directory is a git repository."""
        success, _ = self.run_git("rev-parse", "--git-dir")
        return success

    def get_status_icon(self, status: str) -> str:
        """Get icon for file status."""
        icons = {
            'M': '📝',  # Modified
            'A': '➕',  # Added
            'D': '➖',  # Deleted
            'R': '🔄',  # Renamed
            'C': '📋',  # Copied
            'U': '⚠️',  # Updated but unmerged
            '?': '❓',  # Untracked
            '!': '🚫',  # Ignored
        }
        return icons.get(status, '❔')

    def get_status_color(self, status: str) -> str:
        """Get color for file status."""
        colors = {
            'M': 'yellow',
            'A': 'green',
            'D': 'red',
            'R': 'blue',
            'C': 'cyan',
            'U': 'magenta',
            '?': 'dim',
            '!': 'dim',
        }
        return colors.get(status, 'white')

    def display_header(self):
        """Display repository header."""
        repo_name = self.get_repo_name()
        branch = self.get_current_branch()

        header_text = (
            f"[bold cyan]{repo_name}[/bold cyan] "
            f"on [bold magenta]{branch}[/bold magenta]"
        )

        self.console.print(Panel(header_text, style="white on blue"))

# Usage
if __name__ == "__main__":
    git = GitClient()

    if git.is_git_repo():
        git.display_header()
    else:
        print("Not a git repository")
```

### Step 2: Enhanced Git Status

```python
class GitClient:
    # ... previous code ...

    def parse_status(self) -> Dict[str, List[GitFileStatus]]:
        """Parse git status output."""
        success, output = self.run_git("status", "--porcelain")

        if not success:
            return {}

        categorized = {
            'staged': [],
            'modified': [],
            'untracked': []
        }

        for line in output.strip().split('\n'):
            if not line:
                continue

            # Parse porcelain format: XY filename
            index_status = line[0]
            worktree_status = line[1]
            file_path = line[3:]

            # Handle renames
            if '->' in file_path:
                old_path, file_path = file_path.split(' -> ')
                old_path = old_path.strip()
                file_path = file_path.strip()
            else:
                old_path = None

            # Categorize files
            if index_status != ' ' and index_status != '?':
                # File is staged
                categorized['staged'].append(
                    GitFileStatus(index_status, file_path, old_path)
                )

            if worktree_status != ' ' and worktree_status != '?':
                # File is modified but not staged
                categorized['modified'].append(
                    GitFileStatus(worktree_status, file_path, old_path)
                )

            if index_status == '?' and worktree_status == '?':
                # Untracked file
                categorized['untracked'].append(
                    GitFileStatus('?', file_path, old_path)
                )

        return categorized

    def display_status(self):
        """Display enhanced git status."""
        self.display_header()
        self.console.print()

        status = self.parse_status()

        # Staged files
        if status['staged']:
            table = Table(
                title="[bold green]Changes to be committed[/bold green]",
                show_header=False,
                box=box.SIMPLE,
                padding=(0, 2)
            )
            table.add_column("Icon", width=4)
            table.add_column("Status", style="cyan", width=10)
            table.add_column("File", style="green")

            for file in status['staged']:
                icon = self.get_status_icon(file.status)
                status_text = {
                    'M': 'modified',
                    'A': 'new file',
                    'D': 'deleted',
                    'R': 'renamed',
                }.get(file.status, file.status)

                file_display = file.file_path
                if file.old_path:
                    file_display = f"{file.old_path} → {file.file_path}"

                table.add_row(icon, status_text, file_display)

            self.console.print(table)
            self.console.print()

        # Modified files
        if status['modified']:
            table = Table(
                title="[bold yellow]Changes not staged for commit[/bold yellow]",
                show_header=False,
                box=box.SIMPLE,
                padding=(0, 2)
            )
            table.add_column("Icon", width=4)
            table.add_column("Status", style="cyan", width=10)
            table.add_column("File", style="yellow")

            for file in status['modified']:
                icon = self.get_status_icon(file.status)
                status_text = {
                    'M': 'modified',
                    'D': 'deleted',
                }.get(file.status, file.status)

                table.add_row(icon, status_text, file.file_path)

            self.console.print(table)
            self.console.print()

        # Untracked files
        if status['untracked']:
            table = Table(
                title="[bold dim]Untracked files[/bold dim]",
                show_header=False,
                box=box.SIMPLE,
                padding=(0, 2)
            )
            table.add_column("Icon", width=4)
            table.add_column("File", style="dim")

            for file in status['untracked']:
                table.add_row(self.get_status_icon('?'), file.file_path)

            self.console.print(table)
            self.console.print()

        # No changes
        if not any(status.values()):
            self.console.print("[green]✓ Working tree clean[/green]")

# Usage
if __name__ == "__main__":
    git = GitClient()
    git.display_status()
```

### Step 3: Commit History Browser

```python
class GitClient:
    # ... previous code ...

    def parse_commits(self, max_count: int = 50) -> List[GitCommit]:
        """Parse commit history."""
        # Custom format for easy parsing
        format_str = "%H%n%h%n%an%n%ae%n%at%n%D%n%s%n%b%n---COMMIT-END---"

        success, output = self.run_git(
            "log",
            f"--max-count={max_count}",
            f"--format={format_str}"
        )

        if not success:
            return []

        commits = []
        commit_texts = output.split('---COMMIT-END---')

        for commit_text in commit_texts:
            lines = commit_text.strip().split('\n')
            if len(lines) < 7:
                continue

            hash_full = lines[0]
            hash_short = lines[1]
            author = lines[2]
            email = lines[3]
            timestamp = int(lines[4])
            refs = [r.strip() for r in lines[5].split(',') if r.strip()]
            subject = lines[6]

            # Combine remaining lines as body
            body = '\n'.join(lines[7:]).strip()
            message = f"{subject}\n\n{body}" if body else subject

            commits.append(GitCommit(
                hash=hash_full,
                short_hash=hash_short,
                author=author,
                email=email,
                date=datetime.fromtimestamp(timestamp),
                message=message,
                refs=refs
            ))

        return commits

    def display_commits(self, max_count: int = 20):
        """Display commit history in a table."""
        commits = self.parse_commits(max_count)

        table = Table(
            title="[bold]Commit History[/bold]",
            show_header=True,
            header_style="bold magenta",
            box=box.ROUNDED
        )

        table.add_column("Hash", style="cyan", width=8)
        table.add_column("Author", style="green", width=20)
        table.add_column("Date", style="blue", width=20)
        table.add_column("Message", style="white", overflow="fold")
        table.add_column("Refs", style="yellow", width=15)

        for commit in commits:
            # Format date
            now = datetime.now()
            delta = now - commit.date

            if delta.days == 0:
                if delta.seconds < 3600:
                    date_str = f"{delta.seconds // 60}m ago"
                else:
                    date_str = f"{delta.seconds // 3600}h ago"
            elif delta.days == 1:
                date_str = "yesterday"
            elif delta.days < 7:
                date_str = f"{delta.days}d ago"
            else:
                date_str = commit.date.strftime("%Y-%m-%d")

            # Get first line of message
            message = commit.message.split('\n')[0]
            if len(message) > 50:
                message = message[:47] + "..."

            # Format refs
            refs_str = ""
            if commit.refs:
                refs_str = ", ".join(commit.refs[:2])
                if len(commit.refs) > 2:
                    refs_str += "..."

            table.add_row(
                commit.short_hash,
                commit.author,
                date_str,
                message,
                refs_str
            )

        self.console.print(table)

    def display_commit_details(self, commit_hash: str):
        """Display detailed information about a commit."""
        commits = self.parse_commits(max_count=1000)

        # Find commit
        commit = None
        for c in commits:
            if c.hash.startswith(commit_hash) or c.short_hash == commit_hash:
                commit = c
                break

        if not commit:
            self.console.print(f"[red]Commit {commit_hash} not found[/red]")
            return

        # Create details panel
        details = f"""[bold]Commit:[/bold] {commit.hash}
[bold]Author:[/bold] {commit.author} <{commit.email}>
[bold]Date:[/bold] {commit.date.strftime('%Y-%m-%d %H:%M:%S')}
[bold]Refs:[/bold] {', '.join(commit.refs) if commit.refs else 'none'}

[bold]Message:[/bold]
{commit.message}
"""

        self.console.print(Panel(
            details,
            title=f"[bold cyan]{commit.short_hash}[/bold cyan]",
            border_style="cyan"
        ))

        # Show files changed
        success, stats = self.run_git("show", "--stat", commit.hash)
        if success:
            self.console.print("\n[bold]Files Changed:[/bold]")
            self.console.print(Syntax(stats, "diff", theme="monokai"))

# Usage
if __name__ == "__main__":
    git = GitClient()

    # Show commit history
    git.display_commits(20)

    # Show specific commit details
    git.display_commit_details("abc123")
```

### Step 4: Diff Viewer with Syntax Highlighting

```python
from rich.syntax import Syntax

class GitClient:
    # ... previous code ...

    def get_diff(self, file_path: str = None, staged: bool = False) -> str:
        """Get diff for file or all changes."""
        args = ["diff"]

        if staged:
            args.append("--cached")

        if file_path:
            args.append("--")
            args.append(file_path)

        success, output = self.run_git(*args)
        return output if success else ""

    def display_diff(self, file_path: str = None, staged: bool = False):
        """Display syntax-highlighted diff."""
        diff_output = self.get_diff(file_path, staged)

        if not diff_output.strip():
            self.console.print("[dim]No changes to display[/dim]")
            return

        # Display with syntax highlighting
        syntax = Syntax(
            diff_output,
            "diff",
            theme="monokai",
            line_numbers=True,
            word_wrap=True
        )

        title = "Staged Changes" if staged else "Working Directory Changes"
        if file_path:
            title += f" - {file_path}"

        panel = Panel(
            syntax,
            title=f"[bold]{title}[/bold]",
            border_style="blue"
        )

        self.console.print(panel)

    def display_file_diff(self, file_path: str):
        """Display side-by-side comparison for a file."""
        # Get diff
        success, diff_output = self.run_git("diff", "--", file_path)

        if not success or not diff_output.strip():
            self.console.print(f"[dim]No changes in {file_path}[/dim]")
            return

        # Parse diff to extract old and new content
        self.console.print(f"\n[bold]Diff for {file_path}[/bold]\n")

        # Show unified diff with syntax highlighting
        syntax = Syntax(
            diff_output,
            "diff",
            theme="monokai",
            line_numbers=False
        )

        self.console.print(syntax)

    def get_commit_diff(self, commit_hash: str) -> str:
        """Get diff for a specific commit."""
        success, output = self.run_git("show", commit_hash)
        return output if success else ""

    def display_commit_diff(self, commit_hash: str):
        """Display diff for a specific commit."""
        diff_output = self.get_commit_diff(commit_hash)

        if not diff_output.strip():
            self.console.print(f"[dim]No diff for commit {commit_hash}[/dim]")
            return

        syntax = Syntax(
            diff_output,
            "diff",
            theme="monokai",
            line_numbers=True,
            word_wrap=True
        )

        panel = Panel(
            syntax,
            title=f"[bold]Commit {commit_hash}[/bold]",
            border_style="cyan"
        )

        self.console.print(panel)

# Usage
if __name__ == "__main__":
    git = GitClient()

    # Show diff for all unstaged changes
    git.display_diff()

    # Show diff for staged changes
    git.display_diff(staged=True)

    # Show diff for specific file
    git.display_file_diff("README.md")

    # Show diff for commit
    git.display_commit_diff("abc123")
```

### Step 5: Branch Manager

```python
from rich.tree import Tree

class GitClient:
    # ... previous code ...

    def get_branches(self) -> Dict[str, List[str]]:
        """Get all branches (local and remote)."""
        branches = {
            'local': [],
            'remote': []
        }

        # Local branches
        success, output = self.run_git("branch", "--list")
        if success:
            for line in output.strip().split('\n'):
                if line:
                    # Remove '* ' prefix from current branch
                    branch = line.strip().replace('* ', '')
                    branches['local'].append(branch)

        # Remote branches
        success, output = self.run_git("branch", "-r", "--list")
        if success:
            for line in output.strip().split('\n'):
                if line and '->' not in line:  # Skip HEAD pointer
                    branch = line.strip()
                    branches['remote'].append(branch)

        return branches

    def display_branches(self):
        """Display branch list with current branch highlighted."""
        current_branch = self.get_current_branch()
        branches = self.get_branches()

        table = Table(
            title="[bold]Git Branches[/bold]",
            show_header=True,
            header_style="bold magenta",
            box=box.ROUNDED
        )

        table.add_column("Type", style="cyan", width=10)
        table.add_column("Branch", style="white", width=40)
        table.add_column("Status", style="green", width=10)

        # Local branches
        for branch in sorted(branches['local']):
            is_current = branch == current_branch
            branch_display = f"[bold green]{branch}[/bold green]" if is_current else branch
            status = "→" if is_current else ""

            table.add_row("local", branch_display, status)

        # Remote branches
        for branch in sorted(branches['remote'])[:10]:  # Show first 10
            table.add_row("remote", f"[dim]{branch}[/dim]", "")

        self.console.print(table)

    def create_branch(self, branch_name: str, checkout: bool = True):
        """Create a new branch."""
        if checkout:
            success, output = self.run_git("checkout", "-b", branch_name)
            if success:
                self.console.print(f"[green]✓[/green] Created and switched to branch '{branch_name}'")
            else:
                self.console.print(f"[red]✗[/red] Failed to create branch: {output}")
        else:
            success, output = self.run_git("branch", branch_name)
            if success:
                self.console.print(f"[green]✓[/green] Created branch '{branch_name}'")
            else:
                self.console.print(f"[red]✗[/red] Failed to create branch: {output}")

    def switch_branch(self, branch_name: str):
        """Switch to a different branch."""
        success, output = self.run_git("checkout", branch_name)

        if success:
            self.console.print(f"[green]✓[/green] Switched to branch '{branch_name}'")
        else:
            self.console.print(f"[red]✗[/red] Failed to switch branch: {output}")

    def delete_branch(self, branch_name: str, force: bool = False):
        """Delete a branch."""
        flag = "-D" if force else "-d"
        success, output = self.run_git("branch", flag, branch_name)

        if success:
            self.console.print(f"[green]✓[/green] Deleted branch '{branch_name}'")
        else:
            self.console.print(f"[red]✗[/red] Failed to delete branch: {output}")

    def display_branch_tree(self):
        """Display branch hierarchy as a tree."""
        current_branch = self.get_current_branch()

        tree = Tree(
            f"[bold blue]Repository Branches[/bold blue]",
            guide_style="dim"
        )

        # Local branches
        local = tree.add("[bold cyan]Local Branches[/bold cyan]")
        branches = self.get_branches()

        for branch in sorted(branches['local']):
            if branch == current_branch:
                local.add(f"→ [bold green]{branch}[/bold green] (current)")
            else:
                local.add(f"  [white]{branch}[/white]")

        # Remote branches
        remote = tree.add("[bold magenta]Remote Branches[/bold magenta]")
        for branch in sorted(branches['remote'])[:15]:
            remote.add(f"[dim]{branch}[/dim]")

        self.console.print(tree)

# Usage
if __name__ == "__main__":
    git = GitClient()

    # List branches
    git.display_branches()

    # Show branch tree
    git.display_branch_tree()

    # Create new branch
    git.create_branch("feature/new-feature")

    # Switch branch
    git.switch_branch("main")

    # Delete branch
    git.delete_branch("old-feature")
```

## Expected Output

### Enhanced Status
```
╭──────────────────────────────────────╮
│ my-project on main                   │
╰──────────────────────────────────────╯

Changes to be committed
  ➕  new file     src/feature.py
  📝  modified     README.md

Changes not staged for commit
  📝  modified     tests/test_main.py
  ➖  deleted      old_file.py

Untracked files
  ❓  temp.txt
  ❓  .vscode/
```

### Commit History
```
╭─ Commit History ─────────────────────────────────────╮
│ Hash    Author      Date        Message       Refs   │
│ a1b2c3  John Doe    2h ago      Add feature   main   │
│ d4e5f6  Jane Smith  yesterday   Fix bug       HEAD   │
│ g7h8i9  John Doe    3d ago      Update docs          │
╰──────────────────────────────────────────────────────╯
```

### Diff Viewer
```
╭─ Working Directory Changes ──────────────────────────╮
│ --- a/README.md                                      │
│ +++ b/README.md                                      │
│ @@ -10,6 +10,7 @@                                   │
│  ## Features                                         │
│                                                      │
│  - Fast and efficient                               │
│ +- New awesome feature                              │
│  - Easy to use                                      │
╰──────────────────────────────────────────────────────╯
```

## Bonus Challenges

1. **Interactive Staging**: Stage/unstage files with checkboxes
2. **Merge Conflict Helper**: Visual merge conflict resolution
3. **Rebase Interactive**: TUI for interactive rebase
4. **Tag Manager**: Create and manage tags
5. **Blame Viewer**: Annotated file view with commit info
6. **Stash Browser**: Visual stash management
7. **Cherry-pick Helper**: Interactive cherry-picking
8. **Reflog Viewer**: Navigate reflog history
9. **Submodule Manager**: Manage git submodules
10. **Graph Visualization**: ASCII art commit graph

## Resources

- [Git Documentation](https://git-scm.com/doc)
- [GitPython](https://gitpython.readthedocs.io/)
- [Rich Syntax Highlighting](https://rich.readthedocs.io/en/latest/syntax.html)
- [Git Porcelain Format](https://git-scm.com/docs/git-status#_short_format)

## Success Criteria

- [ ] Display enhanced git status with categorization
- [ ] Show commit history in formatted table
- [ ] Syntax-highlighted diff display
- [ ] Branch listing with current branch indicator
- [ ] Create and switch branches
- [ ] View commit details
- [ ] Display file diffs
- [ ] Parse git command output correctly
- [ ] Handle errors gracefully
- [ ] Clean and readable formatting

## Testing Checklist

- Test in repositories with various states (clean, dirty, merge conflicts)
- Verify with different numbers of branches
- Test with commits containing special characters
- Check diff display for various file types
- Test branch operations (create, switch, delete)
- Verify with repos that have no commits
- Test with detached HEAD state
- Check performance with large commit histories
- Verify color output in different terminals
