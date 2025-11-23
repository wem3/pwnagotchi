# CLAUDE.md - Pwnagotchi AI Assistant Guide

This document provides comprehensive guidance for AI assistants working with the Pwnagotchi codebase. It covers architecture, conventions, workflows, and key patterns to follow.

## Project Overview

**Pwnagotchi** is a Raspberry Pi-based WiFi security tool that captures WPA handshakes and PMKIDs for penetration testing and security research. It leverages bettercap for WiFi operations and features a plugin-based architecture for extensibility.

- **Language**: Python 3.11+
- **License**: GPL3
- **Target Hardware**: Raspberry Pi Zero W, Zero 2W, 3, 4, 5
- **Architecture**: 32-bit (Zero W) and 64-bit (Zero 2W+)
- **Current Version**: 2.9.5.4 (see `pwnagotchi/_version.py`)
- **Original Author**: @evilsocket
- **Current Maintainer**: @jayofelony

### Key Characteristics

1. **No AI/ML**: Previous versions had reinforcement learning, but it was removed due to WiFi firmware instability
2. **Autonomous Operation**: Scans, associates, and captures handshakes automatically
3. **Mesh Networking**: Devices can detect and communicate with each other via custom 802.11 information elements
4. **Plugin Architecture**: Event-driven, threaded plugin system for extensibility
5. **Hardware Abstraction**: Supports 50+ display types through unified interface
6. **Multi-language**: 186 locale directories for internationalization

## Repository Structure

```
pwnagotchi/
├── pwnagotchi/              # Main application package
│   ├── __init__.py          # System utilities (uptime, CPU, memory, temp)
│   ├── _version.py          # Version string
│   ├── cli.py               # Entry point (pwnagotchi_cli function)
│   ├── agent.py             # Main agent (507 lines) - orchestrates all operations
│   ├── automata.py          # Mood/state machine
│   ├── bettercap.py         # Bettercap API client
│   ├── defaults.toml        # Default configuration (~2,747 lines)
│   ├── log.py               # Logging and session parsing
│   ├── utils.py             # Config loading, file operations
│   ├── ai/                  # AI components (epoch tracking, rewards)
│   │   ├── epoch.py
│   │   └── reward.py
│   ├── fs/                  # Filesystem management
│   │   ├── __init__.py      # Mount management, zram support
│   │   └── memory.py
│   ├── google/              # Google Drive integration
│   ├── locale/              # i18n (186 language directories)
│   ├── mesh/                # Peer-to-peer mesh networking
│   │   ├── peer.py
│   │   └── utils.py
│   ├── plugins/             # Plugin system
│   │   ├── __init__.py      # Base Plugin class, event system
│   │   └── default/         # 24+ built-in plugins
│   └── ui/                  # User interface
│       ├── components.py    # UI elements (LabeledValue, Text, etc.)
│       ├── display.py       # Display manager
│       ├── faces.py         # ASCII art faces for moods
│       ├── fonts.py         # Font definitions
│       ├── view.py          # UI state management
│       ├── hw/              # Hardware display drivers (50+ types)
│       │   ├── base.py      # Base display class
│       │   ├── waveshare*.py
│       │   ├── inky.py
│       │   └── ...
│       └── web/             # Flask-based web UI
│           ├── server.py
│           ├── static/
│           └── templates/
├── scripts/                 # Utility scripts
│   ├── BR.sh                # Backup/restore
│   ├── *_connection_share.sh  # Network sharing (Linux/macOS/Windows/OpenBSD)
│   └── compile_lang.sh
├── pyproject.toml           # Modern Python packaging config
├── Makefile                 # Build commands (primarily i18n)
├── CONTRIBUTING.md          # Contribution guidelines
├── README.md                # Main documentation
└── LICENSE.md               # GPL3 license
```

## Technology Stack

### Core Dependencies

```python
# Network/Security
scapy                 # Packet manipulation, WiFi operations
bettercap             # WiFi monitoring (REST API on port 8081)
websockets            # Real-time event streaming

# Web Framework
flask                 # Web UI server (port 8080)
flask-cors            # CORS support
flask-wtf             # CSRF protection

# Hardware/GPIO
gpiozero              # GPIO control
rpi-lgpio             # Raspberry Pi GPIO
rpi_hardware_pwm      # Hardware PWM
inky                  # E-ink displays
smbus, smbus2, spidev # I2C/SPI communication

# Data Processing
numpy                 # Numerical computations
PyYAML                # YAML parsing
tomlkit, toml         # TOML configuration

# Cloud Services
pydrive2              # Google Drive integration
tweepy                # Twitter integration
requests              # HTTP client

# Utilities
dbus-python           # D-Bus integration
file-read-backwards   # Log parsing
python-dateutil       # Date/time handling
pycryptodome          # Cryptography
pisugar               # Battery management
```

### External Dependencies

- **bettercap**: Separate service running on port 8081, communicated via HTTP API
- **Linux kernel modules**: zram for compressed RAM filesystems

## Application Architecture

### Entry Point and Flow

**Entry Point**: `pwnagotchi/cli.py:pwnagotchi_cli()`

**Command Line Interface**:
```bash
pwnagotchi [OPTIONS]
  --config, -C          Main config file (default: /etc/pwnagotchi/default.toml)
  --user-config, -U     User config (default: /etc/pwnagotchi/config.toml)
  --manual              Manual mode (stats only, no automation)
  --clear               Clear e-paper display
  --debug               Debug logging
  --version             Print version
  --print-config        Print merged config
  --wizard              Interactive config wizard
  --check-update        Check for updates
  --donate              Donation info

Subcommands:
  plugins               Plugin management
  google                Google Drive commands
```

**Operating Modes**:
1. **Auto Mode** (default): Autonomous scanning and attacking
2. **Manual Mode**: Display statistics without automation

### Application Flow

```python
cli.py (entry point)
  → utils.load_config()         # Merge default.toml + config.toml
  → log.setup_logging()         # Configure logging
  → plugins.load()              # Load enabled plugins
  → Display()                   # Initialize UI
  → Agent()                     # Main agent
    → start()
      → start_monitor_mode()    # Configure WiFi interface
      → start_advertising()     # Broadcast to peers
      → start_event_polling()   # Listen for bettercap events
      → start_session_fetcher() # Update UI periodically
    → Main loop:
      → recon()                 # Scan for access points
      → get_access_points_by_channel()
      → For each AP: associate() + deauth clients
      → next_epoch()            # Update mood/state
```

### Key Classes

**Agent** (`agent.py`): Multiple inheritance pattern
```python
class Agent(Client, Automata, AsyncAdvertiser):
    # Inherits from:
    # - Client: Bettercap API communication
    # - Automata: Mood/state machine
    # - AsyncAdvertiser: Mesh networking
```

**Important Methods**:
- `agent.py:84-183` - `start_monitor_mode()`: WiFi interface setup
- `agent.py:185-250` - `recon()`: Scan for access points
- `agent.py:252-300` - `associate()`: Send association frames
- `agent.py:302-350` - `deauth()`: Deauthenticate clients

## Configuration System

### Configuration Hierarchy

1. **Built-in defaults**: `pwnagotchi/defaults.toml`
2. **System config**: `/etc/pwnagotchi/default.toml` (CLI default)
3. **User overrides**: `/etc/pwnagotchi/config.toml` (CLI default)
4. **Additional configs**: `/etc/pwnagotchi/conf.d/*.toml` (optional)

**Merge Strategy**: User config overrides are deep-merged into defaults

### Key Configuration Sections

```toml
[main]
name = "pwnagotchi"           # Device name
lang = "en"                    # Language code
iface = "wlan0mon"            # WiFi monitor interface
whitelist = []                # Protected networks (SSIDs or BSSIDs)
custom_plugins = "/usr/local/share/pwnagotchi/custom-plugins/"
custom_plugin_repos = [       # GitHub repos for custom plugins
    "https://github.com/user/repo/archive/master.zip"
]

[main.plugins.<plugin_name>]
enabled = true/false
# Plugin-specific options

[personality]
advertise = true              # Broadcast to peer Pwnagotchis
deauth = true                 # Perform deauth attacks
associate = true              # Send association frames
channels = []                 # Restrict to specific channels (empty = all)
min_rssi = -200              # Minimum signal strength filter
recon_time = 30              # Scan duration per channel
ap_ttl = 120                 # AP cache timeout
sta_ttl = 300                # Station cache timeout
# Mood thresholds (epochs)
excited_num_epochs = 10
bored_num_epochs = 15
sad_num_epochs = 25

[ui.display]
enabled = true
type = "waveshare_4"         # Display hardware type
rotation = 180               # Display rotation (0, 90, 180, 270)

[ui.web]
enabled = true
address = "::"               # Listen address ("::" for IPv4/IPv6)
port = 8080
auth = false                 # Enable HTTP basic auth
username = "changeme"
password = "changeme"

[bettercap]
hostname = "127.0.0.1"       # Bettercap API host
port = 8081
scheme = "http"
username = "pwnagotchi"
password = "pwnagotchi"
handshakes = "/home/pi/handshakes"  # PCAP storage directory
silence = [...]              # Events to ignore

[fs.memory]                  # RAM filesystem configuration
enabled = true
[fs.memory.mounts.log]
enabled = true
mount = "/etc/pwnagotchi/log/"
size = "50M"
sync = 60                    # Sync to disk every N seconds
zram = true                  # Use compression
rsync = true                 # Use rsync vs cp
```

## Plugin System

### Architecture

The plugin system is **event-driven** with **threaded execution**. Each plugin runs callbacks in its own thread via `PluginEventQueue`.

### Base Plugin Class

Location: `pwnagotchi/plugins/__init__.py`

```python
import pwnagotchi.plugins as plugins

class MyPlugin(plugins.Plugin):
    __author__ = 'your@email.com'
    __version__ = '1.0.0'
    __license__ = 'GPL3'
    __description__ = 'Description of your plugin'

    def __init__(self):
        self.options = dict()  # Auto-populated from config

    def on_loaded(self):
        # Plugin initialization
        pass
```

**Auto-registration**: Plugins register automatically via `__init_subclass__` hook

### Plugin Lifecycle Hooks

```python
# Initialization
on_loaded()                    # Plugin loaded
on_unload(ui)                  # Before unload
on_config_changed(config)      # Config reloaded
on_ready(agent)                # Agent fully initialized

# UI
on_ui_setup(ui)                # Add UI elements
on_ui_update(ui)               # Update UI elements
on_display_setup(display)      # Hardware display ready
on_webhook(path, request)      # HTTP endpoint: /plugins/<name>/<path>

# Mood/State
on_bored(agent)
on_sad(agent)
on_excited(agent)
on_lonely(agent)
on_grateful(agent)
on_rebooting(agent)
on_wait(agent, t)
on_sleep(agent, t)

# WiFi Events
on_wifi_update(agent, aps)
on_unfiltered_ap_list(agent, aps)
on_association(agent, ap)
on_deauthentication(agent, ap, sta)
on_channel_hop(agent, channel)
on_handshake(agent, filename, ap, sta)
on_free_channel(agent, channel)

# Network Events
on_internet_available(agent)
on_peer_detected(agent, peer)
on_peer_lost(agent, peer)

# Epoch
on_epoch(agent, epoch, epoch_data)

# Dynamic Bettercap Events
on_bcap_<event_name>(agent, data)  # E.g., on_bcap_wifi_client_new()
```

### Built-in Plugins (24+)

- `auto-tune.py` - Automatic channel optimization
- `auto-update.py` - Automatic software updates
- `auto_backup.py` - Automated backups
- `bt-tether.py` - Bluetooth tethering (Android/iOS)
- `cache.py` - Data caching
- `fix_services.py` - Service management
- `gps.py`, `gps_listener.py` - GPS tracking
- `grid.py` - Grid/community reporting
- `gdrivesync.py` - Google Drive sync
- `gpio_buttons.py` - Hardware button support
- `logtail.py` - Log monitoring
- `memtemp.py` - Memory/temperature display
- `ohcapi.py` - OnlineHashCrack integration
- `pisugarx.py` - PiSugar battery support
- `pwncrack.py` - Handshake cracking
- `session-stats.py` - Session statistics
- `webcfg.py` - Web configuration interface
- `webgpsmap.py` - GPS mapping web UI
- `wigle.py` - WiGLE integration
- `wpa-sec.py` - WPA-SEC integration
- `ups_lite.py` - UPS battery support
- `wittypi.py` - WittyPi power management

### Plugin Loading

**Locations**:
- Default: `pwnagotchi/plugins/default/`
- Custom: `/usr/local/share/pwnagotchi/custom-plugins/` (configurable)
- Repos: Downloaded from GitHub via `custom_plugin_repos`

**Configuration**:
```toml
[main.plugins.my_plugin]
enabled = true
option1 = "value1"
option2 = 123
```

**Example Plugin**: See `pwnagotchi/plugins/default/example.py:1-131`

## Development Workflows

### Contributing Guidelines

From `CONTRIBUTING.md`:

1. **Raise an issue first** to propose any code changes
2. **Do not mix features with refactoring** in the same PR
3. **Sign commits** with `git commit -s` (Developer Certificate of Origin)
4. **Follow code style** of the project
5. **Update documentation** if needed
6. **Test thoroughly** - describe testing in PR

### Pull Request Template

Required sections (`.github/PULL_REQUEST_TEMPLATE.md:1-31`):
- Description of changes
- Motivation and context
- How it was tested
- Type of changes (bug fix, feature, breaking change)
- Checklist:
  - Code follows project style
  - Documentation updated
  - Signed commits (`git commit -s`)

### Git Workflow

```bash
# Clone repository
git clone https://github.com/jayofelony/pwnagotchi.git
cd pwnagotchi

# Create feature branch
git checkout -b feature/my-feature

# Make changes
# ...

# Sign commits
git commit -s -m "Add new feature"

# Push and create PR
git push origin feature/my-feature
```

## Coding Conventions

### Python Style

**Observed patterns from codebase**:

1. **Imports**: Standard library → Third-party → Local
   ```python
   import time
   import json
   import os

   import numpy as np
   from flask import Flask

   import pwnagotchi
   import pwnagotchi.utils as utils
   ```

2. **Logging**: Use `logging` module, not print statements
   ```python
   import logging

   logging.info("Starting operation")
   logging.debug("Debug details: %s", variable)
   logging.warning("Warning message")
   logging.error("Error occurred: %s", error)
   ```

3. **Configuration Access**: Via `self._config` or `agent.config()`
   ```python
   iface = self._config['main']['iface']
   enabled = self._config['main']['plugins']['my_plugin']['enabled']
   ```

4. **Error Handling**: Comprehensive try-except blocks
   ```python
   try:
       # Operation
   except SpecificException as e:
       logging.error("Operation failed: %s", e)
   ```

5. **String Formatting**: Mix of % and f-strings (prefer % for logging)
   ```python
   logging.info("Found %d access points", len(aps))
   path = f"/home/{user}/data"
   ```

6. **Constants**: UPPER_CASE at module level
   ```python
   RECOVERY_DATA_FILE = '/root/.pwnagotchi-recovery'
   DEFAULT_TIMEOUT = 30
   ```

7. **Private Methods/Variables**: Single underscore prefix
   ```python
   def _reset_wifi_settings(self):
       self._current_channel = 0
   ```

### File Organization

**Plugin files** should follow this structure:
```python
import logging
import pwnagotchi.plugins as plugins

class PluginName(plugins.Plugin):
    __author__ = 'email@example.com'
    __version__ = '1.0.0'
    __license__ = 'GPL3'
    __description__ = 'Brief description'

    def __init__(self):
        self.options = dict()

    # Lifecycle hooks
    def on_loaded(self):
        pass

    # Event hooks
    def on_ready(self, agent):
        pass
```

**Module docstrings**: Generally minimal, code is self-documenting

**Line length**: No strict limit, but keep reasonable (~100-120 chars)

## Testing and Quality Assurance

### Current State

**No formal test suite exists** - the project relies on:
1. Live testing on Raspberry Pi hardware
2. Community testing via releases
3. Issue templates for bug reporting
4. Comprehensive logging for debugging

### Logging System

**Log files** (when running on device):
- `/etc/pwnagotchi/log/pwnagotchi.log` - Main log
- `/etc/pwnagotchi/log/pwnagotchi-debug.log` - Debug log

**Log rotation**: 10MB max size

**Session parsing**: `log.py` parses logs for statistics

### Manual Testing Approach

When developing features:
1. Test on actual Raspberry Pi hardware if possible
2. Test with various display types
3. Test with different WiFi environments
4. Check logs for errors
5. Verify plugin interactions
6. Test configuration changes

## Build and Deployment

### Package Management

**Modern Python packaging**: `pyproject.toml` with setuptools

**Version**: `pwnagotchi/_version.py` (dynamic loading)

**Build commands**:
```bash
# Language compilation
make compile_langs     # Compile .po to .mo files
make update_langs      # Update translation files

# Development install
pip install -e .

# Production install
pip install .

# Build distribution
python -m build
```

### Installation

**Primary method**: Pre-configured SD card images

**Wiki**: https://github.com/jayofelony/pwnagotchi/wiki

**Manual install**: Via pip on Raspberry Pi OS

### Deployment Architecture

**System services** (systemd):
- `pwnagotchi.service` - Main application
- `bettercap.service` - WiFi monitoring

**Filesystem management**:
- RAM-based filesystem with zram compression
- Automatic sync to persistent storage (reduces SD card wear)
- Mount management via `fs/__init__.py`

**Update mechanism**:
```bash
# CLI
pwnagotchi --check-update

# Plugin (auto-update.py)
# - Checks GitHub API every N hours
# - Downloads and installs updates
# - Restarts service
```

## Key Patterns and Best Practices

### For AI Assistants

1. **Always read files before modifying**: Never propose changes to code you haven't read
2. **Preserve existing patterns**: Match the style and structure of surrounding code
3. **Test on hardware**: Pwnagotchi is hardware-dependent; suggest testing on actual devices
4. **Consider plugin system**: Most features should be implemented as plugins, not core changes
5. **Check configuration**: Many behaviors are configurable via TOML
6. **Respect GPL3 license**: All contributions must be GPL3-compatible
7. **Sign commits**: Use `git commit -s` for Developer Certificate of Origin

### Plugin Development

**Best practices**:
1. Start from `example.py` as template
2. Only implement hooks you need
3. Handle errors gracefully
4. Log important events
5. Make options configurable
6. Test with plugin disabled/enabled
7. Document options in docstring or comments
8. Consider thread safety (hooks run in separate threads)

**Common patterns**:
```python
# Accessing config
enabled = self.options.get('enabled', False)

# Running bettercap commands
agent.run('wifi.recon on')

# Adding UI elements
from pwnagotchi.ui.components import LabeledValue
ui.add_element('my_label', LabeledValue(...))

# Updating UI
ui.set('my_label', 'new value')

# Getting agent state
if agent.is_bored():
    # Do something
```

### Configuration Changes

**Best practices**:
1. Add defaults to `pwnagotchi/defaults.toml`
2. Document options with comments
3. Use hierarchical structure: `[section.subsection]`
4. Provide examples for complex options
5. Use sensible defaults
6. Validate configuration in code

### UI Development

**Display abstraction**:
1. All displays inherit from `ui/hw/base.py:DisplayImpl`
2. Use PIL (Pillow) for rendering
3. Support multiple resolutions
4. Test with different display types
5. Handle rotation (0, 90, 180, 270)

**Web UI**:
1. Flask-based (`ui/web/server.py`)
2. Templates in `ui/web/templates/`
3. Static assets in `ui/web/static/`
4. CSRF protection via flask-wtf
5. Optional HTTP basic auth

## Important Files Reference

### Core Application
- `pwnagotchi/cli.py` - Entry point
- `pwnagotchi/agent.py:1-507` - Main agent logic
- `pwnagotchi/automata.py` - State machine
- `pwnagotchi/bettercap.py` - API client
- `pwnagotchi/utils.py` - Config loading, utilities

### Configuration
- `pwnagotchi/defaults.toml` - Default configuration
- `/etc/pwnagotchi/config.toml` - User configuration (on device)

### Plugin System
- `pwnagotchi/plugins/__init__.py` - Base Plugin class, event system
- `pwnagotchi/plugins/default/example.py:1-131` - Plugin template

### UI System
- `pwnagotchi/ui/view.py` - UI state management
- `pwnagotchi/ui/display.py` - Display manager
- `pwnagotchi/ui/components.py` - UI components
- `pwnagotchi/ui/hw/base.py` - Display driver base class

### Mesh Networking
- `pwnagotchi/mesh/peer.py` - Peer detection and tracking
- `pwnagotchi/mesh/utils.py` - AsyncAdvertiser

### Filesystem
- `pwnagotchi/fs/__init__.py` - Mount management, zram

## Common Tasks

### Adding a New Plugin

1. Create file in `pwnagotchi/plugins/default/my_plugin.py`
2. Inherit from `plugins.Plugin`
3. Implement required hooks
4. Add default config to `pwnagotchi/defaults.toml`:
   ```toml
   [main.plugins.my_plugin]
   enabled = false
   option1 = "default_value"
   ```
5. Test by enabling in config
6. Document in code comments

### Adding a New Display Driver

1. Create file in `pwnagotchi/ui/hw/my_display.py`
2. Inherit from `DisplayImpl` (base.py)
3. Implement required methods:
   - `initialize()` - Setup hardware
   - `render()` - Draw to display
   - `clear()` - Clear display
4. Add to display type mapping
5. Test with `type = "my_display"` in config

### Modifying Core Behavior

1. **Prefer plugins** over core changes
2. If core change needed:
   - Raise issue first
   - Discuss approach with maintainers
   - Keep changes minimal and focused
   - Ensure backward compatibility
   - Update `defaults.toml` if adding config options
3. Test thoroughly across different hardware

### Adding Configuration Options

1. Add to `pwnagotchi/defaults.toml` with comment
2. Access in code via `config['section']['key']`
3. Provide validation if needed
4. Document in PR or wiki

## Security Considerations

**Important**: Pwnagotchi is a WiFi security testing tool

1. **Use responsibly**: Only test networks you own or have authorization to test
2. **Legal compliance**: WiFi attacks may be illegal in your jurisdiction
3. **Ethical use**: This is for penetration testing and security research
4. **Whitelist**: Always configure `main.whitelist` to protect your own networks
5. **Authentication**: Enable web UI auth in untrusted environments
6. **Updates**: Keep software updated (auto-update plugin)

## Resources

- **Wiki**: https://github.com/jayofelony/pwnagotchi/wiki
- **Website**: https://pwnagotchi.org
- **Discord**: https://discord.gg/PGgnzFbz4M
- **Reddit**: https://www.reddit.com/r/pwnagotchi/
- **Issues**: https://github.com/jayofelony/pwnagotchi/issues
- **Releases**: https://github.com/jayofelony/pwnagotchi/releases

## Notes for AI Assistants

### When Working with This Codebase

1. **This is security research software**: Treat it professionally and emphasize responsible use
2. **Hardware-dependent**: Many features require Raspberry Pi and WiFi hardware
3. **No test suite**: Manual testing on hardware is required
4. **Active community**: Check recent issues/PRs for context
5. **Plugin-first approach**: Most features should be plugins
6. **Configuration-driven**: Many behaviors are configurable
7. **GPL3 compliance**: All code must be GPL3-compatible
8. **Sign commits**: Use `git commit -s` for DCO

### Common Pitfalls

1. **Don't assume AI functionality exists**: It was removed in current versions
2. **Bettercap is separate**: It's not part of this codebase, just an API client
3. **Threading complexity**: Plugins run in threads, be careful with shared state
4. **Hardware abstraction**: Display code must work across 50+ display types
5. **Config merging**: User config overrides defaults, handle missing keys gracefully
6. **Filesystem concerns**: RAM filesystems require sync logic for persistence

### Useful Search Patterns

```bash
# Find plugin implementations
grep -r "class.*plugins.Plugin" pwnagotchi/plugins/

# Find bettercap API calls
grep -r "self.run(" pwnagotchi/

# Find configuration usage
grep -r "self._config\[" pwnagotchi/

# Find UI elements
grep -r "ui.add_element" pwnagotchi/

# Find event hooks
grep -r "def on_" pwnagotchi/plugins/
```

---

**Document Version**: 1.0.0
**Last Updated**: 2025-11-23
**Pwnagotchi Version**: 2.9.5.4
**Maintainer**: @jayofelony
