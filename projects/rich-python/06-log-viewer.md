# Project 6: Log Viewer with Syntax Highlighting

## Overview
Build a powerful log file viewer with syntax highlighting, filtering, searching, real-time tailing, and log level visualization - making it easier to parse and analyze application logs directly in the terminal.

## Learning Objectives
- Parse and colorize log formats (JSON, syslog, custom)
- Implement real-time file tailing (like `tail -f`)
- Create efficient filtering for large log files
- Build pattern matching and search functionality
- Handle multiple log formats
- Implement log level severity coloring

## Difficulty Level
**Intermediate to Advanced** - Requires file I/O, regex parsing, and real-time updates

## Technical Stack
- **Rich**: Console, Syntax, Table, Live, Panel, Text
- **Python stdlib**: re, pathlib, datetime, collections
- **Optional**: watchdog (file monitoring), dateutil (date parsing)

## Requirements

### Core Features
1. **Multi-format Support**: JSON logs, syslog, custom formats
2. **Log Level Highlighting**: DEBUG, INFO, WARN, ERROR, CRITICAL
3. **Real-time Tailing**: Watch logs as they update
4. **Filtering**: By level, timestamp, keywords
5. **Search**: Regex and text search
6. **Statistics**: Log level distribution

### Layout Components
1. **Log Display**: Formatted log entries with colors
2. **Filter Panel**: Active filters display
3. **Statistics Bar**: Count by log level
4. **Search Results**: Highlighted matches
5. **Timestamp Column**: Formatted timestamps
6. **Status Bar**: File info, line count, update status

### Interaction Features
- Scroll through logs
- Filter by log level
- Search with regex
- Follow mode (real-time)
- Export filtered logs
- Jump to timestamp

## Step-by-Step Implementation

### Step 1: Log Parser Framework

```python
# log_viewer.py
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from pathlib import Path
from typing import Optional, List, Dict
import re
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.text import Text

class LogLevel(Enum):
    DEBUG = "DEBUG"
    INFO = "INFO"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"
    UNKNOWN = "UNKNOWN"

@dataclass
class LogEntry:
    timestamp: Optional[datetime]
    level: LogLevel
    message: str
    logger: Optional[str] = None
    extra: Optional[Dict] = None
    raw_line: str = ""

class LogParser:
    def __init__(self):
        # Common log patterns
        self.patterns = {
            # Python logging: 2024-01-15 10:30:45,123 - logger - ERROR - message
            'python': re.compile(
                r'(?P<timestamp>\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}(?:,\d{3})?)'
                r'\s+-\s+(?P<logger>[\w\.]+)'
                r'\s+-\s+(?P<level>\w+)'
                r'\s+-\s+(?P<message>.*)'
            ),

            # Syslog: Jan 15 10:30:45 hostname program[pid]: message
            'syslog': re.compile(
                r'(?P<timestamp>\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})'
                r'\s+(?P<hostname>\S+)'
                r'\s+(?P<program>\S+?)(\[(?P<pid>\d+)\])?:'
                r'\s+(?P<message>.*)'
            ),

            # Apache/Nginx: [timestamp] [level] message
            'apache': re.compile(
                r'\[(?P<timestamp>[^\]]+)\]'
                r'\s+\[(?P<level>\w+)\]'
                r'\s+(?P<message>.*)'
            ),

            # Simple: LEVEL: message
            'simple': re.compile(
                r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL):\s+(?P<message>.*)'
            ),
        }

    def parse_level(self, level_str: str) -> LogLevel:
        """Parse log level from string."""
        level_str = level_str.upper()

        # Handle variations
        if 'DEBUG' in level_str or 'DBG' in level_str:
            return LogLevel.DEBUG
        elif 'INFO' in level_str or 'INF' in level_str:
            return LogLevel.INFO
        elif 'WARN' in level_str:
            return LogLevel.WARNING
        elif 'ERROR' in level_str or 'ERR' in level_str:
            return LogLevel.ERROR
        elif 'CRITICAL' in level_str or 'CRIT' in level_str or 'FATAL' in level_str:
            return LogLevel.CRITICAL
        else:
            return LogLevel.UNKNOWN

    def parse_line(self, line: str) -> LogEntry:
        """Parse a single log line."""
        line = line.strip()

        if not line:
            return None

        # Try each pattern
        for pattern_name, pattern in self.patterns.items():
            match = pattern.match(line)

            if match:
                groups = match.groupdict()

                # Parse timestamp
                timestamp = None
                if 'timestamp' in groups and groups['timestamp']:
                    timestamp = self.parse_timestamp(groups['timestamp'])

                # Parse level
                level = LogLevel.UNKNOWN
                if 'level' in groups and groups['level']:
                    level = self.parse_level(groups['level'])

                # Extract message and logger
                message = groups.get('message', line)
                logger = groups.get('logger') or groups.get('program')

                # Extra fields
                extra = {k: v for k, v in groups.items()
                        if k not in ['timestamp', 'level', 'message', 'logger', 'program']
                        and v is not None}

                return LogEntry(
                    timestamp=timestamp,
                    level=level,
                    message=message,
                    logger=logger,
                    extra=extra if extra else None,
                    raw_line=line
                )

        # Fallback: treat as plain message
        return LogEntry(
            timestamp=None,
            level=LogLevel.UNKNOWN,
            message=line,
            raw_line=line
        )

    def parse_timestamp(self, ts_str: str) -> Optional[datetime]:
        """Parse various timestamp formats."""
        formats = [
            '%Y-%m-%d %H:%M:%S,%f',
            '%Y-%m-%d %H:%M:%S.%f',
            '%Y-%m-%d %H:%M:%S',
            '%b %d %H:%M:%S',  # Syslog
            '%d/%b/%Y:%H:%M:%S %z',  # Apache
        ]

        for fmt in formats:
            try:
                return datetime.strptime(ts_str, fmt)
            except ValueError:
                continue

        return None

class LogViewer:
    def __init__(self, log_file: Path):
        self.log_file = Path(log_file)
        self.console = Console()
        self.parser = LogParser()
        self.entries: List[LogEntry] = []

    def load_logs(self, max_lines: int = None):
        """Load log file into memory."""
        if not self.log_file.exists():
            self.console.print(f"[red]File not found: {self.log_file}[/red]")
            return

        with open(self.log_file, 'r', encoding='utf-8', errors='ignore') as f:
            for i, line in enumerate(f):
                if max_lines and i >= max_lines:
                    break

                entry = self.parser.parse_line(line)
                if entry:
                    self.entries.append(entry)

    def get_level_color(self, level: LogLevel) -> str:
        """Get color for log level."""
        colors = {
            LogLevel.DEBUG: "dim cyan",
            LogLevel.INFO: "green",
            LogLevel.WARNING: "yellow",
            LogLevel.ERROR: "red",
            LogLevel.CRITICAL: "bold white on red",
            LogLevel.UNKNOWN: "dim"
        }
        return colors.get(level, "white")

    def get_level_icon(self, level: LogLevel) -> str:
        """Get icon for log level."""
        icons = {
            LogLevel.DEBUG: "🔍",
            LogLevel.INFO: "ℹ️",
            LogLevel.WARNING: "⚠️",
            LogLevel.ERROR: "❌",
            LogLevel.CRITICAL: "🔥",
            LogLevel.UNKNOWN: "❓"
        }
        return icons.get(level, "·")

# Usage
if __name__ == "__main__":
    viewer = LogViewer("app.log")
    viewer.load_logs()
    print(f"Loaded {len(viewer.entries)} log entries")
```

### Step 2: Formatted Log Display

```python
from rich.table import Table
from rich import box

class LogViewer:
    # ... previous code ...

    def display_logs(
        self,
        start: int = 0,
        count: int = 50,
        filter_level: Optional[LogLevel] = None,
        search_term: Optional[str] = None
    ):
        """Display logs in formatted table."""
        # Filter entries
        filtered = self.entries[start:]

        if filter_level:
            filtered = [e for e in filtered if e.level == filter_level]

        if search_term:
            pattern = re.compile(search_term, re.IGNORECASE)
            filtered = [e for e in filtered if pattern.search(e.message)]

        # Limit count
        filtered = filtered[:count]

        if not filtered:
            self.console.print("[dim]No log entries to display[/dim]")
            return

        # Create table
        table = Table(
            title=f"[bold]Log Viewer - {self.log_file.name}[/bold]",
            show_header=True,
            header_style="bold magenta",
            box=box.SIMPLE,
            expand=True
        )

        table.add_column("Time", style="cyan", width=20)
        table.add_column("Level", width=10)
        table.add_column("Logger", style="blue", width=15, overflow="ellipsis")
        table.add_column("Message", style="white", overflow="fold")

        for entry in filtered:
            # Format timestamp
            time_str = ""
            if entry.timestamp:
                time_str = entry.timestamp.strftime("%Y-%m-%d %H:%M:%S")

            # Format level with color
            level_color = self.get_level_color(entry.level)
            level_icon = self.get_level_icon(entry.level)
            level_str = f"[{level_color}]{level_icon} {entry.level.value}[/{level_color}]"

            # Highlight search term in message
            message = entry.message
            if search_term:
                # Highlight matches
                message = re.sub(
                    f'({search_term})',
                    r'[bold yellow on black]\1[/bold yellow on black]',
                    message,
                    flags=re.IGNORECASE
                )

            table.add_row(
                time_str,
                level_str,
                entry.logger or "-",
                message
            )

        self.console.print(table)
        self.console.print(f"\n[dim]Showing {len(filtered)} of {len(self.entries)} entries[/dim]")

    def display_entry_details(self, entry: LogEntry):
        """Display detailed view of a log entry."""
        details = Text()

        # Timestamp
        if entry.timestamp:
            details.append("Timestamp: ", style="cyan bold")
            details.append(f"{entry.timestamp}\n", style="white")

        # Level
        level_color = self.get_level_color(entry.level)
        details.append("Level: ", style="cyan bold")
        details.append(f"{entry.level.value}\n", style=level_color)

        # Logger
        if entry.logger:
            details.append("Logger: ", style="cyan bold")
            details.append(f"{entry.logger}\n", style="blue")

        # Message
        details.append("Message: ", style="cyan bold")
        details.append(f"{entry.message}\n", style="white")

        # Extra fields
        if entry.extra:
            details.append("\nExtra Fields:\n", style="cyan bold")
            for key, value in entry.extra.items():
                details.append(f"  {key}: ", style="dim")
                details.append(f"{value}\n", style="white")

        # Raw line
        details.append("\nRaw:\n", style="cyan bold")
        details.append(entry.raw_line, style="dim")

        panel = Panel(
            details,
            title="[bold]Log Entry Details[/bold]",
            border_style="blue"
        )

        self.console.print(panel)

# Usage
if __name__ == "__main__":
    viewer = LogViewer("app.log")
    viewer.load_logs()

    # Display all logs
    viewer.display_logs()

    # Display only errors
    viewer.display_logs(filter_level=LogLevel.ERROR)

    # Search for specific term
    viewer.display_logs(search_term="exception")
```

### Step 3: Statistics and Analysis

```python
from collections import Counter
from rich.progress import Progress, BarColumn, TextColumn

class LogViewer:
    # ... previous code ...

    def get_statistics(self) -> Dict:
        """Calculate log statistics."""
        stats = {
            'total': len(self.entries),
            'by_level': Counter(e.level for e in self.entries),
            'by_logger': Counter(e.logger for e in self.entries if e.logger),
            'time_range': None,
            'entries_per_hour': {}
        }

        # Time range
        timestamps = [e.timestamp for e in self.entries if e.timestamp]
        if timestamps:
            stats['time_range'] = (min(timestamps), max(timestamps))

            # Entries per hour
            for ts in timestamps:
                hour = ts.replace(minute=0, second=0, microsecond=0)
                stats['entries_per_hour'][hour] = stats['entries_per_hour'].get(hour, 0) + 1

        return stats

    def display_statistics(self):
        """Display log statistics."""
        stats = self.get_statistics()

        self.console.print("\n[bold cyan]Log Statistics[/bold cyan]\n")

        # Basic stats
        self.console.print(f"Total Entries: [white]{stats['total']}[/white]")

        if stats['time_range']:
            start, end = stats['time_range']
            self.console.print(f"Time Range: [white]{start} to {end}[/white]")
            duration = end - start
            self.console.print(f"Duration: [white]{duration}[/white]")

        # Log level distribution
        self.console.print("\n[bold]Distribution by Level:[/bold]\n")

        total = stats['total']
        with Progress(
            TextColumn("[progress.description]{task.description}"),
            BarColumn(bar_width=40),
            TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
            TextColumn("({task.completed}/{task.total})"),
            console=self.console
        ) as progress:

            for level in LogLevel:
                count = stats['by_level'].get(level, 0)
                if count > 0:
                    color = self.get_level_color(level)
                    icon = self.get_level_icon(level)

                    task = progress.add_task(
                        f"[{color}]{icon} {level.value:10}[/{color}]",
                        total=total,
                        completed=count
                    )

        # Top loggers
        if stats['by_logger']:
            self.console.print("\n[bold]Top Loggers:[/bold]\n")

            table = Table(show_header=True, box=box.SIMPLE)
            table.add_column("Logger", style="blue")
            table.add_column("Count", justify="right", style="cyan")
            table.add_column("Percentage", justify="right", style="green")

            for logger, count in stats['by_logger'].most_common(10):
                percentage = (count / total) * 100
                table.add_row(
                    logger,
                    str(count),
                    f"{percentage:.1f}%"
                )

            self.console.print(table)

        # Activity timeline
        if stats['entries_per_hour']:
            self.console.print("\n[bold]Activity Timeline (entries per hour):[/bold]\n")

            sorted_hours = sorted(stats['entries_per_hour'].items())
            max_count = max(stats['entries_per_hour'].values())

            for hour, count in sorted_hours[:24]:  # Show last 24 hours
                bar_length = int((count / max_count) * 30)
                bar = "█" * bar_length

                self.console.print(
                    f"{hour.strftime('%Y-%m-%d %H:00')} "
                    f"[cyan]{bar}[/cyan] {count}"
                )

    def search_logs(self, pattern: str, case_sensitive: bool = False) -> List[LogEntry]:
        """Search logs with regex pattern."""
        flags = 0 if case_sensitive else re.IGNORECASE
        regex = re.compile(pattern, flags)

        matches = []
        for entry in self.entries:
            if regex.search(entry.message) or (entry.logger and regex.search(entry.logger)):
                matches.append(entry)

        return matches

    def display_search_results(self, pattern: str):
        """Display search results."""
        matches = self.search_logs(pattern)

        if not matches:
            self.console.print(f"[yellow]No matches found for '{pattern}'[/yellow]")
            return

        self.console.print(f"\n[bold]Search Results for '{pattern}'[/bold]")
        self.console.print(f"[dim]Found {len(matches)} matches[/dim]\n")

        # Display matches
        for i, entry in enumerate(matches[:50], 1):  # Limit to 50 results
            level_color = self.get_level_color(entry.level)
            level_icon = self.get_level_icon(entry.level)

            time_str = ""
            if entry.timestamp:
                time_str = entry.timestamp.strftime("%H:%M:%S")

            # Highlight pattern in message
            highlighted_message = re.sub(
                f'({pattern})',
                r'[bold yellow on black]\1[/bold yellow on black]',
                entry.message,
                flags=re.IGNORECASE
            )

            self.console.print(
                f"[dim]{i:3d}.[/dim] "
                f"[dim]{time_str}[/dim] "
                f"[{level_color}]{level_icon}[/{level_color}] "
                f"{highlighted_message}"
            )

        if len(matches) > 50:
            self.console.print(f"\n[dim]... and {len(matches) - 50} more matches[/dim]")

# Usage
if __name__ == "__main__":
    viewer = LogViewer("app.log")
    viewer.load_logs()

    # Display statistics
    viewer.display_statistics()

    # Search logs
    viewer.display_search_results("error|exception|failed")
```

### Step 4: Real-time Log Tailing

```python
from rich.live import Live
import time
from threading import Thread, Event

class LogTailer:
    def __init__(self, log_file: Path, viewer: LogViewer):
        self.log_file = log_file
        self.viewer = viewer
        self.stop_event = Event()
        self.new_entries: List[LogEntry] = []
        self.last_position = 0

    def tail(self):
        """Tail log file for new entries."""
        # Get initial file size
        if self.log_file.exists():
            with open(self.log_file, 'r') as f:
                f.seek(0, 2)  # Seek to end
                self.last_position = f.tell()

        while not self.stop_event.is_set():
            if self.log_file.exists():
                with open(self.log_file, 'r', encoding='utf-8', errors='ignore') as f:
                    f.seek(self.last_position)

                    for line in f:
                        entry = self.viewer.parser.parse_line(line)
                        if entry:
                            self.new_entries.append(entry)
                            self.viewer.entries.append(entry)

                    self.last_position = f.tell()

            time.sleep(0.5)  # Check every 500ms

    def start(self):
        """Start tailing in background thread."""
        self.thread = Thread(target=self.tail, daemon=True)
        self.thread.start()

    def stop(self):
        """Stop tailing."""
        self.stop_event.set()
        if hasattr(self, 'thread'):
            self.thread.join(timeout=1)

class LogViewer:
    # ... previous code ...

    def follow_logs(self, filter_level: Optional[LogLevel] = None, max_lines: int = 20):
        """Follow log file in real-time (like tail -f)."""
        self.console.print(f"[bold cyan]Following {self.log_file}[/bold cyan]")
        self.console.print("[dim]Press Ctrl+C to stop[/dim]\n")

        tailer = LogTailer(self.log_file, self)
        tailer.start()

        try:
            with Live(console=self.console, refresh_per_second=2, screen=False) as live:
                last_displayed = len(self.entries)

                while True:
                    # Get new entries
                    new_entries = self.entries[last_displayed:]

                    if filter_level:
                        new_entries = [e for e in new_entries if e.level == filter_level]

                    # Display new entries
                    if new_entries:
                        for entry in new_entries[-max_lines:]:
                            level_color = self.get_level_color(entry.level)
                            level_icon = self.get_level_icon(entry.level)

                            time_str = ""
                            if entry.timestamp:
                                time_str = entry.timestamp.strftime("%H:%M:%S.%f")[:-3]

                            text = Text()
                            text.append(f"{time_str} ", style="dim")
                            text.append(f"{level_icon} ", style=level_color)

                            if entry.logger:
                                text.append(f"[{entry.logger}] ", style="blue")

                            text.append(entry.message, style=level_color if entry.level == LogLevel.ERROR else "white")

                            self.console.print(text)

                    last_displayed = len(self.entries)
                    time.sleep(0.5)

        except KeyboardInterrupt:
            tailer.stop()
            self.console.print("\n[dim]Stopped following logs[/dim]")

# Usage
if __name__ == "__main__":
    viewer = LogViewer("app.log")
    viewer.load_logs()

    # Follow logs in real-time
    viewer.follow_logs()

    # Follow only errors
    viewer.follow_logs(filter_level=LogLevel.ERROR)
```

### Step 5: JSON Log Support and Export

```python
import json

class LogViewer:
    # ... previous code ...

    def parse_json_logs(self, json_field_mapping: Dict = None):
        """Parse JSON formatted logs."""
        if json_field_mapping is None:
            json_field_mapping = {
                'timestamp': ['timestamp', 'time', '@timestamp'],
                'level': ['level', 'severity', 'levelname'],
                'message': ['message', 'msg'],
                'logger': ['logger', 'name', 'logger_name']
            }

        with open(self.log_file, 'r', encoding='utf-8', errors='ignore') as f:
            for line in f:
                line = line.strip()
                if not line:
                    continue

                try:
                    log_obj = json.loads(line)

                    # Extract fields using mapping
                    timestamp_str = None
                    for field in json_field_mapping['timestamp']:
                        if field in log_obj:
                            timestamp_str = log_obj[field]
                            break

                    level_str = None
                    for field in json_field_mapping['level']:
                        if field in log_obj:
                            level_str = log_obj[field]
                            break

                    message = None
                    for field in json_field_mapping['message']:
                        if field in log_obj:
                            message = log_obj[field]
                            break

                    logger = None
                    for field in json_field_mapping['logger']:
                        if field in log_obj:
                            logger = log_obj[field]
                            break

                    # Parse timestamp
                    timestamp = None
                    if timestamp_str:
                        timestamp = self.parser.parse_timestamp(timestamp_str)

                    # Parse level
                    level = LogLevel.UNKNOWN
                    if level_str:
                        level = self.parser.parse_level(level_str)

                    # Create entry
                    entry = LogEntry(
                        timestamp=timestamp,
                        level=level,
                        message=message or str(log_obj),
                        logger=logger,
                        extra={k: v for k, v in log_obj.items()
                              if k not in ['timestamp', 'time', 'level', 'message', 'logger']},
                        raw_line=line
                    )

                    self.entries.append(entry)

                except json.JSONDecodeError:
                    # Not JSON, try regular parsing
                    entry = self.parser.parse_line(line)
                    if entry:
                        self.entries.append(entry)

    def export_logs(
        self,
        output_file: Path,
        format: str = 'json',
        filter_level: Optional[LogLevel] = None
    ):
        """Export logs to file."""
        # Filter entries
        entries = self.entries
        if filter_level:
            entries = [e for e in entries if e.level == filter_level]

        output_file = Path(output_file)

        if format == 'json':
            # Export as JSON
            json_entries = []
            for entry in entries:
                json_entry = {
                    'timestamp': entry.timestamp.isoformat() if entry.timestamp else None,
                    'level': entry.level.value,
                    'message': entry.message,
                    'logger': entry.logger
                }
                if entry.extra:
                    json_entry['extra'] = entry.extra

                json_entries.append(json_entry)

            with open(output_file, 'w') as f:
                json.dump(json_entries, f, indent=2)

        elif format == 'csv':
            # Export as CSV
            import csv

            with open(output_file, 'w', newline='') as f:
                writer = csv.writer(f)
                writer.writerow(['Timestamp', 'Level', 'Logger', 'Message'])

                for entry in entries:
                    writer.writerow([
                        entry.timestamp.isoformat() if entry.timestamp else '',
                        entry.level.value,
                        entry.logger or '',
                        entry.message
                    ])

        elif format == 'text':
            # Export as plain text
            with open(output_file, 'w') as f:
                for entry in entries:
                    f.write(entry.raw_line + '\n')

        self.console.print(f"[green]✓[/green] Exported {len(entries)} entries to {output_file}")

# Usage
if __name__ == "__main__":
    # Parse JSON logs
    viewer = LogViewer("app.json")
    viewer.parse_json_logs()
    viewer.display_logs()

    # Export filtered logs
    viewer.export_logs("errors.json", format='json', filter_level=LogLevel.ERROR)
    viewer.export_logs("all_logs.csv", format='csv')
```

## Expected Output

### Log Display
```
╭─ Log Viewer - app.log ───────────────────────────────────╮
│ Time                Level      Logger     Message        │
│ 2024-01-15 10:30:45 ℹ️ INFO    api.auth   User logged in │
│ 2024-01-15 10:30:46 ⚠️ WARNING db.conn    Slow query     │
│ 2024-01-15 10:30:50 ❌ ERROR   api.orders Order failed   │
│ 2024-01-15 10:31:00 🔥 CRITICAL sys.db    DB offline!    │
╰──────────────────────────────────────────────────────────╯
```

### Statistics
```
Log Statistics

Total Entries: 1,234
Time Range: 2024-01-15 00:00:00 to 2024-01-15 23:59:59

Distribution by Level:
ℹ️ INFO      ████████████████████████░░░ 75% (925/1234)
⚠️ WARNING   █████░░░░░░░░░░░░░░░░░░░░░ 15% (185/1234)
❌ ERROR     ███░░░░░░░░░░░░░░░░░░░░░░░ 8%  (99/1234)
🔥 CRITICAL  ░░░░░░░░░░░░░░░░░░░░░░░░░░ 2%  (25/1234)
```

## Bonus Challenges

1. **Multi-file Support**: View multiple log files simultaneously
2. **Log Rotation Handling**: Handle rotated logs (.1, .2, .gz)
3. **Custom Parsers**: User-defined regex patterns
4. **Bookmark Lines**: Mark important log lines
5. **Diff Mode**: Compare two log files
6. **Alert Rules**: Notify on specific patterns
7. **Performance Mode**: Stream large files without loading all
8. **Graph Visualization**: Plot error rates over time
9. **Correlate Logs**: Link related log entries
10. **Context View**: Show lines before/after match

## Resources

- [Python Logging](https://docs.python.org/3/library/logging.html)
- [Regular Expressions](https://docs.python.org/3/library/re.html)
- [Rich Live Display](https://rich.readthedocs.io/en/latest/live.html)
- [watchdog - File monitoring](https://python-watchdog.readthedocs.io/)

## Success Criteria

- [ ] Parse multiple log formats correctly
- [ ] Display logs with syntax highlighting
- [ ] Filter by log level
- [ ] Search with regex support
- [ ] Real-time log tailing
- [ ] Display statistics and analysis
- [ ] Handle large log files efficiently
- [ ] Export filtered logs
- [ ] Color-code log levels appropriately
- [ ] Show timestamps in readable format

## Testing Checklist

- Test with various log formats (Python, syslog, JSON)
- Verify with large log files (100MB+)
- Test real-time tailing with active logs
- Check regex search performance
- Test filtering by multiple criteria
- Verify timestamp parsing accuracy
- Test export in different formats
- Check memory usage with large files
- Verify color rendering in terminals
- Test with malformed log entries
