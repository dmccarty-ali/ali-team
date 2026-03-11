---
name: ali-macos-admin
description: |
  macOS operations with safety enforcement. Use when:

  PLANNING: Planning system configuration, Homebrew package management,
  launchd service architecture, automation workflows, system preferences

  IMPLEMENTATION: Executing brew commands, managing launchd services,
  configuring system preferences, running automations, managing processes

  GUIDANCE: Asking about macOS best practices, Homebrew management,
  launchd vs cron, system security settings, automation approaches

  REVIEW: Reviewing system configuration, auditing running services,
  checking installed packages, validating security settings
---

# macOS Admin

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Planning system configuration changes
- Designing Homebrew package management strategy
- Architecting launchd service automation
- Planning system preference workflows
- Evaluating automation approaches (Shortcuts, AppleScript, Automator)

**Implementation:**
- Executing Homebrew commands (install, upgrade, uninstall)
- Managing launchd services (load, unload, enable, disable)
- Configuring system preferences via defaults
- Running automations and shortcuts
- Managing processes and system resources

**Guidance/Best Practices:**
- Homebrew best practices and package selection
- launchd vs cron for scheduled tasks
- System security settings and recommendations
- Automation tool selection (when to use each)
- macOS system administration patterns

**Review/Validation:**
- Auditing running services and launch agents
- Checking installed packages and dependencies
- Validating security settings and configurations
- Reviewing automation scripts and workflows

---

## Core Principles

### 1. System Safety (Non-Destructive Defaults)

| Rule | Rationale |
|------|-----------|
| **Never disable SIP** | System Integrity Protection is macOS's core security |
| **Check dependencies before uninstall** | Prevent breaking dependent services |
| **Graceful termination first** | Try SIGTERM before SIGKILL |
| **Backup before system changes** | Time Machine should be enabled |
| **Respect macOS design** | Work with the system, not against it |

### 2. Homebrew Management

| Practice | Impact |
|----------|--------|
| **Keep Homebrew updated** | `brew update` before installs |
| **Check `brew doctor` regularly** | Catch configuration issues early |
| **Review dependencies** | `brew deps` and `brew uses` before changes |
| **Use cask for GUI apps** | Consistent update mechanism |
| **Pin critical packages** | Prevent accidental upgrades |

### 3. launchd Best Practices

| Practice | When to Use |
|----------|-------------|
| **LaunchAgents** | User-level tasks, GUI interactions |
| **LaunchDaemons** | System-level services, no GUI |
| **StartInterval** | Simple periodic tasks |
| **StartCalendarInterval** | Cron-like scheduling |
| **WatchPaths** | Trigger on file changes |

---

## Dangerous Operations (REQUIRE CONFIRMATION)

### Critical Risk (System Damage)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| `csrutil disable` | **ALWAYS BLOCKED** | System Integrity Protection |
| `rm -rf /` or `rm -rf ~` | **ALWAYS BLOCKED** | Catastrophic deletion |
| Modifying `/System/` | **ALWAYS BLOCKED** | SIP violation |
| `spctl --master-disable` | **ALWAYS BLOCKED** | Gatekeeper disable |
| `brew uninstall` | High - May break deps | "Yes, uninstall package [name]" |
| `launchctl unload` | High - Stops service | "Yes, unload service [name]" |
| `defaults delete` | Medium - Loses prefs | "Yes, delete preference [domain]" |
| `killall -9` | Medium - Data loss risk | "Yes, force quit [process]" |
| System directory chmod/chown | High - Permission issues | "Yes, modify permissions [path]" |

### High Risk (Major Changes)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| `brew upgrade --greedy` | Medium - Mass updates | "Yes, upgrade all packages" |
| `launchctl kickstart -k` | Medium - Force restart | "Yes, force restart [service]" |
| `networksetup` mods | High - Connectivity loss | "Yes, modify network settings" |
| `pmset` changes | Medium - Power behavior | "Yes, modify power settings" |

---

## Safe Operations (No Confirmation Needed)

```bash
# Homebrew - read-only
brew list
brew info <package>
brew search <term>
brew doctor
brew deps <package>
brew uses --installed <package>

# Homebrew - safe installs
brew install <package>          # New packages generally safe
brew tap <tap>

# launchd - read-only
launchctl list
launchctl print <service>
launchctl blame <service>

# System preferences - read-only
defaults read <domain>
defaults find <word>

# System information
system_profiler <type>
sw_vers
diskutil list
ps aux
lsof
netstat -an

# Safe utilities
open <file>
mdfind <query>
shortcuts list
shortcuts run <shortcut>
```

---

## Homebrew Patterns

### Package Installation

```bash
# Update Homebrew first
brew update

# Install package
brew install postgresql@15

# Install specific version
brew install postgresql@14

# Install with options
brew install ffmpeg --with-libvpx

# Install from tap
brew tap homebrew/cask
brew install --cask visual-studio-code
```

### Package Management

```bash
# Check for updates
brew outdated

# Upgrade specific package
brew upgrade postgresql

# Upgrade all (with confirmation if >10 packages)
brew upgrade

# Pin to prevent upgrades
brew pin postgresql

# Unpin
brew unpin postgresql

# Uninstall (always requires confirmation)
brew uninstall redis
```

### Dependency Management

```bash
# Show dependencies
brew deps postgresql

# Show reverse dependencies (what uses this)
brew uses --installed postgresql

# Show dependency tree
brew deps --tree postgresql

# Clean up old versions
brew cleanup
```

### Troubleshooting

```bash
# Check for issues
brew doctor

# Get package info
brew info postgresql

# Show caveats (post-install instructions)
brew info postgresql | grep -A 20 "Caveats"

# Link/unlink
brew unlink postgresql@14
brew link postgresql@15
```

---

## launchd Patterns

### Service Discovery

```bash
# List all loaded services
launchctl list

# Filter for specific service
launchctl list | grep postgresql

# Print service details
launchctl print gui/501/homebrew.mxcl.postgresql@15

# Show why service loaded
launchctl blame gui/501/homebrew.mxcl.postgresql@15
```

### Service Management

```bash
# Load service (enable and start)
launchctl load ~/Library/LaunchAgents/com.example.service.plist

# Unload service (disable and stop)
launchctl unload ~/Library/LaunchAgents/com.example.service.plist

# Start service (if loaded)
launchctl start com.example.service

# Stop service (if loaded)
launchctl stop com.example.service

# Restart service gracefully
launchctl kickstart gui/501/com.example.service

# Force restart (requires confirmation)
launchctl kickstart -k gui/501/com.example.service
```

### Plist Structure

**Simple periodic task:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.backup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/backup.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>3600</integer>
    <key>StandardOutPath</key>
    <string>/tmp/backup.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/backup.err</string>
</dict>
</plist>
```

**Scheduled task (cron-like):**
```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>2</integer>
    <key>Minute</key>
    <integer>0</integer>
</dict>
```

**File watch trigger:**
```xml
<key>WatchPaths</key>
<array>
    <string>/path/to/watch</string>
</array>
```

### Service Domains

| Domain | Path | Use Case |
|--------|------|----------|
| `~/Library/LaunchAgents/` | User agents | User automation, GUI apps |
| `/Library/LaunchAgents/` | Global agents | System-wide user tasks |
| `/Library/LaunchDaemons/` | Global daemons | System services (root) |

---

## System Preferences (defaults)

### Reading Preferences

```bash
# Read entire domain
defaults read com.apple.dock

# Read specific key
defaults read com.apple.dock autohide

# Find preferences containing word
defaults find autohide

# Export to plist
defaults export com.apple.dock ~/Desktop/dock-prefs.plist
```

### Writing Preferences

```bash
# Boolean values
defaults write com.apple.dock autohide -bool true

# Integer values
defaults write com.apple.dock tilesize -int 48

# String values
defaults write com.apple.dock orientation -string left

# Array values
defaults write com.apple.dock persistent-apps -array

# Apply changes (restart affected service)
killall Dock
```

### Common Preferences

**Dock:**
```bash
defaults write com.apple.dock autohide -bool true
defaults write com.apple.dock tilesize -int 48
defaults write com.apple.dock orientation -string left
killall Dock
```

**Finder:**
```bash
defaults write com.apple.finder AppleShowAllFiles -bool true
defaults write com.apple.finder ShowPathbar -bool true
killall Finder
```

**Screenshots:**
```bash
defaults write com.apple.screencapture location ~/Desktop/Screenshots
defaults write com.apple.screencapture type -string png
killall SystemUIServer
```

---

## Automation

### Shortcuts.app

```bash
# List available shortcuts
shortcuts list

# View shortcut details
shortcuts view "Backup Files"

# Run shortcut
shortcuts run "Backup Files"

# Run with input
shortcuts run "Process Document" --input-path ~/file.txt

# Run and get output
shortcuts run "Generate Report" --output-path ~/report.pdf
```

### AppleScript

```bash
# Run script file
osascript ~/Scripts/backup.scpt

# Run inline
osascript -e 'tell application "Safari" to activate'

# Multi-line inline
osascript <<EOF
tell application "Finder"
    make new Finder window
    set target of Finder window 1 to home
end tell
EOF
```

**Common patterns:**

```applescript
-- Activate application
tell application "Safari" to activate

-- Display dialog
display dialog "Backup complete" buttons {"OK"}

-- File operations
tell application "Finder"
    duplicate file "source" to folder "destination"
end tell

-- System Events
tell application "System Events"
    keystroke "f" using {command down}
end tell
```

### Automator

```bash
# Run workflow
automator ~/Documents/MyWorkflow.workflow

# Run from command line
automator -i input.txt ~/Documents/ProcessText.workflow
```

---

## Process Management

### Viewing Processes

```bash
# All processes
ps aux

# Sorted by CPU
ps aux --sort=-%cpu | head -10

# Sorted by memory
ps aux --sort=-%mem | head -10

# Process tree
pstree

# Real-time monitoring
top -o cpu
```

### Terminating Processes

**Graceful (SIGTERM):**
```bash
killall <process-name>
kill <pid>
```

**Force (SIGKILL) - requires confirmation:**
```bash
killall -9 <process-name>
kill -9 <pid>
```

### Common Processes to Restart

| Process | Purpose | Safe to Kill |
|---------|---------|--------------|
| Dock | macOS Dock | Yes (auto-restarts) |
| Finder | File browser | Yes (auto-restarts) |
| SystemUIServer | Menu bar | Yes (auto-restarts) |
| cfprefsd | Preferences daemon | Yes after defaults write |
| mds | Spotlight | No (use `mdutil` instead) |

---

## System Information

### Hardware

```bash
system_profiler SPHardwareDataType
system_profiler SPStorageDataType
system_profiler SPMemoryDataType
system_profiler SPDisplaysDataType
```

### Software

```bash
sw_vers                               # macOS version
system_profiler SPSoftwareDataType    # Detailed software info
```

### Network

```bash
# List network interfaces
networksetup -listallhardwareports

# Get interface info
networksetup -getinfo "Wi-Fi"

# DNS configuration
scutil --dns

# Active connections
netstat -an | grep ESTABLISHED
```

### Storage

```bash
# Disk usage
df -h

# Directory sizes
du -sh */

# Spotlight index status
mdutil -s /
```

---

## Security and Permissions

### System Integrity Protection (SIP)

```bash
# Check SIP status (read-only)
csrutil status

# NEVER disable SIP in production
# Disabling SIP is ALWAYS BLOCKED by macos-admin
```

### Gatekeeper

```bash
# Check Gatekeeper status (read-only)
spctl --status

# NEVER disable Gatekeeper globally
# Use per-app exceptions if needed:
xattr -d com.apple.quarantine /path/to/app
```

### Firewall

```bash
# Check firewall status (read-only)
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# NEVER disable firewall
# Use System Preferences to configure
```

### File Permissions

```bash
# View permissions
ls -la

# Change ownership (requires confirmation for system dirs)
chown user:group file

# Change permissions (requires confirmation for system dirs)
chmod 755 file
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Disabling SIP | Removes critical security | Use officially supported methods |
| Running everything with sudo | Unnecessary privilege escalation | Use appropriate user permissions |
| Force-quitting without checking | May cause data loss | Try graceful termination first |
| Modifying /System/ files | Breaks system integrity, SIP violation | Use system tools (softwareupdate, etc.) |
| Using cron on macOS | Deprecated, inconsistent | Use launchd instead |
| Ignoring Homebrew warnings | May indicate conflicts | Address warnings before proceeding |
| Uninstalling without checking deps | Breaks dependent services | Check `brew uses` first |
| Disabling Gatekeeper globally | Exposes to malware | Use per-app exceptions |
| Direct plist editing while loaded | Changes may be ignored | Unload, edit, reload |
| Multiple package managers | Conflicts and confusion | Stick to Homebrew |

---

## Pre-Execution Checklist

Before ANY macOS operation:

1. **Verify system state**
   ```bash
   sw_vers          # macOS version
   whoami           # Current user
   brew doctor      # Homebrew health
   ```

2. **Identify risk level**
   - Does it modify system files?
   - Does it affect running services?
   - Is it reversible?

3. **Check dependencies** (for uninstall/disable)
   ```bash
   brew uses --installed <package>
   ```

4. **Verify backup** (for critical operations)
   ```bash
   tmutil latestbackup      # Time Machine status
   ```

---

## Audit Trail

All macOS operations are logged to:
```
~/.claude/logs/macos-operations.log  # Detailed log
~/.claude/logs/macos-audit.log       # Structured audit trail
```

### Audit Log Format

```
[2026-02-14 10:30:45] ACTION=ALLOW COMMAND="brew install postgresql" RESULT=success USER=donmccarty
[2026-02-14 10:31:02] ACTION=BLOCK COMMAND="csrutil disable" RESULT=security-violation USER=donmccarty
[2026-02-14 10:32:15] ACTION=CONFIRM COMMAND="brew uninstall redis" RESULT=pending-confirmation USER=donmccarty
```

---

## Quick Reference

### Daily Commands (Safe)

```bash
brew list                             # List packages
brew info <package>                   # Package details
launchctl list                        # List services
defaults read <domain>                # Read preferences
system_profiler <type>                # System info
sw_vers                               # macOS version
ps aux                                # Process list
```

### Management Commands (Safe Installs)

```bash
brew install <package>                # Install new package
brew upgrade <package>                # Upgrade package
launchctl load <plist>                # Load service
defaults write <domain> <key> <val>   # Write preference
shortcuts run <name>                  # Run automation
```

### Dangerous Commands (Confirmation Required)

```bash
brew uninstall <package>              # REQUIRES CONFIRMATION
brew upgrade --greedy                 # REQUIRES CONFIRMATION
launchctl unload <plist>              # REQUIRES CONFIRMATION
defaults delete <domain>              # REQUIRES CONFIRMATION
killall -9 <process>                  # REQUIRES CONFIRMATION
```

---

## References

- [Homebrew Documentation](https://docs.brew.sh/)
- [launchd.info](https://www.launchd.info/)
- [macOS defaults](https://macos-defaults.com/)
- [Apple System Administration Guide](https://support.apple.com/guide/deployment/welcome/web)
- [Shortcuts User Guide](https://support.apple.com/guide/shortcuts-mac/welcome/mac)
