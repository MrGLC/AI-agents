# Project 8: Terminal-Based Music Player Interface

## Overview
Build a beautiful terminal music player interface with playlist management, audio visualization, playback controls, and metadata display - creating a TUI alternative to GUI music players like iTunes or Spotify.

## Learning Objectives
- Integrate audio playback libraries
- Create real-time progress displays
- Design media player controls
- Parse audio metadata (ID3 tags)
- Build playlist management systems
- Implement audio visualization in terminal
- Handle keyboard input for media control

## Difficulty Level
**Advanced** - Requires audio library integration, real-time updates, and complex UI state management

## Technical Stack
- **Rich**: Live, Progress, Panel, Table, Layout, Text
- **pygame.mixer**: Audio playback
- **mutagen**: Audio metadata reading
- **Python stdlib**: pathlib, threading, queue
- **Optional**: pynput (keyboard controls), numpy (visualization)

## Requirements

### Core Features
1. **Playback Control**: Play, pause, stop, next, previous
2. **Playlist Management**: Create, edit, shuffle, repeat
3. **Metadata Display**: Song title, artist, album, artwork (ASCII)
4. **Progress Tracking**: Real-time playback progress
5. **Volume Control**: Adjust volume with visual feedback
6. **Search/Filter**: Find songs in library

### Layout Components
1. **Now Playing Panel**: Current song info and artwork
2. **Progress Bar**: Time elapsed and remaining
3. **Playlist View**: Song queue with current track highlighted
4. **Controls Bar**: Play/pause, next, previous buttons
5. **Volume Indicator**: Visual volume level
6. **Status Bar**: Current mode, shuffle, repeat status

### Interaction Features
- Keyboard shortcuts (space, arrows, +/-)
- Navigate playlist with arrow keys
- Seek within track
- Toggle shuffle and repeat
- Add/remove songs from playlist
- Save and load playlists

## Step-by-Step Implementation

### Step 1: Audio Player Core

```python
# music_player.py
from pathlib import Path
from typing import List, Optional
from dataclasses import dataclass
from enum import Enum
import pygame
from mutagen import File as MutagenFile
from mutagen.id3 import ID3
from rich.console import Console
import time
import threading

class PlaybackState(Enum):
    PLAYING = "playing"
    PAUSED = "paused"
    STOPPED = "stopped"

class RepeatMode(Enum):
    OFF = "off"
    ONE = "one"
    ALL = "all"

@dataclass
class Track:
    path: Path
    title: str
    artist: str = "Unknown Artist"
    album: str = "Unknown Album"
    duration: float = 0.0
    track_number: int = 0
    year: str = ""
    genre: str = ""

    def __str__(self):
        return f"{self.artist} - {self.title}"

class AudioPlayer:
    def __init__(self):
        pygame.mixer.init()
        self.current_track: Optional[Track] = None
        self.state = PlaybackState.STOPPED
        self.volume = 0.7
        self.position = 0.0
        self.console = Console()

        pygame.mixer.music.set_volume(self.volume)

    def load_track(self, track: Track) -> bool:
        """Load a track for playback."""
        try:
            pygame.mixer.music.load(str(track.path))
            self.current_track = track
            self.position = 0.0
            return True
        except Exception as e:
            self.console.print(f"[red]Error loading track: {e}[/red]")
            return False

    def play(self):
        """Start or resume playback."""
        if self.state == PlaybackState.PAUSED:
            pygame.mixer.music.unpause()
        else:
            pygame.mixer.music.play()

        self.state = PlaybackState.PLAYING

    def pause(self):
        """Pause playback."""
        if self.state == PlaybackState.PLAYING:
            pygame.mixer.music.pause()
            self.state = PlaybackState.PAUSED

    def stop(self):
        """Stop playback."""
        pygame.mixer.music.stop()
        self.state = PlaybackState.STOPPED
        self.position = 0.0

    def set_volume(self, volume: float):
        """Set volume (0.0 to 1.0)."""
        self.volume = max(0.0, min(1.0, volume))
        pygame.mixer.music.set_volume(self.volume)

    def get_position(self) -> float:
        """Get current playback position in seconds."""
        if self.state == PlaybackState.PLAYING:
            # pygame.mixer.music.get_pos() returns milliseconds
            return pygame.mixer.music.get_pos() / 1000.0
        return self.position

    def is_playing(self) -> bool:
        """Check if currently playing."""
        return self.state == PlaybackState.PLAYING and pygame.mixer.music.get_busy()

class MusicLibrary:
    def __init__(self):
        self.tracks: List[Track] = []
        self.console = Console()

    def scan_directory(self, directory: Path, recursive: bool = True):
        """Scan directory for audio files."""
        audio_extensions = {'.mp3', '.wav', '.ogg', '.flac', '.m4a'}

        if recursive:
            files = directory.rglob('*')
        else:
            files = directory.glob('*')

        for file_path in files:
            if file_path.suffix.lower() in audio_extensions:
                track = self.load_track_metadata(file_path)
                if track:
                    self.tracks.append(track)

        self.console.print(f"[green]Loaded {len(self.tracks)} tracks[/green]")

    def load_track_metadata(self, file_path: Path) -> Optional[Track]:
        """Load track metadata using mutagen."""
        try:
            audio = MutagenFile(str(file_path))

            if audio is None:
                return None

            # Extract metadata
            title = file_path.stem  # Default to filename
            artist = "Unknown Artist"
            album = "Unknown Album"
            duration = 0.0
            track_number = 0
            year = ""
            genre = ""

            # Get duration
            if hasattr(audio.info, 'length'):
                duration = audio.info.length

            # Try to get ID3 tags (for MP3)
            if hasattr(audio, 'tags') and audio.tags:
                tags = audio.tags

                # Title
                if 'TIT2' in tags:
                    title = str(tags['TIT2'])
                elif 'title' in tags:
                    title = str(tags['title'][0])

                # Artist
                if 'TPE1' in tags:
                    artist = str(tags['TPE1'])
                elif 'artist' in tags:
                    artist = str(tags['artist'][0])

                # Album
                if 'TALB' in tags:
                    album = str(tags['TALB'])
                elif 'album' in tags:
                    album = str(tags['album'][0])

                # Track number
                if 'TRCK' in tags:
                    track_str = str(tags['TRCK'])
                    track_number = int(track_str.split('/')[0]) if '/' in track_str else int(track_str)
                elif 'tracknumber' in tags:
                    track_number = int(tags['tracknumber'][0])

                # Year
                if 'TDRC' in tags:
                    year = str(tags['TDRC'])
                elif 'date' in tags:
                    year = str(tags['date'][0])

                # Genre
                if 'TCON' in tags:
                    genre = str(tags['TCON'])
                elif 'genre' in tags:
                    genre = str(tags['genre'][0])

            return Track(
                path=file_path,
                title=title,
                artist=artist,
                album=album,
                duration=duration,
                track_number=track_number,
                year=year,
                genre=genre
            )

        except Exception as e:
            self.console.print(f"[red]Error loading {file_path}: {e}[/red]")
            return None

# Usage
if __name__ == "__main__":
    # Initialize player
    player = AudioPlayer()

    # Scan library
    library = MusicLibrary()
    library.scan_directory(Path("~/Music").expanduser())

    # Play first track
    if library.tracks:
        track = library.tracks[0]
        if player.load_track(track):
            player.play()
            print(f"Now playing: {track}")
```

### Step 2: Playlist Management

```python
import json
import random

class Playlist:
    def __init__(self, name: str = "Default"):
        self.name = name
        self.tracks: List[Track] = []
        self.current_index = 0
        self.shuffle_enabled = False
        self.repeat_mode = RepeatMode.OFF
        self.shuffle_order: List[int] = []

    def add_track(self, track: Track):
        """Add track to playlist."""
        self.tracks.append(track)
        self._update_shuffle_order()

    def remove_track(self, index: int):
        """Remove track from playlist."""
        if 0 <= index < len(self.tracks):
            self.tracks.pop(index)
            self._update_shuffle_order()

            # Adjust current index if needed
            if self.current_index >= len(self.tracks) and self.tracks:
                self.current_index = len(self.tracks) - 1

    def clear(self):
        """Clear all tracks."""
        self.tracks.clear()
        self.current_index = 0
        self.shuffle_order.clear()

    def get_current_track(self) -> Optional[Track]:
        """Get currently selected track."""
        if not self.tracks:
            return None

        if self.shuffle_enabled and self.shuffle_order:
            actual_index = self.shuffle_order[self.current_index]
            return self.tracks[actual_index]
        else:
            return self.tracks[self.current_index]

    def next_track(self) -> Optional[Track]:
        """Move to next track."""
        if not self.tracks:
            return None

        if self.repeat_mode == RepeatMode.ONE:
            # Stay on current track
            return self.get_current_track()

        self.current_index += 1

        if self.current_index >= len(self.tracks):
            if self.repeat_mode == RepeatMode.ALL:
                self.current_index = 0
            else:
                self.current_index = len(self.tracks) - 1
                return None

        return self.get_current_track()

    def previous_track(self) -> Optional[Track]:
        """Move to previous track."""
        if not self.tracks:
            return None

        self.current_index -= 1

        if self.current_index < 0:
            if self.repeat_mode == RepeatMode.ALL:
                self.current_index = len(self.tracks) - 1
            else:
                self.current_index = 0

        return self.get_current_track()

    def toggle_shuffle(self):
        """Toggle shuffle mode."""
        self.shuffle_enabled = not self.shuffle_enabled
        if self.shuffle_enabled:
            self._update_shuffle_order()

    def _update_shuffle_order(self):
        """Update shuffle order."""
        if self.shuffle_enabled and self.tracks:
            self.shuffle_order = list(range(len(self.tracks)))
            random.shuffle(self.shuffle_order)

    def set_repeat_mode(self, mode: RepeatMode):
        """Set repeat mode."""
        self.repeat_mode = mode

    def cycle_repeat_mode(self):
        """Cycle through repeat modes."""
        modes = [RepeatMode.OFF, RepeatMode.ONE, RepeatMode.ALL]
        current_idx = modes.index(self.repeat_mode)
        next_idx = (current_idx + 1) % len(modes)
        self.repeat_mode = modes[next_idx]

    def save_to_file(self, file_path: Path):
        """Save playlist to JSON file."""
        data = {
            'name': self.name,
            'tracks': [
                {
                    'path': str(track.path),
                    'title': track.title,
                    'artist': track.artist,
                    'album': track.album,
                    'duration': track.duration
                }
                for track in self.tracks
            ]
        }

        with open(file_path, 'w') as f:
            json.dump(data, f, indent=2)

    @classmethod
    def load_from_file(cls, file_path: Path) -> 'Playlist':
        """Load playlist from JSON file."""
        with open(file_path, 'r') as f:
            data = json.load(f)

        playlist = cls(name=data['name'])

        for track_data in data['tracks']:
            track = Track(
                path=Path(track_data['path']),
                title=track_data['title'],
                artist=track_data['artist'],
                album=track_data['album'],
                duration=track_data['duration']
            )
            playlist.add_track(track)

        return playlist

# Usage
if __name__ == "__main__":
    playlist = Playlist("My Favorites")

    # Add tracks
    library = MusicLibrary()
    library.scan_directory(Path("~/Music").expanduser())

    for track in library.tracks[:10]:
        playlist.add_track(track)

    # Enable shuffle
    playlist.toggle_shuffle()

    # Set repeat mode
    playlist.set_repeat_mode(RepeatMode.ALL)
```

### Step 3: Player Interface Display

```python
from rich.layout import Layout
from rich.panel import Panel
from rich.table import Table
from rich.progress import Progress, BarColumn, TextColumn, TimeRemainingColumn
from rich.text import Text
from rich import box

class MusicPlayerUI:
    def __init__(self, player: AudioPlayer, playlist: Playlist):
        self.player = player
        self.playlist = playlist
        self.console = Console()

    def format_time(self, seconds: float) -> str:
        """Format time as MM:SS."""
        minutes = int(seconds // 60)
        secs = int(seconds % 60)
        return f"{minutes:02d}:{secs:02d}"

    def create_now_playing_panel(self) -> Panel:
        """Create now playing panel."""
        track = self.player.current_track

        if not track:
            content = Text("No track loaded", style="dim", justify="center")
            return Panel(content, title="[bold]Now Playing[/bold]", border_style="blue")

        # Create content
        lines = []
        lines.append(Text(track.title, style="bold cyan", justify="center"))
        lines.append(Text(track.artist, style="green", justify="center"))
        lines.append(Text(track.album, style="yellow", justify="center"))

        if track.year:
            lines.append(Text(f"({track.year})", style="dim", justify="center"))

        if track.genre:
            lines.append(Text(f"Genre: {track.genre}", style="blue", justify="center"))

        content = Text("\n").join(lines)

        return Panel(
            content,
            title="[bold]♪ Now Playing ♪[/bold]",
            border_style="cyan",
            padding=(1, 2)
        )

    def create_progress_bar(self) -> Panel:
        """Create playback progress bar."""
        track = self.player.current_track

        if not track or track.duration == 0:
            return Panel("[dim]No track loaded[/dim]", title="Progress", border_style="dim")

        position = self.player.get_position()
        duration = track.duration

        # Create progress visualization
        progress_width = 50
        progress_percent = position / duration if duration > 0 else 0
        filled = int(progress_width * progress_percent)

        bar = "━" * filled + "○" + "─" * (progress_width - filled - 1)

        # State indicator
        if self.player.state == PlaybackState.PLAYING:
            state_icon = "▶"
            state_color = "green"
        elif self.player.state == PlaybackState.PAUSED:
            state_icon = "⏸"
            state_color = "yellow"
        else:
            state_icon = "⏹"
            state_color = "red"

        # Time display
        time_text = f"{self.format_time(position)} / {self.format_time(duration)}"

        content = Text()
        content.append(f"{state_icon} ", style=f"bold {state_color}")
        content.append(bar, style="cyan")
        content.append(f"\n{time_text}", style="white", justify="center")

        return Panel(content, border_style="blue", padding=(0, 2))

    def create_playlist_table(self, visible_range: int = 10) -> Panel:
        """Create playlist table."""
        if not self.playlist.tracks:
            return Panel("[dim]Playlist is empty[/dim]", title="Playlist", border_style="dim")

        table = Table(
            show_header=True,
            header_style="bold magenta",
            box=box.SIMPLE,
            padding=(0, 1)
        )

        table.add_column("#", style="dim", width=4)
        table.add_column("Title", style="cyan", overflow="ellipsis", width=30)
        table.add_column("Artist", style="green", overflow="ellipsis", width=20)
        table.add_column("Duration", style="yellow", justify="right", width=8)

        # Calculate visible range around current track
        current = self.playlist.current_index
        start = max(0, current - visible_range // 2)
        end = min(len(self.playlist.tracks), start + visible_range)

        for i in range(start, end):
            track = self.playlist.tracks[i]
            is_current = (i == current)

            # Highlight current track
            number = f"▶ {i+1}" if is_current else str(i+1)
            title = f"[bold]{track.title}[/bold]" if is_current else track.title
            artist = f"[bold]{track.artist}[/bold]" if is_current else track.artist

            table.add_row(
                number,
                title,
                artist,
                self.format_time(track.duration)
            )

        # Show shuffle and repeat status
        status_text = ""
        if self.playlist.shuffle_enabled:
            status_text += "🔀 Shuffle "
        if self.playlist.repeat_mode == RepeatMode.ONE:
            status_text += "🔂 Repeat One"
        elif self.playlist.repeat_mode == RepeatMode.ALL:
            status_text += "🔁 Repeat All"

        title = f"[bold]Playlist ({len(self.playlist.tracks)} tracks)[/bold]"
        if status_text:
            title += f" - {status_text}"

        return Panel(table, title=title, border_style="magenta")

    def create_controls_panel(self) -> Panel:
        """Create controls panel."""
        controls = Text()

        controls.append("Controls: ", style="bold cyan")
        controls.append("Space", style="bold green")
        controls.append(" Play/Pause | ", style="dim")
        controls.append("→", style="bold green")
        controls.append(" Next | ", style="dim")
        controls.append("←", style="bold green")
        controls.append(" Previous | ", style="dim")
        controls.append("+/-", style="bold green")
        controls.append(" Volume | ", style="dim")
        controls.append("S", style="bold green")
        controls.append(" Shuffle | ", style="dim")
        controls.append("R", style="bold green")
        controls.append(" Repeat | ", style="dim")
        controls.append("Q", style="bold green")
        controls.append(" Quit", style="dim")

        return Panel(controls, border_style="white")

    def create_volume_panel(self) -> Panel:
        """Create volume indicator."""
        volume_percent = int(self.player.volume * 100)

        # Volume bar
        bar_width = 20
        filled = int(bar_width * self.player.volume)
        bar = "█" * filled + "░" * (bar_width - filled)

        # Volume icon
        if self.player.volume == 0:
            icon = "🔇"
        elif self.player.volume < 0.3:
            icon = "🔈"
        elif self.player.volume < 0.7:
            icon = "🔉"
        else:
            icon = "🔊"

        content = Text()
        content.append(f"{icon} ", style="white")
        content.append(bar, style="green")
        content.append(f" {volume_percent}%", style="white")

        return Panel(content, title="Volume", border_style="green", padding=(0, 1))

    def create_layout(self) -> Layout:
        """Create complete player layout."""
        layout = Layout()

        layout.split_column(
            Layout(name="header", size=3),
            Layout(name="main"),
            Layout(name="footer", size=3)
        )

        layout["main"].split_row(
            Layout(name="left", ratio=2),
            Layout(name="right", ratio=3)
        )

        layout["left"].split_column(
            Layout(name="now_playing"),
            Layout(name="volume", size=3)
        )

        # Update panels
        layout["header"].update(
            Panel("[bold magenta]♫ Terminal Music Player ♫[/bold magenta]", style="white on blue")
        )

        layout["now_playing"].update(self.create_now_playing_panel())
        layout["volume"].update(self.create_volume_panel())
        layout["right"].split_column(
            Layout(name="progress", size=5),
            Layout(name="playlist")
        )
        layout["progress"].update(self.create_progress_bar())
        layout["playlist"].update(self.create_playlist_table())
        layout["footer"].update(self.create_controls_panel())

        return layout

# Usage
if __name__ == "__main__":
    player = AudioPlayer()
    playlist = Playlist("My Music")

    # Add some tracks
    library = MusicLibrary()
    library.scan_directory(Path("~/Music").expanduser())

    for track in library.tracks[:20]:
        playlist.add_track(track)

    # Create UI
    ui = MusicPlayerUI(player, playlist)
    console = Console()
    console.print(ui.create_layout())
```

### Step 4: Interactive Player with Keyboard Controls

```python
from rich.live import Live
from pynput import keyboard
import threading

class InteractiveMusicPlayer:
    def __init__(self, library: MusicLibrary):
        self.player = AudioPlayer()
        self.playlist = Playlist("Current")
        self.library = library
        self.ui = MusicPlayerUI(self.player, self.playlist)
        self.console = Console()
        self.running = True

        # Add all library tracks to playlist
        for track in library.tracks:
            self.playlist.add_track(track)

    def on_key_press(self, key):
        """Handle keyboard input."""
        try:
            if hasattr(key, 'char'):
                if key.char == ' ':
                    # Play/Pause
                    if self.player.state == PlaybackState.PLAYING:
                        self.player.pause()
                    else:
                        if self.player.current_track:
                            self.player.play()
                        else:
                            self.play_current_track()

                elif key.char == 'q':
                    # Quit
                    self.running = False

                elif key.char == 's':
                    # Toggle shuffle
                    self.playlist.toggle_shuffle()

                elif key.char == 'r':
                    # Cycle repeat mode
                    self.playlist.cycle_repeat_mode()

                elif key.char == '+' or key.char == '=':
                    # Volume up
                    self.player.set_volume(self.player.volume + 0.1)

                elif key.char == '-':
                    # Volume down
                    self.player.set_volume(self.player.volume - 0.1)

            elif key == keyboard.Key.right:
                # Next track
                self.next_track()

            elif key == keyboard.Key.left:
                # Previous track
                self.previous_track()

            elif key == keyboard.Key.up:
                # Move up in playlist
                if self.playlist.current_index > 0:
                    self.playlist.current_index -= 1

            elif key == keyboard.Key.down:
                # Move down in playlist
                if self.playlist.current_index < len(self.playlist.tracks) - 1:
                    self.playlist.current_index += 1

            elif key == keyboard.Key.enter:
                # Play selected track
                self.play_current_track()

        except AttributeError:
            pass

    def play_current_track(self):
        """Play the currently selected track."""
        track = self.playlist.get_current_track()
        if track:
            self.player.stop()
            if self.player.load_track(track):
                self.player.play()

    def next_track(self):
        """Play next track."""
        track = self.playlist.next_track()
        if track:
            self.player.stop()
            if self.player.load_track(track):
                self.player.play()

    def previous_track(self):
        """Play previous track."""
        track = self.playlist.previous_track()
        if track:
            self.player.stop()
            if self.player.load_track(track):
                self.player.play()

    def check_track_end(self):
        """Check if current track has ended and play next."""
        if self.player.current_track and not self.player.is_playing():
            if self.player.state == PlaybackState.PLAYING:
                # Track ended, play next
                self.next_track()

    def run(self):
        """Run interactive player."""
        self.console.clear()

        # Start keyboard listener
        listener = keyboard.Listener(on_press=self.on_key_press)
        listener.start()

        # Play first track
        if self.playlist.tracks:
            self.play_current_track()

        # Main loop with live display
        with Live(
            self.ui.create_layout(),
            refresh_per_second=4,
            screen=True,
            console=self.console
        ) as live:
            while self.running:
                # Update display
                live.update(self.ui.create_layout())

                # Check if track ended
                self.check_track_end()

                time.sleep(0.25)

        # Cleanup
        listener.stop()
        self.player.stop()
        pygame.mixer.quit()

# Usage
if __name__ == "__main__":
    # Scan music library
    library = MusicLibrary()
    music_dir = Path("~/Music").expanduser()

    if music_dir.exists():
        console = Console()
        with console.status("[bold green]Scanning music library..."):
            library.scan_directory(music_dir, recursive=True)

        # Run player
        if library.tracks:
            player = InteractiveMusicPlayer(library)
            player.run()
        else:
            console.print("[yellow]No music files found![/yellow]")
    else:
        print(f"Music directory not found: {music_dir}")
```

## Expected Output

### Player Interface
```
╭──────────────────────────────────────────────────────────╮
│               ♫ Terminal Music Player ♫                  │
╰──────────────────────────────────────────────────────────╯
╭─────────────────────────┬────────────────────────────────╮
│ ♪ Now Playing ♪         │ Progress                       │
│                         │ ▶ ━━━━━━━━━○────────────       │
│   Bohemian Rhapsody     │   02:34 / 05:55                │
│   Queen                 │                                │
│   A Night at the Opera  │ Playlist (156 tracks) - 🔀     │
│   (1975)                │ #  Title              Artist   │
│                         │ 45 Another One...     Queen    │
│ 🔊 ████████████████░░   │ 46 We Will Rock You   Queen    │
│    80%                  │▶47 Bohemian Rhapsody  Queen    │
╰─────────────────────────┴─ 48 Love of My Life   Queen   ─╯
Controls: Space Play/Pause | → Next | ← Previous | +/- Volume
```

## Bonus Challenges

1. **Equalizer**: ASCII-based audio visualizer/spectrum analyzer
2. **Lyrics Display**: Show synchronized lyrics
3. **Album Art**: Convert and display album art as ASCII
4. **Scrobbling**: Last.fm integration
5. **Smart Playlists**: Auto-generate playlists by genre/mood
6. **Radio Mode**: Internet radio stream support
7. **Sleep Timer**: Auto-stop after duration
8. **Queue System**: Temporary queue vs permanent playlist
9. **Crossfade**: Smooth transitions between tracks
10. **Mini Mode**: Compact player view

## Resources

- [pygame.mixer Documentation](https://www.pygame.org/docs/ref/mixer.html)
- [mutagen - Audio Metadata](https://mutagen.readthedocs.io/)
- [pynput - Keyboard Input](https://pynput.readthedocs.io/)
- [Rich Live Display](https://rich.readthedocs.io/en/latest/live.html)

## Success Criteria

- [ ] Load and parse audio files with metadata
- [ ] Play, pause, stop audio playback
- [ ] Display now playing information
- [ ] Show real-time progress bar
- [ ] Navigate playlist with arrow keys
- [ ] Next/previous track functionality
- [ ] Volume control with visual feedback
- [ ] Shuffle and repeat modes
- [ ] Save and load playlists
- [ ] Handle track end and auto-advance
- [ ] Keyboard shortcuts working

## Testing Checklist

- Test with various audio formats (MP3, WAV, OGG, FLAC)
- Verify metadata parsing (ID3 tags)
- Test with files missing metadata
- Check playback control responsiveness
- Test shuffle randomization
- Verify repeat modes (off, one, all)
- Test volume control limits
- Check playlist navigation
- Test with empty playlist
- Verify keyboard shortcuts
- Test auto-advance to next track
- Check UI update performance
