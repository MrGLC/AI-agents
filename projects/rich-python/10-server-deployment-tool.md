# Project 10: Server Deployment Tool with Progress Tracking

## Overview
Build a comprehensive terminal-based deployment tool that automates server deployments with visual progress tracking, health checks, rollback capabilities, and multi-server orchestration - creating a beautiful DevOps dashboard in your terminal.

## Learning Objectives
- Execute remote commands over SSH
- Orchestrate multi-step deployment workflows
- Implement progress tracking for long operations
- Handle errors and rollback mechanisms
- Create deployment pipelines
- Display real-time server status
- Manage deployment configurations

## Difficulty Level
**Advanced** - Requires SSH, subprocess management, error handling, and orchestration

## Technical Stack
- **Rich**: Progress, Live, Table, Tree, Panel, Console
- **paramiko**: SSH connections and remote execution
- **Python stdlib**: subprocess, threading, queue, yaml
- **Optional**: docker (container deployments), ansible (automation)

## Requirements

### Core Features
1. **Multi-server Deployment**: Deploy to multiple servers simultaneously
2. **Progress Tracking**: Visual progress for each deployment step
3. **Health Checks**: Pre/post deployment validation
4. **Rollback**: Automatic or manual rollback on failure
5. **Configuration Management**: YAML-based deployment configs
6. **Logging**: Detailed deployment logs

### Layout Components
1. **Deployment Dashboard**: Overall deployment status
2. **Server Status Panel**: Individual server health
3. **Progress Bars**: Multi-stage deployment progress
4. **Log Viewer**: Real-time deployment logs
5. **Command Output**: SSH command results
6. **Summary Panel**: Deployment statistics

### Interaction Features
- Select deployment environment (dev/staging/prod)
- Choose servers to deploy to
- Confirm before destructive operations
- Monitor deployment in real-time
- Trigger rollback if needed
- View deployment history

## Step-by-Step Implementation

### Step 1: Server Connection and Command Execution

```python
# deployment_tool.py
from dataclasses import dataclass
from typing import List, Dict, Optional, Tuple
from pathlib import Path
import paramiko
from rich.console import Console
from rich.panel import Panel
from rich.table import Table
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn
import yaml
import time
from datetime import datetime
from enum import Enum

class ServerStatus(Enum):
    CONNECTED = "connected"
    DISCONNECTED = "disconnected"
    DEPLOYING = "deploying"
    FAILED = "failed"
    SUCCESS = "success"

@dataclass
class Server:
    name: str
    host: str
    port: int = 22
    user: str = "deploy"
    key_file: Optional[Path] = None
    password: Optional[str] = None
    status: ServerStatus = ServerStatus.DISCONNECTED

@dataclass
class DeploymentStep:
    name: str
    command: str
    description: str = ""
    critical: bool = True  # If True, failure stops deployment
    timeout: int = 300  # Seconds

@dataclass
class DeploymentConfig:
    name: str
    servers: List[Server]
    steps: List[DeploymentStep]
    pre_checks: List[DeploymentStep] = None
    post_checks: List[DeploymentStep] = None
    rollback_steps: List[DeploymentStep] = None

    def __post_init__(self):
        if self.pre_checks is None:
            self.pre_checks = []
        if self.post_checks is None:
            self.post_checks = []
        if self.rollback_steps is None:
            self.rollback_steps = []

class SSHConnection:
    def __init__(self, server: Server):
        self.server = server
        self.client: Optional[paramiko.SSHClient] = None
        self.console = Console()

    def connect(self) -> bool:
        """Establish SSH connection to server."""
        try:
            self.client = paramiko.SSHClient()
            self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

            if self.server.key_file:
                # Use key-based authentication
                self.client.connect(
                    hostname=self.server.host,
                    port=self.server.port,
                    username=self.server.user,
                    key_filename=str(self.server.key_file),
                    timeout=10
                )
            else:
                # Use password authentication
                self.client.connect(
                    hostname=self.server.host,
                    port=self.server.port,
                    username=self.server.user,
                    password=self.server.password,
                    timeout=10
                )

            self.server.status = ServerStatus.CONNECTED
            return True

        except Exception as e:
            self.console.print(f"[red]Connection failed to {self.server.name}: {e}[/red]")
            self.server.status = ServerStatus.FAILED
            return False

    def execute_command(self, command: str, timeout: int = 300) -> Tuple[bool, str, str]:
        """
        Execute command on remote server.
        Returns: (success, stdout, stderr)
        """
        if not self.client:
            return False, "", "Not connected"

        try:
            stdin, stdout, stderr = self.client.exec_command(command, timeout=timeout)

            # Read output
            exit_status = stdout.channel.recv_exit_status()
            stdout_text = stdout.read().decode('utf-8')
            stderr_text = stderr.read().decode('utf-8')

            success = exit_status == 0

            return success, stdout_text, stderr_text

        except Exception as e:
            return False, "", str(e)

    def disconnect(self):
        """Close SSH connection."""
        if self.client:
            self.client.close()
            self.client = None
            self.server.status = ServerStatus.DISCONNECTED

class DeploymentExecutor:
    def __init__(self, config: DeploymentConfig):
        self.config = config
        self.console = Console()
        self.connections: Dict[str, SSHConnection] = {}
        self.deployment_log: List[str] = []

    def log(self, message: str):
        """Add message to deployment log."""
        timestamp = datetime.now().strftime("%H:%M:%S")
        log_entry = f"[{timestamp}] {message}"
        self.deployment_log.append(log_entry)
        self.console.print(f"[dim]{log_entry}[/dim]")

    def connect_servers(self) -> bool:
        """Connect to all servers."""
        self.log("Connecting to servers...")

        all_connected = True

        for server in self.config.servers:
            connection = SSHConnection(server)

            if connection.connect():
                self.connections[server.name] = connection
                self.log(f"✓ Connected to {server.name} ({server.host})")
            else:
                self.log(f"✗ Failed to connect to {server.name}")
                all_connected = False

        return all_connected

    def disconnect_servers(self):
        """Disconnect from all servers."""
        for connection in self.connections.values():
            connection.disconnect()

        self.connections.clear()

    def execute_step_on_server(
        self,
        server_name: str,
        step: DeploymentStep
    ) -> Tuple[bool, str]:
        """Execute a deployment step on a specific server."""
        connection = self.connections.get(server_name)

        if not connection:
            return False, "Not connected"

        self.log(f"[{server_name}] Executing: {step.name}")

        success, stdout, stderr = connection.execute_command(
            step.command,
            timeout=step.timeout
        )

        if success:
            self.log(f"[{server_name}] ✓ {step.name} completed")
            return True, stdout
        else:
            error_msg = stderr or "Command failed"
            self.log(f"[{server_name}] ✗ {step.name} failed: {error_msg}")
            return False, error_msg

    def run_deployment(self) -> bool:
        """Run complete deployment process."""
        self.log(f"Starting deployment: {self.config.name}")

        # Connect to servers
        if not self.connect_servers():
            self.log("Failed to connect to all servers")
            return False

        try:
            # Run pre-checks
            if self.config.pre_checks:
                self.log("Running pre-deployment checks...")

                for step in self.config.pre_checks:
                    for server_name in self.connections.keys():
                        success, output = self.execute_step_on_server(server_name, step)

                        if not success and step.critical:
                            self.log(f"Pre-check failed: {step.name}")
                            return False

            # Run deployment steps
            self.log("Running deployment steps...")

            for step in self.config.steps:
                step_success = True

                for server_name in self.connections.keys():
                    success, output = self.execute_step_on_server(server_name, step)

                    if not success:
                        step_success = False

                        if step.critical:
                            self.log(f"Critical step failed: {step.name}")
                            self.log("Initiating rollback...")
                            self.run_rollback()
                            return False

                if step_success:
                    self.log(f"✓ Step completed on all servers: {step.name}")

            # Run post-checks
            if self.config.post_checks:
                self.log("Running post-deployment checks...")

                for step in self.config.post_checks:
                    for server_name in self.connections.keys():
                        success, output = self.execute_step_on_server(server_name, step)

                        if not success and step.critical:
                            self.log(f"Post-check failed: {step.name}")
                            self.log("Initiating rollback...")
                            self.run_rollback()
                            return False

            self.log("✓ Deployment completed successfully!")
            return True

        finally:
            self.disconnect_servers()

    def run_rollback(self):
        """Execute rollback steps."""
        if not self.config.rollback_steps:
            self.log("No rollback steps defined")
            return

        self.log("Executing rollback...")

        for step in self.config.rollback_steps:
            for server_name in self.connections.keys():
                self.execute_step_on_server(server_name, step)

        self.log("Rollback completed")

# Usage
if __name__ == "__main__":
    # Define a deployment configuration
    config = DeploymentConfig(
        name="Web App Deployment",
        servers=[
            Server(
                name="web-1",
                host="192.168.1.10",
                user="deploy",
                key_file=Path("~/.ssh/deploy_key").expanduser()
            ),
            Server(
                name="web-2",
                host="192.168.1.11",
                user="deploy",
                key_file=Path("~/.ssh/deploy_key").expanduser()
            )
        ],
        pre_checks=[
            DeploymentStep(
                name="Check disk space",
                command="df -h | grep -v tmpfs",
                description="Verify sufficient disk space"
            )
        ],
        steps=[
            DeploymentStep(
                name="Pull latest code",
                command="cd /var/www/app && git pull origin main"
            ),
            DeploymentStep(
                name="Install dependencies",
                command="cd /var/www/app && pip install -r requirements.txt"
            ),
            DeploymentStep(
                name="Run migrations",
                command="cd /var/www/app && python manage.py migrate"
            ),
            DeploymentStep(
                name="Restart application",
                command="sudo systemctl restart webapp"
            )
        ],
        post_checks=[
            DeploymentStep(
                name="Health check",
                command="curl -f http://localhost:8000/health || exit 1"
            )
        ],
        rollback_steps=[
            DeploymentStep(
                name="Revert to previous version",
                command="cd /var/www/app && git checkout HEAD~1"
            ),
            DeploymentStep(
                name="Restart application",
                command="sudo systemctl restart webapp"
            )
        ]
    )

    executor = DeploymentExecutor(config)
    success = executor.run_deployment()

    print(f"Deployment {'succeeded' if success else 'failed'}")
```

### Step 2: Visual Deployment Dashboard

```python
from rich.layout import Layout
from rich.live import Live
from rich.progress import Progress, TaskID
from rich import box
import threading

class DeploymentDashboard:
    def __init__(self, config: DeploymentConfig):
        self.config = config
        self.console = Console()
        self.server_statuses: Dict[str, ServerStatus] = {}
        self.current_step: Optional[str] = None
        self.logs: List[str] = []

    def create_header(self) -> Panel:
        """Create dashboard header."""
        header_text = (
            f"[bold cyan]{self.config.name}[/bold cyan]\n"
            f"Deploying to {len(self.config.servers)} servers"
        )

        return Panel(
            header_text,
            style="white on blue",
            box=box.ROUNDED
        )

    def create_server_status_table(self) -> Table:
        """Create server status table."""
        table = Table(
            title="[bold]Server Status[/bold]",
            show_header=True,
            header_style="bold magenta",
            box=box.ROUNDED
        )

        table.add_column("Server", style="cyan", width=20)
        table.add_column("Host", style="blue", width=20)
        table.add_column("Status", width=15)
        table.add_column("Progress", width=30)

        for server in self.config.servers:
            status = self.server_statuses.get(server.name, ServerStatus.DISCONNECTED)

            # Status indicator
            if status == ServerStatus.CONNECTED:
                status_text = "[green]● Connected[/green]"
            elif status == ServerStatus.DEPLOYING:
                status_text = "[yellow]● Deploying...[/yellow]"
            elif status == ServerStatus.SUCCESS:
                status_text = "[green]✓ Success[/green]"
            elif status == ServerStatus.FAILED:
                status_text = "[red]✗ Failed[/red]"
            else:
                status_text = "[dim]○ Disconnected[/dim]"

            # Progress bar (placeholder)
            progress_bar = "━" * 20

            table.add_row(
                server.name,
                server.host,
                status_text,
                progress_bar
            )

        return table

    def create_deployment_steps_panel(self) -> Panel:
        """Create deployment steps progress panel."""
        lines = []

        all_steps = (
            [(step, "pre") for step in self.config.pre_checks] +
            [(step, "main") for step in self.config.steps] +
            [(step, "post") for step in self.config.post_checks]
        )

        for step, step_type in all_steps:
            icon = "⏳" if step.name == self.current_step else "○"
            color = "yellow" if step.name == self.current_step else "dim"

            step_text = f"[{color}]{icon} {step.name}[/{color}]"

            if step_type == "pre":
                step_text = f"[dim](Pre)[/dim] {step_text}"
            elif step_type == "post":
                step_text = f"[dim](Post)[/dim] {step_text}"

            lines.append(step_text)

        content = "\n".join(lines)

        return Panel(
            content,
            title="[bold]Deployment Steps[/bold]",
            border_style="blue"
        )

    def create_log_panel(self, max_lines: int = 10) -> Panel:
        """Create log viewer panel."""
        recent_logs = self.logs[-max_lines:] if len(self.logs) > max_lines else self.logs

        log_text = "\n".join(recent_logs) if recent_logs else "[dim]No logs yet[/dim]"

        return Panel(
            log_text,
            title=f"[bold]Deployment Logs ({len(self.logs)} entries)[/bold]",
            border_style="green"
        )

    def create_layout(self) -> Layout:
        """Create complete dashboard layout."""
        layout = Layout()

        layout.split_column(
            Layout(name="header", size=4),
            Layout(name="main"),
            Layout(name="footer", size=12)
        )

        layout["main"].split_row(
            Layout(name="servers", ratio=2),
            Layout(name="steps", ratio=1)
        )

        # Update components
        layout["header"].update(self.create_header())
        layout["servers"].update(self.create_server_status_table())
        layout["steps"].update(self.create_deployment_steps_panel())
        layout["footer"].update(self.create_log_panel())

        return layout

class VisualDeploymentExecutor(DeploymentExecutor):
    def __init__(self, config: DeploymentConfig):
        super().__init__(config)
        self.dashboard = DeploymentDashboard(config)

    def log(self, message: str):
        """Override log to update dashboard."""
        timestamp = datetime.now().strftime("%H:%M:%S")
        log_entry = f"[{timestamp}] {message}"
        self.deployment_log.append(log_entry)
        self.dashboard.logs.append(log_entry)

    def run_deployment_with_ui(self) -> bool:
        """Run deployment with live UI updates."""
        # Initialize server statuses
        for server in self.config.servers:
            self.dashboard.server_statuses[server.name] = ServerStatus.DISCONNECTED

        # Run deployment in background thread
        result_container = {'success': False}

        def deploy_thread():
            result_container['success'] = self.run_deployment()

        thread = threading.Thread(target=deploy_thread, daemon=True)

        with Live(
            self.dashboard.create_layout(),
            refresh_per_second=4,
            screen=True,
            console=self.console
        ) as live:
            thread.start()

            while thread.is_alive():
                live.update(self.dashboard.create_layout())
                time.sleep(0.25)

            # Final update
            live.update(self.dashboard.create_layout())

        return result_container['success']

# Usage
if __name__ == "__main__":
    config = DeploymentConfig(
        name="Production Deployment",
        servers=[
            Server(name="web-1", host="192.168.1.10", user="deploy"),
            Server(name="web-2", host="192.168.1.11", user="deploy"),
        ],
        steps=[
            DeploymentStep(name="Backup current version", command="./backup.sh"),
            DeploymentStep(name="Pull latest code", command="git pull"),
            DeploymentStep(name="Install dependencies", command="npm install"),
            DeploymentStep(name="Build application", command="npm run build"),
            DeploymentStep(name="Restart services", command="sudo systemctl restart app"),
        ]
    )

    executor = VisualDeploymentExecutor(config)
    success = executor.run_deployment_with_ui()

    print(f"\nDeployment {'succeeded' if success else 'failed'}")
```

### Step 3: Configuration Management

```python
import yaml

class DeploymentConfigLoader:
    @staticmethod
    def load_from_yaml(file_path: Path) -> DeploymentConfig:
        """Load deployment configuration from YAML file."""
        with open(file_path, 'r') as f:
            data = yaml.safe_load(f)

        # Parse servers
        servers = []
        for server_data in data.get('servers', []):
            key_file = server_data.get('key_file')

            server = Server(
                name=server_data['name'],
                host=server_data['host'],
                port=server_data.get('port', 22),
                user=server_data.get('user', 'deploy'),
                key_file=Path(key_file).expanduser() if key_file else None,
                password=server_data.get('password')
            )
            servers.append(server)

        # Parse steps
        def parse_steps(steps_data):
            steps = []
            for step_data in steps_data:
                step = DeploymentStep(
                    name=step_data['name'],
                    command=step_data['command'],
                    description=step_data.get('description', ''),
                    critical=step_data.get('critical', True),
                    timeout=step_data.get('timeout', 300)
                )
                steps.append(step)
            return steps

        # Create config
        config = DeploymentConfig(
            name=data.get('name', 'Deployment'),
            servers=servers,
            steps=parse_steps(data.get('steps', [])),
            pre_checks=parse_steps(data.get('pre_checks', [])),
            post_checks=parse_steps(data.get('post_checks', [])),
            rollback_steps=parse_steps(data.get('rollback_steps', []))
        )

        return config

    @staticmethod
    def save_to_yaml(config: DeploymentConfig, file_path: Path):
        """Save deployment configuration to YAML file."""
        data = {
            'name': config.name,
            'servers': [
                {
                    'name': server.name,
                    'host': server.host,
                    'port': server.port,
                    'user': server.user,
                    'key_file': str(server.key_file) if server.key_file else None
                }
                for server in config.servers
            ],
            'pre_checks': [
                {
                    'name': step.name,
                    'command': step.command,
                    'description': step.description,
                    'critical': step.critical,
                    'timeout': step.timeout
                }
                for step in config.pre_checks
            ],
            'steps': [
                {
                    'name': step.name,
                    'command': step.command,
                    'description': step.description,
                    'critical': step.critical,
                    'timeout': step.timeout
                }
                for step in config.steps
            ],
            'post_checks': [
                {
                    'name': step.name,
                    'command': step.command,
                    'description': step.description,
                    'critical': step.critical,
                    'timeout': step.timeout
                }
                for step in config.post_checks
            ],
            'rollback_steps': [
                {
                    'name': step.name,
                    'command': step.command,
                    'description': step.description,
                    'critical': step.critical,
                    'timeout': step.timeout
                }
                for step in config.rollback_steps
            ]
        }

        with open(file_path, 'w') as f:
            yaml.dump(data, f, default_flow_style=False, sort_keys=False)

# Example YAML configuration
"""
name: Web Application Deployment

servers:
  - name: web-1
    host: 192.168.1.10
    port: 22
    user: deploy
    key_file: ~/.ssh/deploy_key

  - name: web-2
    host: 192.168.1.11
    port: 22
    user: deploy
    key_file: ~/.ssh/deploy_key

pre_checks:
  - name: Check disk space
    command: df -h / | awk 'NR==2 {if ($5+0 > 90) exit 1}'
    description: Ensure sufficient disk space
    critical: true
    timeout: 30

  - name: Check service status
    command: systemctl is-active webapp
    description: Verify service is running
    critical: false
    timeout: 10

steps:
  - name: Create backup
    command: ./scripts/backup.sh
    description: Backup current deployment
    critical: true
    timeout: 300

  - name: Pull latest code
    command: cd /var/www/app && git pull origin main
    description: Update application code
    critical: true
    timeout: 60

  - name: Install dependencies
    command: cd /var/www/app && pip install -r requirements.txt
    description: Install Python packages
    critical: true
    timeout: 300

  - name: Run database migrations
    command: cd /var/www/app && python manage.py migrate
    description: Apply database changes
    critical: true
    timeout: 120

  - name: Collect static files
    command: cd /var/www/app && python manage.py collectstatic --noinput
    description: Gather static assets
    critical: false
    timeout: 60

  - name: Restart application
    command: sudo systemctl restart webapp
    description: Restart web application service
    critical: true
    timeout: 30

post_checks:
  - name: Health check
    command: curl -f http://localhost:8000/health
    description: Verify application is responding
    critical: true
    timeout: 30

  - name: Check response time
    command: curl -o /dev/null -s -w '%{time_total}' http://localhost:8000 | awk '{if ($1 > 2) exit 1}'
    description: Ensure acceptable performance
    critical: false
    timeout: 10

rollback_steps:
  - name: Restore backup
    command: ./scripts/restore.sh
    description: Restore previous deployment
    critical: true
    timeout: 300

  - name: Restart application
    command: sudo systemctl restart webapp
    description: Restart with restored version
    critical: true
    timeout: 30
"""

# Usage
if __name__ == "__main__":
    # Load from YAML
    config = DeploymentConfigLoader.load_from_yaml(Path("deployment.yml"))

    # Run deployment
    executor = VisualDeploymentExecutor(config)
    success = executor.run_deployment_with_ui()
```

### Step 4: Interactive Deployment Tool

```python
from rich.prompt import Prompt, Confirm

class InteractiveDeploymentTool:
    def __init__(self):
        self.console = Console()
        self.configs: Dict[str, DeploymentConfig] = {}

    def load_configurations(self, config_dir: Path):
        """Load all deployment configurations from directory."""
        config_dir = Path(config_dir)

        if not config_dir.exists():
            self.console.print(f"[yellow]Config directory not found: {config_dir}[/yellow]")
            return

        for yaml_file in config_dir.glob("*.yml"):
            try:
                config = DeploymentConfigLoader.load_from_yaml(yaml_file)
                self.configs[config.name] = config
                self.console.print(f"[green]✓[/green] Loaded: {config.name}")
            except Exception as e:
                self.console.print(f"[red]✗[/red] Failed to load {yaml_file}: {e}")

    def display_menu(self):
        """Display main menu."""
        menu_text = """
[bold cyan]Server Deployment Tool[/bold cyan]

[1] 🚀 Run deployment
[2] 📋 List configurations
[3] 🔍 View configuration details
[4] ✨ Create new configuration
[5] 📊 View deployment history
[0] 🚪 Exit
"""
        self.console.print(Panel(menu_text, border_style="blue"))

    def list_configurations(self):
        """List all available configurations."""
        if not self.configs:
            self.console.print("[yellow]No configurations loaded[/yellow]")
            return

        table = Table(
            title="[bold]Available Configurations[/bold]",
            show_header=True,
            header_style="bold cyan"
        )

        table.add_column("Name", style="cyan")
        table.add_column("Servers", justify="right", style="green")
        table.add_column("Steps", justify="right", style="yellow")

        for config in self.configs.values():
            table.add_row(
                config.name,
                str(len(config.servers)),
                str(len(config.steps))
            )

        self.console.print(table)

    def view_configuration_details(self, config_name: str):
        """Display detailed configuration information."""
        config = self.configs.get(config_name)

        if not config:
            self.console.print(f"[red]Configuration '{config_name}' not found[/red]")
            return

        # Servers
        self.console.print(f"\n[bold]Configuration: {config.name}[/bold]\n")

        servers_table = Table(title="Servers", show_header=True)
        servers_table.add_column("Name", style="cyan")
        servers_table.add_column("Host", style="green")
        servers_table.add_column("User", style="yellow")

        for server in config.servers:
            servers_table.add_row(server.name, server.host, server.user)

        self.console.print(servers_table)

        # Steps
        steps_table = Table(title="Deployment Steps", show_header=True)
        steps_table.add_column("#", style="dim", width=4)
        steps_table.add_column("Name", style="cyan")
        steps_table.add_column("Critical", style="yellow", width=10)

        for i, step in enumerate(config.steps, 1):
            steps_table.add_row(
                str(i),
                step.name,
                "Yes" if step.critical else "No"
            )

        self.console.print(steps_table)

    def run_deployment_interactive(self):
        """Interactively run a deployment."""
        if not self.configs:
            self.console.print("[yellow]No configurations available[/yellow]")
            return

        # Select configuration
        self.list_configurations()

        config_names = list(self.configs.keys())
        config_name = Prompt.ask(
            "\nSelect configuration",
            choices=config_names,
            default=config_names[0]
        )

        config = self.configs[config_name]

        # Show details
        self.view_configuration_details(config_name)

        # Confirm
        if not Confirm.ask(f"\n[bold]Deploy {config_name}?[/bold]", default=False):
            self.console.print("[dim]Deployment cancelled[/dim]")
            return

        # Run deployment
        executor = VisualDeploymentExecutor(config)
        success = executor.run_deployment_with_ui()

        if success:
            self.console.print("\n[bold green]✓ Deployment completed successfully![/bold green]")
        else:
            self.console.print("\n[bold red]✗ Deployment failed[/bold red]")

    def run(self):
        """Run interactive tool."""
        self.console.clear()
        self.console.print("[bold magenta]Server Deployment Tool[/bold magenta]\n")

        # Load configurations
        config_dir = Path("./deployments")
        if config_dir.exists():
            self.load_configurations(config_dir)
        else:
            self.console.print(f"[yellow]Creating config directory: {config_dir}[/yellow]")
            config_dir.mkdir(parents=True, exist_ok=True)

        while True:
            self.display_menu()

            choice = Prompt.ask("Select option", choices=["0","1","2","3","4","5"], default="1")

            self.console.print()

            if choice == "0":
                self.console.print("[green]Goodbye![/green]")
                break

            elif choice == "1":
                self.run_deployment_interactive()

            elif choice == "2":
                self.list_configurations()

            elif choice == "3":
                config_name = Prompt.ask("Configuration name")
                self.view_configuration_details(config_name)

            elif choice == "4":
                self.console.print("[yellow]Feature coming soon![/yellow]")

            elif choice == "5":
                self.console.print("[yellow]Feature coming soon![/yellow]")

            # Pause
            if choice != "0":
                Prompt.ask("\n[dim]Press Enter to continue[/dim]", default="")
                self.console.clear()

# Usage
if __name__ == "__main__":
    tool = InteractiveDeploymentTool()
    tool.run()
```

## Expected Output

### Deployment Dashboard
```
╭──────────────────────────────────────────────────────────╮
│            Production Deployment                         │
│            Deploying to 3 servers                        │
╰──────────────────────────────────────────────────────────╯
╭─ Server Status ──────────────╮╭─ Deployment Steps ──────╮
│ Server  Host        Status   ││ ○ Backup current version│
│ web-1   10.0.1.10   ● Deploy ││ ○ Pull latest code      │
│ web-2   10.0.1.11   ● Deploy ││ ⏳ Install dependencies │
│ web-3   10.0.1.12   ● Deploy ││ ○ Build application     │
╰──────────────────────────────╯│ ○ Restart services      │
╭─ Deployment Logs ────────────╰─────────────────────────╯
│ [10:30:45] ✓ Connected to web-1 (10.0.1.10)             │
│ [10:30:46] ✓ Connected to web-2 (10.0.1.11)             │
│ [10:30:47] ✓ Connected to web-3 (10.0.1.12)             │
│ [10:30:48] [web-1] Executing: Pull latest code          │
│ [10:30:50] [web-1] ✓ Pull latest code completed         │
│ [10:30:51] [web-2] Executing: Pull latest code          │
╰──────────────────────────────────────────────────────────╯
```

## Bonus Challenges

1. **Docker Deployment**: Deploy containerized applications
2. **Kubernetes Integration**: Deploy to K8s clusters
3. **Blue-Green Deployment**: Zero-downtime deployments
4. **Canary Releases**: Gradual rollout strategies
5. **Database Migrations**: Schema change management
6. **Load Balancer Integration**: Update LB during deployment
7. **Slack/Discord Notifications**: Send deployment alerts
8. **Deployment Approvals**: Multi-stage approval workflow
9. **Environment Variables**: Secure config management
10. **Metrics Collection**: Deployment performance tracking

## Resources

- [paramiko Documentation](https://www.paramiko.org/)
- [PyYAML Documentation](https://pyyaml.org/)
- [Rich Progress Bars](https://rich.readthedocs.io/en/latest/progress.html)
- [Fabric - Pythonic SSH](https://www.fabfile.org/)

## Success Criteria

- [ ] Connect to remote servers via SSH
- [ ] Execute commands on multiple servers
- [ ] Display real-time deployment progress
- [ ] Handle deployment failures gracefully
- [ ] Implement rollback functionality
- [ ] Load configurations from YAML
- [ ] Show server status in real-time
- [ ] Log all deployment activities
- [ ] Run pre and post deployment checks
- [ ] Support multi-server orchestration

## Testing Checklist

- Test SSH connection with key-based auth
- Test SSH connection with password auth
- Verify command execution and output capture
- Test deployment with all steps succeeding
- Test deployment with step failure
- Verify rollback execution
- Test with multiple servers
- Check configuration loading from YAML
- Test timeout handling
- Verify log capture and display
- Test with network interruptions
- Check graceful error handling
