# Project 2: System Monitoring Dashboard with Live Updates

## Overview
Create a real-time system monitoring dashboard that displays CPU usage, memory statistics, disk I/O, network activity, and running processes - similar to htop or Windows Task Manager, but with Rich's beautiful formatting.

## Learning Objectives
- Master Rich's Live display for real-time updates
- Work with system metrics using psutil
- Create responsive layouts with multiple panels
- Implement data visualization with progress bars and sparklines
- Handle concurrent data collection efficiently
- Design information-dense interfaces clearly

## Difficulty Level
**Intermediate to Advanced** - Requires understanding of system metrics, async operations, and real-time UI updates

## Technical Stack
- **Rich**: Live, Table, Panel, Layout, ProgressBar, Console
- **psutil**: System and process monitoring
- **Python stdlib**: threading, collections, time, datetime
- **Optional**: plotext (for terminal graphs)

## Requirements

### Layout Components
1. **Header**: System info, uptime, load average
2. **CPU Panel**: Per-core usage with progress bars
3. **Memory Panel**: RAM and swap usage visualization
4. **Disk Panel**: Disk usage and I/O statistics
5. **Network Panel**: Network interfaces and traffic
6. **Process Table**: Top processes by CPU/memory
7. **Status Bar**: Update frequency, keyboard shortcuts

### Interaction Features
- Real-time updates (1-2 second refresh)
- Sort processes by CPU, memory, or PID
- Kill process functionality
- Pause/resume updates
- Toggle between different views
- Adjust update frequency

### Styling Requirements
- Color-coded resource usage (green → yellow → red)
- Clear progress bars for percentages
- Sparklines for historical data
- Formatted numbers (KB, MB, GB)
- Highlighting for high resource usage

## Step-by-Step Implementation

### Step 1: Basic System Information Display

```python
# system_monitor.py
import psutil
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.layout import Layout
from rich.progress import Progress, BarColumn, TextColumn
from datetime import datetime, timedelta
import platform

class SystemMonitor:
    def __init__(self):
        self.console = Console()

    def get_system_info(self) -> dict:
        """Collect basic system information."""
        boot_time = datetime.fromtimestamp(psutil.boot_time())
        uptime = datetime.now() - boot_time

        return {
            'hostname': platform.node(),
            'platform': platform.system(),
            'platform_release': platform.release(),
            'architecture': platform.machine(),
            'processor': platform.processor(),
            'cpu_count': psutil.cpu_count(logical=False),
            'cpu_count_logical': psutil.cpu_count(logical=True),
            'boot_time': boot_time,
            'uptime': uptime
        }

    def format_bytes(self, bytes_value: int) -> str:
        """Format bytes to human readable string."""
        for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
            if bytes_value < 1024.0:
                return f"{bytes_value:.2f} {unit}"
            bytes_value /= 1024.0
        return f"{bytes_value:.2f} PB"

    def format_uptime(self, uptime: timedelta) -> str:
        """Format uptime to readable string."""
        days = uptime.days
        hours, remainder = divmod(uptime.seconds, 3600)
        minutes, seconds = divmod(remainder, 60)

        if days > 0:
            return f"{days}d {hours}h {minutes}m"
        elif hours > 0:
            return f"{hours}h {minutes}m {seconds}s"
        else:
            return f"{minutes}m {seconds}s"

    def create_header(self) -> Panel:
        """Create header with system information."""
        info = self.get_system_info()

        header_text = (
            f"[bold cyan]{info['hostname']}[/bold cyan] | "
            f"{info['platform']} {info['platform_release']} | "
            f"{info['cpu_count']} cores ({info['cpu_count_logical']} logical) | "
            f"Uptime: {self.format_uptime(info['uptime'])}"
        )

        return Panel(
            header_text,
            title="[bold]System Monitor[/bold]",
            style="white on blue"
        )

    def display_basic_info(self):
        """Display basic system information."""
        self.console.print(self.create_header())

        # CPU Info
        cpu_freq = psutil.cpu_freq()
        cpu_percent = psutil.cpu_percent(interval=1)

        cpu_table = Table(title="CPU Information", show_header=True)
        cpu_table.add_column("Metric", style="cyan")
        cpu_table.add_column("Value", style="green")

        cpu_table.add_row("Usage", f"{cpu_percent}%")
        cpu_table.add_row("Frequency", f"{cpu_freq.current:.2f} MHz")
        cpu_table.add_row("Max Frequency", f"{cpu_freq.max:.2f} MHz")

        self.console.print(cpu_table)

        # Memory Info
        mem = psutil.virtual_memory()

        mem_table = Table(title="Memory Information", show_header=True)
        mem_table.add_column("Metric", style="cyan")
        mem_table.add_column("Value", style="green")

        mem_table.add_row("Total", self.format_bytes(mem.total))
        mem_table.add_row("Available", self.format_bytes(mem.available))
        mem_table.add_row("Used", self.format_bytes(mem.used))
        mem_table.add_row("Usage", f"{mem.percent}%")

        self.console.print(mem_table)

# Usage
if __name__ == "__main__":
    monitor = SystemMonitor()
    monitor.display_basic_info()
```

### Step 2: CPU Monitoring with Progress Bars

```python
from rich.progress import Progress, BarColumn, TextColumn, SpinnerColumn
from rich.table import Table
from rich import box

class SystemMonitor:
    # ... previous code ...

    def get_usage_color(self, percent: float) -> str:
        """Return color based on usage percentage."""
        if percent < 50:
            return "green"
        elif percent < 80:
            return "yellow"
        elif percent < 95:
            return "orange1"
        else:
            return "red"

    def create_cpu_panel(self) -> Panel:
        """Create CPU monitoring panel with per-core usage."""
        # Get CPU percentages per core
        cpu_percent = psutil.cpu_percent(interval=0.1, percpu=True)
        cpu_freq = psutil.cpu_freq()

        # Create table for CPU cores
        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("Core", style="cyan", width=6)
        table.add_column("Usage", width=40)
        table.add_column("%", justify="right", style="white", width=6)

        for i, percent in enumerate(cpu_percent):
            color = self.get_usage_color(percent)

            # Create progress bar
            bar_length = 30
            filled = int(bar_length * percent / 100)
            bar = f"[{color}]{'█' * filled}{'░' * (bar_length - filled)}[/{color}]"

            table.add_row(
                f"CPU{i}",
                bar,
                f"{percent:.1f}"
            )

        # Add overall CPU usage
        overall = sum(cpu_percent) / len(cpu_percent)
        color = self.get_usage_color(overall)
        bar_length = 30
        filled = int(bar_length * overall / 100)
        bar = f"[{color}]{'█' * filled}{'░' * (bar_length - filled)}[/{color}]"

        table.add_row(
            "[bold]Total[/bold]",
            bar,
            f"[bold]{overall:.1f}[/bold]"
        )

        # Add frequency info
        freq_text = f"\nFrequency: [green]{cpu_freq.current:.0f} MHz[/green]"

        return Panel(
            table,
            title=f"[bold]CPU Monitor[/bold] {freq_text}",
            border_style="blue"
        )

    def create_memory_panel(self) -> Panel:
        """Create memory monitoring panel."""
        mem = psutil.virtual_memory()
        swap = psutil.swap_memory()

        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("Type", style="cyan", width=8)
        table.add_column("Usage", width=40)
        table.add_column("Used/Total", justify="right", style="white")

        # RAM
        ram_color = self.get_usage_color(mem.percent)
        bar_length = 30
        filled = int(bar_length * mem.percent / 100)
        ram_bar = f"[{ram_color}]{'█' * filled}{'░' * (bar_length - filled)}[/{ram_color}]"

        table.add_row(
            "RAM",
            ram_bar,
            f"{self.format_bytes(mem.used)} / {self.format_bytes(mem.total)} ({mem.percent:.1f}%)"
        )

        # Swap
        if swap.total > 0:
            swap_color = self.get_usage_color(swap.percent)
            filled = int(bar_length * swap.percent / 100)
            swap_bar = f"[{swap_color}]{'█' * filled}{'░' * (bar_length - filled)}[/{swap_color}]"

            table.add_row(
                "Swap",
                swap_bar,
                f"{self.format_bytes(swap.used)} / {self.format_bytes(swap.total)} ({swap.percent:.1f}%)"
            )

        return Panel(table, title="[bold]Memory Monitor[/bold]", border_style="green")

    def create_disk_panel(self) -> Panel:
        """Create disk usage panel."""
        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("Mount", style="cyan", width=15)
        table.add_column("Usage", width=30)
        table.add_column("Used/Total", justify="right", style="white")

        for partition in psutil.disk_partitions():
            try:
                usage = psutil.disk_usage(partition.mountpoint)
                color = self.get_usage_color(usage.percent)

                bar_length = 20
                filled = int(bar_length * usage.percent / 100)
                bar = f"[{color}]{'█' * filled}{'░' * (bar_length - filled)}[/{color}]"

                table.add_row(
                    partition.mountpoint,
                    bar,
                    f"{self.format_bytes(usage.used)} / {self.format_bytes(usage.total)} ({usage.percent:.1f}%)"
                )
            except PermissionError:
                continue

        return Panel(table, title="[bold]Disk Usage[/bold]", border_style="yellow")
```

### Step 3: Real-Time Updates with Live Display

```python
from rich.live import Live
from rich.layout import Layout
import time
from collections import deque

class RealtimeSystemMonitor(SystemMonitor):
    def __init__(self):
        super().__init__()
        self.cpu_history = deque(maxlen=60)  # Last 60 readings
        self.mem_history = deque(maxlen=60)
        self.net_history = deque(maxlen=60)
        self.running = True

    def create_network_panel(self) -> Panel:
        """Create network monitoring panel."""
        net_io = psutil.net_io_counters()

        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("Metric", style="cyan")
        table.add_column("Value", style="green", justify="right")

        table.add_row("Bytes Sent", self.format_bytes(net_io.bytes_sent))
        table.add_row("Bytes Received", self.format_bytes(net_io.bytes_recv))
        table.add_row("Packets Sent", f"{net_io.packets_sent:,}")
        table.add_row("Packets Received", f"{net_io.packets_recv:,}")

        if net_io.errin > 0 or net_io.errout > 0:
            table.add_row("Errors In", f"[red]{net_io.errin}[/red]")
            table.add_row("Errors Out", f"[red]{net_io.errout}[/red]")

        return Panel(table, title="[bold]Network I/O[/bold]", border_style="cyan")

    def create_process_table(self, sort_by: str = "cpu") -> Panel:
        """Create table of top processes."""
        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("PID", style="cyan", width=8)
        table.add_column("Name", style="white", width=25, overflow="ellipsis")
        table.add_column("CPU %", justify="right", style="green", width=8)
        table.add_column("Memory %", justify="right", style="yellow", width=10)
        table.add_column("Status", style="blue", width=10)

        # Get all processes
        processes = []
        for proc in psutil.process_iter(['pid', 'name', 'cpu_percent', 'memory_percent', 'status']):
            try:
                processes.append(proc.info)
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                pass

        # Sort processes
        if sort_by == "cpu":
            processes.sort(key=lambda p: p.get('cpu_percent', 0), reverse=True)
        elif sort_by == "memory":
            processes.sort(key=lambda p: p.get('memory_percent', 0), reverse=True)

        # Add top 15 processes
        for proc in processes[:15]:
            cpu_color = self.get_usage_color(proc.get('cpu_percent', 0))
            mem_color = self.get_usage_color(proc.get('memory_percent', 0))

            table.add_row(
                str(proc.get('pid', 'N/A')),
                proc.get('name', 'Unknown')[:25],
                f"[{cpu_color}]{proc.get('cpu_percent', 0):.1f}[/{cpu_color}]",
                f"[{mem_color}]{proc.get('memory_percent', 0):.1f}[/{mem_color}]",
                proc.get('status', 'unknown')
            )

        return Panel(table, title="[bold]Top Processes[/bold]", border_style="magenta")

    def create_sparkline(self, data: deque, width: int = 40) -> str:
        """Create ASCII sparkline from data."""
        if not data or len(data) < 2:
            return "░" * width

        # Normalize data to 0-8 range
        min_val = min(data)
        max_val = max(data)
        range_val = max_val - min_val if max_val != min_val else 1

        # Sparkline characters
        chars = " ▁▂▃▄▅▆▇█"

        # Sample data to fit width
        step = len(data) / width
        sparkline = ""

        for i in range(width):
            idx = int(i * step)
            if idx < len(data):
                normalized = (data[idx] - min_val) / range_val
                char_idx = int(normalized * (len(chars) - 1))
                sparkline += chars[char_idx]
            else:
                sparkline += " "

        return sparkline

    def create_layout(self) -> Layout:
        """Create the main layout."""
        layout = Layout()

        layout.split_column(
            Layout(name="header", size=3),
            Layout(name="main"),
            Layout(name="footer", size=3)
        )

        layout["main"].split_row(
            Layout(name="left"),
            Layout(name="right")
        )

        layout["left"].split_column(
            Layout(name="cpu"),
            Layout(name="memory"),
        )

        layout["right"].split_column(
            Layout(name="disk"),
            Layout(name="network"),
            Layout(name="processes")
        )

        return layout

    def update_layout(self, layout: Layout):
        """Update all panels in the layout."""
        # Collect current metrics
        cpu_percent = psutil.cpu_percent(interval=0.1)
        mem_percent = psutil.virtual_memory().percent

        self.cpu_history.append(cpu_percent)
        self.mem_history.append(mem_percent)

        # Update panels
        layout["header"].update(self.create_header())
        layout["cpu"].update(self.create_cpu_panel())
        layout["memory"].update(self.create_memory_panel())
        layout["disk"].update(self.create_disk_panel())
        layout["network"].update(self.create_network_panel())
        layout["processes"].update(self.create_process_table(sort_by="cpu"))

        # Footer with sparklines
        cpu_spark = self.create_sparkline(self.cpu_history, 30)
        mem_spark = self.create_sparkline(self.mem_history, 30)

        footer_text = (
            f"CPU History: [{self.get_usage_color(cpu_percent)}]{cpu_spark}[/{self.get_usage_color(cpu_percent)}] | "
            f"MEM History: [{self.get_usage_color(mem_percent)}]{mem_spark}[/{self.get_usage_color(mem_percent)}] | "
            f"[dim]Press Ctrl+C to exit[/dim]"
        )

        layout["footer"].update(Panel(footer_text, style="white on dark_blue"))

    def run(self, refresh_rate: float = 2.0):
        """Run the real-time monitor."""
        layout = self.create_layout()

        with Live(layout, refresh_per_second=4, screen=True) as live:
            try:
                while self.running:
                    self.update_layout(layout)
                    time.sleep(refresh_rate)
            except KeyboardInterrupt:
                self.running = False

# Usage
if __name__ == "__main__":
    monitor = RealtimeSystemMonitor()
    monitor.run(refresh_rate=1.5)
```

### Step 4: Advanced Features - Historical Graphs

```python
from collections import defaultdict
import time

class AdvancedSystemMonitor(RealtimeSystemMonitor):
    def __init__(self):
        super().__init__()
        self.disk_io_last = psutil.disk_io_counters()
        self.net_io_last = psutil.net_io_counters()
        self.last_update = time.time()

    def create_io_panel(self) -> Panel:
        """Create disk I/O panel with rate calculations."""
        current_io = psutil.disk_io_counters()
        current_time = time.time()
        time_delta = current_time - self.last_update

        if time_delta > 0:
            read_rate = (current_io.read_bytes - self.disk_io_last.read_bytes) / time_delta
            write_rate = (current_io.write_bytes - self.disk_io_last.write_bytes) / time_delta
        else:
            read_rate = 0
            write_rate = 0

        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("Metric", style="cyan")
        table.add_column("Total", style="green", justify="right")
        table.add_column("Rate", style="yellow", justify="right")

        table.add_row(
            "Disk Read",
            self.format_bytes(current_io.read_bytes),
            f"{self.format_bytes(read_rate)}/s"
        )
        table.add_row(
            "Disk Write",
            self.format_bytes(current_io.write_bytes),
            f"{self.format_bytes(write_rate)}/s"
        )
        table.add_row(
            "Read Count",
            f"{current_io.read_count:,}",
            ""
        )
        table.add_row(
            "Write Count",
            f"{current_io.write_count:,}",
            ""
        )

        self.disk_io_last = current_io
        self.last_update = current_time

        return Panel(table, title="[bold]Disk I/O[/bold]", border_style="yellow")

    def create_detailed_network_panel(self) -> Panel:
        """Create detailed network panel with per-interface stats."""
        net_stats = psutil.net_if_stats()
        net_io = psutil.net_io_counters(pernic=True)

        table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
        table.add_column("Interface", style="cyan", width=12)
        table.add_column("Status", style="white", width=6)
        table.add_column("Speed", style="green", justify="right", width=10)
        table.add_column("Sent", style="yellow", justify="right", width=12)
        table.add_column("Recv", style="blue", justify="right", width=12)

        for interface, stats in net_stats.items():
            if interface in net_io:
                io = net_io[interface]
                status = "[green]UP[/green]" if stats.isup else "[red]DOWN[/red]"
                speed = f"{stats.speed} Mb/s" if stats.speed > 0 else "N/A"

                table.add_row(
                    interface,
                    status,
                    speed,
                    self.format_bytes(io.bytes_sent),
                    self.format_bytes(io.bytes_recv)
                )

        return Panel(table, title="[bold]Network Interfaces[/bold]", border_style="cyan")

    def create_system_temperatures(self) -> Panel:
        """Create temperature monitoring panel (if available)."""
        try:
            temps = psutil.sensors_temperatures()

            if not temps:
                return Panel("[dim]Temperature sensors not available[/dim]",
                           title="[bold]Temperatures[/bold]")

            table = Table(box=box.SIMPLE, show_header=True, padding=(0, 1))
            table.add_column("Sensor", style="cyan")
            table.add_column("Temperature", style="white", justify="right")
            table.add_column("Status", style="white", justify="center")

            for name, entries in temps.items():
                for entry in entries:
                    temp = entry.current

                    # Color code based on temperature
                    if temp < 50:
                        temp_color = "green"
                        status = "✓"
                    elif temp < 70:
                        temp_color = "yellow"
                        status = "⚠"
                    else:
                        temp_color = "red"
                        status = "⚠"

                    label = entry.label or name
                    table.add_row(
                        label,
                        f"[{temp_color}]{temp:.1f}°C[/{temp_color}]",
                        f"[{temp_color}]{status}[/{temp_color}]"
                    )

            return Panel(table, title="[bold]Temperatures[/bold]", border_style="red")

        except AttributeError:
            return Panel("[dim]Temperature monitoring not supported on this platform[/dim]",
                       title="[bold]Temperatures[/bold]")

# Usage
if __name__ == "__main__":
    monitor = AdvancedSystemMonitor()
    monitor.run(refresh_rate=1.0)
```

## Expected Output

### Real-time Dashboard View
```
┌─────────────────────────────────────────────────────────────────────────┐
│ System Monitor - mycomputer | Linux 5.15.0 | 8 cores (16 logical) | ... │
└─────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────┬──────────────────────────────────────────┐
│ CPU Monitor 2.4 GHz          │ Disk Usage                               │
│ CPU0  ██████████░░░░░  45.2  │ /      ████████████████░░  80.5%        │
│ CPU1  ████████░░░░░░░  35.1  │ /home  ██████░░░░░░░░░░░  30.2%        │
│ CPU2  ███████████░░░░  52.8  │                                          │
│ CPU3  █████░░░░░░░░░░  25.4  │ Network I/O                              │
│ Total ████████░░░░░░░  39.6  │ Sent:     2.4 GB | 125 KB/s             │
├──────────────────────────────┤ Recv:     5.1 GB | 450 KB/s             │
│ Memory Monitor               │                                          │
│ RAM   █████████████░░  65.2% │ Top Processes (CPU)                      │
│       5.2 GB / 8.0 GB        │ PID    Name          CPU%   MEM%         │
│ Swap  ██░░░░░░░░░░░░░  10.1% │ 1234   python        45.2   12.3        │
│       404 MB / 4.0 GB        │ 5678   chrome        32.1   18.7        │
└──────────────────────────────┴──────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────────┐
│ CPU History: ▁▂▃▄▅▆▇█▇▆▅▄▃▂▁ | MEM: ▄▄▅▅▅▆▆▆▅▅▄▄▃▃ | Ctrl+C to exit    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Bonus Challenges

1. **GPU Monitoring**: Add NVIDIA/AMD GPU monitoring (temperature, memory, utilization)
2. **Battery Status**: Show battery level and time remaining for laptops
3. **Process Tree**: Display process hierarchy in tree format
4. **Custom Alerts**: Alert when resources exceed thresholds
5. **Log Export**: Export metrics to CSV/JSON for analysis
6. **Historical Graphs**: Use plotext for terminal-based line graphs
7. **Container Monitoring**: Monitor Docker containers
8. **Service Status**: Show systemd service status
9. **Kill Process**: Interactive process termination
10. **Custom Widgets**: Create custom metric visualizations

## Resources

- [psutil Documentation](https://psutil.readthedocs.io/)
- [Rich Live Display](https://rich.readthedocs.io/en/latest/live.html)
- [Rich Progress Bars](https://rich.readthedocs.io/en/latest/progress.html)
- [plotext - plotting in terminal](https://github.com/piccolomo/plotext)

## Success Criteria

- [ ] Display real-time CPU usage per core
- [ ] Show memory (RAM and swap) utilization
- [ ] Display disk usage and I/O statistics
- [ ] Monitor network interfaces and traffic
- [ ] List top processes by CPU/memory
- [ ] Update display smoothly (no flickering)
- [ ] Color-code resource usage (green/yellow/red)
- [ ] Show historical data with sparklines
- [ ] Handle missing sensors/permissions gracefully
- [ ] Responsive layout for different terminal sizes
- [ ] Clean exit on Ctrl+C

## Testing Checklist

- Test on systems with different CPU counts
- Verify memory calculations match system tools
- Test with multiple disk partitions
- Verify network stats with active transfers
- Test process list sorting
- Stress test with high CPU/memory load
- Verify sparkline visualization
- Test on different platforms (Linux, macOS, Windows)
- Check performance impact of monitoring itself
- Verify clean shutdown without errors
