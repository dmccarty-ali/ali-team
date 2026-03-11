---
name: ali-macos-automation
description: |
  macOS automation with Shortcuts.app, AppleScript, Quick Actions, and shell integration. Use when:

  PLANNING: Designing macOS automations, planning Quick Actions workflows, architecting
  Shortcuts.app integrations, evaluating automation tool selection

  IMPLEMENTATION: Writing Shortcuts workflows, creating AppleScript scripts, building
  Quick Actions, implementing macOS shell automations, configuring app permissions

  GUIDANCE: Asking about Shortcuts.app best practices, AppleScript patterns, Quick Actions
  setup, macOS automation capabilities, file system permissions for automation

  REVIEW: Checking Shortcuts configurations, reviewing AppleScript for compatibility,
  validating Quick Actions setup, auditing automation permissions and security
---

# macOS Automation

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing macOS automation workflows
- Planning Shortcuts.app integrations with other apps
- Evaluating automation approach (Shortcuts vs AppleScript vs shell)
- Architecting Quick Actions for Finder and Services
- Considering app permissions and sandboxing constraints

**Implementation:**
- Creating Shortcuts.app workflows
- Writing AppleScript or JavaScript for Automation (JXA)
- Building Quick Actions or Automator workflows
- Implementing macOS shell commands for automation
- Setting up Launch Agents or Launch Daemons
- Configuring app permissions and accessibility access

**Guidance/Best Practices:**
- Asking about Shortcuts.app capabilities and limitations
- Asking how to trigger automations from different contexts
- Asking about AppleScript vs JXA vs shell scripting trade-offs
- Asking about file system access and permission patterns
- Asking about notification and dialog APIs

**Review/Validation:**
- Reviewing Shortcuts for efficiency and error handling
- Checking AppleScript compatibility across macOS versions
- Validating Quick Actions setup and registration
- Auditing automation security and permissions
- Checking path handling with spaces and special characters

---

## Key Principles

- **Shortcuts.app for user workflows**: Best for simple, user-facing automations with visual feedback
- **AppleScript for app control**: Use osascript when you need to control other Mac apps programmatically
- **Shell for system operations**: Prefer bash/zsh for file operations, system commands, and CLI tools
- **Quote all paths**: Always use double quotes for file paths (spaces are common on macOS)
- **Check permissions first**: Many automations require Full Disk Access, Accessibility, or Automation permissions
- **Fail gracefully**: Provide clear error messages when permissions or apps are missing
- **Use notification APIs**: osascript display notification for user feedback
- **Export shortcuts**: Use .shortcut files for version control and sharing
- **Test across macOS versions**: Shortcuts and AppleScript APIs change between macOS releases

---

## Shortcuts.app Patterns

### Basic Shortcut Structure

Shortcuts are visual workflows built in the Shortcuts app with actions that process input and produce output.

**Common Actions:**
- Get File/Folder - Select files from Finder
- Run Shell Script - Execute bash/zsh commands
- Run AppleScript - Execute AppleScript code
- Show Notification - Display user notifications
- Set Variable - Store values for later use
- If/Otherwise - Conditional logic
- Repeat - Loop over items
- Get Contents of URL - HTTP requests

### Triggering Shortcuts

```bash
# Run shortcut from command line
shortcuts run "My Shortcut Name"

# Run with input (pass file path)
shortcuts run "Process File" -i "/path/to/file.txt"

# List all shortcuts
shortcuts list
```

### Quick Action Integration

Quick Actions appear in:
- Finder context menu (right-click on files/folders)
- Services menu in apps
- Touch Bar on supported Macs

**Setup:**
1. Create shortcut in Shortcuts.app
2. Set Quick Actions for Finder and Services in shortcut settings
3. Choose accepted file types (images, text files, PDFs, etc.)
4. Shortcut receives selected files/folders as input

### Shortcuts Limitations

- **No loops over large datasets**: Slow performance with 100+ items
- **Limited debugging**: No breakpoints or step-through debugging
- **Version control challenges**: .shortcut files are binary plists (use export for sharing)
- **App-specific actions**: Some actions only work when specific apps are installed
- **Sandboxing**: Limited access to system files without Full Disk Access
- **Error handling**: Limited try/catch capabilities compared to scripting

---

## AppleScript Patterns

### Running AppleScript

```bash
# Execute AppleScript from command line
osascript -e 'display notification "Hello" with title "My Script"'

# Execute AppleScript file
osascript /path/to/script.scpt

# Pass arguments to AppleScript
osascript script.scpt "arg1" "arg2"
```

### Display Notifications

```applescript
-- Simple notification
display notification "Task completed" with title "Success"

-- Notification with sound
display notification "Build finished" with title "CI/CD" sound name "Glass"

-- Available sounds: Basso, Blow, Bottle, Frog, Funk, Glass, Hero,
-- Morse, Ping, Pop, Purr, Sosumi, Submarine, Tink
```

### Display Dialogs

```applescript
-- Simple alert
display alert "Error occurred" message "Could not connect to server"

-- Dialog with buttons
set response to display dialog "Continue?" buttons {"Cancel", "OK"} default button "OK"
if button returned of response is "OK" then
    -- User clicked OK
end if

-- Text input dialog
set userInput to text returned of (display dialog "Enter your name:" default answer "")
```

### File Operations

```applescript
-- Choose file dialog
set theFile to choose file with prompt "Select a file:"

-- Choose folder dialog
set theFolder to choose folder with prompt "Select a folder:"

-- Get file path as string
set filePath to POSIX path of theFile

-- Read file contents
set fileContents to read theFile as «class utf8»

-- Write to file
set theFile to POSIX file "/path/to/file.txt"
set fileRef to open for access theFile with write permission
write "Hello World" to fileRef
close access fileRef
```

### Application Control

```applescript
-- Tell application to do something
tell application "Finder"
    activate
    make new Finder window
    set target of Finder window 1 to home
end tell

-- Get application info
tell application "System Events"
    set appList to name of every application process
end tell

-- Open URL in default browser
open location "https://example.com"

-- Open file with default application
tell application "Finder"
    open POSIX file "/path/to/document.pdf"
end tell
```

### System Information

```applescript
-- Get username
set userName to short user name of (system info)

-- Get computer name
set computerName to computer name of (system info)

-- Get macOS version
set osVersion to system version of (system info)

-- Get clipboard contents
set clipboardData to the clipboard
```

### Error Handling

```applescript
try
    -- Code that might fail
    set theFile to choose file
on error errMsg number errNum
    display alert "Error " & errNum message errMsg
end try
```

---

## Shell Command Patterns

### Opening Files and Applications

```bash
# Open file with default application
open "/path/to/document.pdf"

# Open file with specific application
open -a "Preview" "/path/to/image.png"

# Open URL in default browser
open "https://example.com"

# Open folder in Finder
open "/path/to/folder"

# Reveal file in Finder
open -R "/path/to/file.txt"

# Open multiple files
open file1.txt file2.txt file3.txt
```

### Clipboard Operations

```bash
# Copy to clipboard
echo "Hello World" | pbcopy

# Read from clipboard
pbpaste

# Copy file contents to clipboard
cat file.txt | pbcopy

# Save clipboard to file
pbpaste > output.txt
```

### Path Handling with Spaces

```bash
# ALWAYS quote paths with spaces
open "/Users/donmccarty/My Documents/file.txt"

# Variable assignment with spaces
file_path="/path/with spaces/file.txt"
open "$file_path"

# Loop over files with spaces in names
find . -name "*.txt" -print0 | while IFS= read -r -d '' file; do
    echo "Processing: $file"
done
```

### Notifications from Shell

```bash
# Display notification
osascript -e 'display notification "Build complete" with title "CI/CD"'

# Display dialog
osascript -e 'display dialog "Process finished" buttons {"OK"} default button "OK"'

# Display alert with error icon
osascript -e 'display alert "Error" message "Something went wrong"'
```

### Launch Agents

Launch Agents run scripts on schedule or when triggered.

```xml
<!-- ~/Library/LaunchAgents/com.example.task.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.task</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/my-script.sh</string>
    </array>

    <key>StartInterval</key>
    <integer>3600</integer> <!-- Run every hour -->

    <key>RunAtLoad</key>
    <true/> <!-- Run when loaded -->

    <key>StandardOutPath</key>
    <string>/tmp/my-script.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/my-script-error.log</string>
</dict>
</plist>
```

```bash
# Load Launch Agent
launchctl load ~/Library/LaunchAgents/com.example.task.plist

# Unload Launch Agent
launchctl unload ~/Library/LaunchAgents/com.example.task.plist

# Start immediately
launchctl start com.example.task

# Check status
launchctl list | grep com.example.task
```

---

## Quick Actions and Automator

### Creating Quick Actions

1. Open Automator.app
2. Choose Quick Action (previously called Service)
3. Set Workflow receives current: files or folders, in: Finder
4. Add actions: Run Shell Script, Run AppleScript, etc.
5. Save with descriptive name (appears in Finder context menu)

### Quick Action Shell Script Example

Workflow settings:
- Workflow receives current: files or folders
- in: Finder.app

Action: Run Shell Script
- Shell: /bin/bash
- Pass input: as arguments

```bash
#!/bin/bash

# Process each selected file
for file in "$@"; do
    # Get filename and extension
    filename=$(basename "$file")
    extension="${filename##*.}"
    name="${filename%.*}"

    echo "Processing: $file"

    # Your automation here
done

# Display notification when done
osascript -e 'display notification "Processing complete" with title "Quick Action"'
```

### Quick Action AppleScript Example

Workflow settings:
- Workflow receives current: files or folders
- in: Finder.app

Action: Run AppleScript

```applescript
on run {input, parameters}
    repeat with theFile in input
        -- Process each file
        set filePath to POSIX path of theFile
        -- Your automation here
    end repeat

    display notification "Processing complete" with title "Quick Action"
    return input
end run
```

---

## File System Permissions

### Permission Types

| Permission | Purpose | Required For |
|------------|---------|--------------|
| Full Disk Access | Access files outside app sandbox | Reading/writing protected locations |
| Accessibility | Control other applications | GUI automation, key presses |
| Automation | Control specific apps | AppleScript app control |
| Files and Folders | Access specific locations | Documents, Downloads, Desktop |

### Granting Permissions

System Settings > Privacy & Security > [Permission Type]

**Common apps needing permissions:**
- Terminal.app - For running shell scripts
- Shortcuts.app - For file operations
- Script Editor.app - For running AppleScript
- Automator.app - For Quick Actions

### Checking Permissions in Scripts

```bash
# Check if running with Full Disk Access
if [[ ! -r ~/Library/Mail ]]; then
    osascript -e 'display alert "Permission Required" message "This script needs Full Disk Access. Add Terminal to System Settings > Privacy & Security > Full Disk Access."'
    exit 1
fi
```

```applescript
-- Check if app is running
tell application "System Events"
    set isRunning to exists (processes where name is "MyApp")
end tell

if not isRunning then
    display alert "App Not Running" message "Please launch MyApp first."
end if
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Unquoted paths in shell | Breaks with spaces | Always use double quotes: "$file_path" |
| No error handling | Silent failures | Add try/catch in AppleScript, check exit codes in shell |
| Hardcoded absolute paths | Breaks on other machines | Use relative paths or environment variables |
| No permission checks | Cryptic failures | Check permissions first, provide clear error messages |
| Shortcuts for heavy processing | Slow, no progress feedback | Use shell scripts called from Shortcuts |
| No user feedback | User doesn't know script is running | Use notifications or dialogs |
| Binary .shortcut in git | Can't see changes | Export as text description or screenshots |
| Accessing ~/Library without FDA | Access denied errors | Request Full Disk Access in instructions |
| Using deprecated System Events | Breaks on newer macOS | Check macOS version, use modern APIs |
| No macOS version checks | Breaks on older/newer systems | Check system version before using new APIs |

---

## Quick Reference

### Common osascript Commands

```bash
# Notification
osascript -e 'display notification "Message" with title "Title"'

# Alert dialog
osascript -e 'display alert "Title" message "Message"'

# Get clipboard
osascript -e 'the clipboard'

# Set clipboard
osascript -e 'set the clipboard to "Text"'

# Get username
osascript -e 'short user name of (system info)'

# Get macOS version
osascript -e 'system version of (system info)'
```

### Common File Operations

```bash
# Get directory of current script
script_dir="$(cd "$(dirname "$0")" && pwd)"

# Get user's home directory
home_dir="$HOME"

# Standard macOS paths
~/Desktop
~/Documents
~/Downloads
~/Library
~/Library/Application Support
~/Library/Preferences

# Application paths
/Applications
/System/Applications
~/Applications
```

### Keyboard Maestro Alternative Patterns

If Keyboard Maestro is available, it offers:
- Better GUI automation than AppleScript
- More reliable keyboard/mouse control
- Extensive macro library
- Better debugging tools

But for scripts without external dependencies, prefer built-in macOS tools.

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| User shortcuts | ~/Library/Shortcuts/ |
| Launch Agents | ~/Library/LaunchAgents/ |
| Quick Actions (saved) | ~/Library/Services/ |
| AppleScript files | Project-specific locations |
| Shell scripts | Project bin/ directories |

---

## References

- [Shortcuts User Guide](https://support.apple.com/guide/shortcuts-mac)
- [AppleScript Language Guide](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide)
- [Launch Agent/Daemon Documentation](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
- [osascript Man Page](https://ss64.com/osx/osascript.html)
- [macOS Automation with JavaScript](https://developer.apple.com/library/archive/releasenotes/InterapplicationCommunication/RN-JavaScriptForAutomation)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-22
**Maintained By:** ALI AI Team
