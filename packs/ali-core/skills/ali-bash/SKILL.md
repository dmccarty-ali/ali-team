---
name: ali-bash
description: |
  Bash shell scripting best practices, patterns, and security. Use when:

  PLANNING: Designing shell scripts, planning automation workflows, architecting
  script-based tools, choosing between bash and other languages, designing error
  handling and logging strategies

  IMPLEMENTATION: Writing shell scripts, implementing automation, creating hooks,
  building deployment scripts, fixing shell script bugs, adding error handling

  GUIDANCE: Asking about bash best practices, asking how to do X in bash safely,
  asking about portability (POSIX vs bash), asking about performance optimization,
  asking about shellcheck rules

  REVIEW: Reviewing shell scripts for safety, checking for security issues, validating
  error handling, assessing portability, checking for common pitfalls
---

# Bash Shell Scripting

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing shell scripts or automation workflows
- Planning error handling and logging strategies
- Architecting script-based deployment or build tools
- Choosing between bash scripting and other languages
- Designing hook systems or plugin architectures

**Implementation:**
- Writing bash, sh, or zsh scripts
- Implementing automation workflows
- Creating git hooks or build hooks
- Building deployment or setup scripts
- Fixing shell script bugs or errors
- Adding error handling or validation

**Guidance/Best Practices:**
- Asking about bash best practices or security
- Asking how to accomplish specific tasks safely
- Asking about portability (POSIX vs bash-specific)
- Asking about shellcheck warnings or rules
- Asking about performance optimization
- Asking when to use bash vs Python/other

**Review/Validation:**
- Reviewing shell scripts for correctness
- Checking for security vulnerabilities
- Validating error handling and edge cases
- Assessing script portability
- Checking for common anti-patterns

---

## Key Principles

- **Strict mode by default** - Use set -euo pipefail to catch errors early
- **Always quote variables** - Unquoted variables cause word splitting and globbing bugs
- **Check exit codes** - Verify critical commands succeeded before proceeding
- **Fail fast, fail loud** - Detect errors immediately and report clearly
- **POSIX when possible** - Use portable syntax unless bash features are needed
- **Functions over duplication** - DRY applies to shell scripts too
- **Validate inputs** - Never trust user input or environment variables
- **Atomic operations** - Avoid race conditions (TOCTOU vulnerabilities)
- **Write-script-first** - If your Bash command is too long to meaningfully review in a permission prompt, it's too long to be inline

---

## Patterns We Use

### Strict Mode (Mandatory)

Every script starts with strict mode to catch errors early:

```bash
#!/bin/bash
# script-name.sh
# Description of what this script does
#
# Usage: script-name.sh [OPTIONS]

set -euo pipefail

# -e: Exit immediately if a command exits with a non-zero status
# -u: Treat unset variables as an error
# -o pipefail: Return exit code of first failed command in a pipe
```

**Why:**
- `-e` prevents cascading failures (script continues after errors)
- `-u` catches typos in variable names immediately
- `pipefail` detects pipe failures (otherwise only last command's exit code matters)

**When to disable:**
```bash
# Temporarily disable for commands expected to fail
set +e
command_that_might_fail
exit_code=$?
set -e

# Or use conditional
if command_that_might_fail; then
    echo "Succeeded"
else
    echo "Failed (expected)"
fi
```

---

### Variable Quoting (Always)

**Always quote variable expansions:**

```bash
# WRONG - word splitting and globbing issues
rm -rf $DIR/*.txt
cp $FILE $DEST

# CORRECT - prevents splitting and globbing
rm -rf "${DIR}"/*.txt
cp "${FILE}" "${DEST}"

# CORRECT - array expansion
files=( file1.txt file2.txt "file with spaces.txt" )
for file in "${files[@]}"; do
    process "$file"
done
```

**Why:**
- Unquoted variables split on spaces: `DIR="/my files"` → `rm -rf /my files/*.txt` deletes `/my` and `files/*.txt`
- Unquoted variables undergo globbing: `FILE="*.txt"` → expands to all .txt files
- Quoting prevents both issues

**When not to quote:**
```bash
# Intentional word splitting (rare)
flags="-v -x -f"
command $flags  # Intentionally split into separate arguments
```

---

### Error Handling with trap

Clean up on error, exit, or interrupt:

```bash
#!/bin/bash
set -euo pipefail

# Cleanup function
cleanup() {
    local exit_code=$?
    echo "Cleaning up..."
    rm -f /tmp/script.$$.*
    [ -n "${TEMP_DIR:-}" ] && rm -rf "${TEMP_DIR}"
    exit "${exit_code}"
}

# Register cleanup on EXIT (covers normal exit, error, Ctrl-C)
trap cleanup EXIT

# Script logic
TEMP_DIR=$(mktemp -d)
echo "Working in ${TEMP_DIR}"
# ... do work ...
```

**Benefits:**
- Cleanup always runs (error, success, or interrupt)
- Resources are released (temp files, locks, connections)
- No leaked state on error

**Advanced trap usage:**
```bash
# Different handlers for different signals
trap 'echo "Error on line ${LINENO}"' ERR
trap 'echo "Interrupted"; exit 130' INT TERM
trap cleanup EXIT
```

---

### Exit Code Checking

Check critical commands explicitly:

```bash
# WRONG - continues on failure
mkdir /critical/path
cd /critical/path
rm -rf *

# CORRECT - exit on failure (with set -e)
mkdir /critical/path || { echo "Failed to create directory"; exit 1; }
cd /critical/path || { echo "Failed to change directory"; exit 1; }
rm -rf *

# CORRECT - conditional check
if ! mkdir /critical/path; then
    echo "Error: Failed to create /critical/path" >&2
    exit 1
fi
```

**Standard exit codes:**
- `0` - Success
- `1` - General error
- `2` - Misuse of shell builtin
- `126` - Command cannot execute
- `127` - Command not found
- `128+n` - Fatal error signal n (e.g., 130 = Ctrl-C)
- `255` - Exit code out of range

---

### Input Validation

Never trust input:

```bash
# Validate arguments
if [ $# -ne 2 ]; then
    echo "Usage: $0 <source> <dest>" >&2
    exit 1
fi

SOURCE="$1"
DEST="$2"

# Validate file exists
if [ ! -f "${SOURCE}" ]; then
    echo "Error: Source file '${SOURCE}' not found" >&2
    exit 1
fi

# Validate path doesn't escape
case "${DEST}" in
    *..*)
        echo "Error: Path traversal detected in '${DEST}'" >&2
        exit 1
        ;;
esac

# Validate environment variable
: "${REQUIRED_VAR:?Error: REQUIRED_VAR must be set}"
```

---

### Functions

Use functions for reusable logic:

```bash
#!/bin/bash
set -euo pipefail

# Function with documentation
# Args:
#   $1 - Source file path
#   $2 - Destination directory
# Returns:
#   0 on success, 1 on error
copy_and_validate() {
    local src="$1"
    local dest_dir="$2"
    local dest_file="${dest_dir}/$(basename "${src}")"

    if [ ! -f "${src}" ]; then
        echo "Error: Source '${src}' not found" >&2
        return 1
    fi

    cp "${src}" "${dest_dir}/"

    if ! cmp -s "${src}" "${dest_file}"; then
        echo "Error: Copy verification failed" >&2
        return 1
    fi

    echo "Copied ${src} to ${dest_dir}"
    return 0
}

# Use local variables in functions
process_files() {
    local file
    for file in "$@"; do
        echo "Processing: ${file}"
        # ... processing logic ...
    done
}

# Call functions
copy_and_validate "/path/to/file" "/dest/dir" || exit 1
process_files file1.txt file2.txt file3.txt
```

**Function best practices:**
- Use `local` for all function variables (prevents global pollution)
- Document parameters and return codes
- Return meaningful exit codes (0 = success)
- Use `"$@"` to pass all arguments through

---

### Arrays

Use arrays for lists of items:

```bash
# Declare array
files=( file1.txt file2.txt "file with spaces.txt" )

# Append to array
files+=( file4.txt )

# Iterate over array
for file in "${files[@]}"; do
    echo "Processing: ${file}"
done

# Array length
echo "Count: ${#files[@]}"

# Associative arrays (bash 4+)
declare -A colors
colors[red]="#FF0000"
colors[green]="#00FF00"

# Iterate associative array
for key in "${!colors[@]}"; do
    echo "${key} = ${colors[$key]}"
done
```

**Array gotchas:**
- `"${arr[@]}"` - expands to separate words (correct)
- `"${arr[*]}"` - expands to single word (usually wrong)
- `${arr[@]}` without quotes - word splits each element (wrong)

---

### Logging

Structured logging with timestamps:

```bash
# Log levels
log_info() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] INFO: $*"
}

log_warn() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] WARN: $*" >&2
}

log_error() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $*" >&2
}

# Usage
log_info "Starting deployment"
log_warn "Disk space low"
log_error "Failed to connect to database"
```

**Redirect to log file:**
```bash
# Log to file and stdout
exec > >(tee -a script.log)
exec 2>&1

# Or redirect manually
{
    log_info "Starting"
    # ... script logic ...
    log_info "Completed"
} >> script.log 2>&1
```

---

### Argument Parsing with getopts

```bash
#!/bin/bash
set -euo pipefail

usage() {
    cat <<EOF
Usage: $0 [OPTIONS]

Options:
  -f FILE    Input file (required)
  -o DIR     Output directory (default: ./output)
  -v         Verbose mode
  -h         Show this help
EOF
    exit 1
}

# Defaults
OUTPUT_DIR="./output"
VERBOSE=false
INPUT_FILE=""

# Parse options
while getopts "f:o:vh" opt; do
    case "${opt}" in
        f)
            INPUT_FILE="${OPTARG}"
            ;;
        o)
            OUTPUT_DIR="${OPTARG}"
            ;;
        v)
            VERBOSE=true
            ;;
        h)
            usage
            ;;
        *)
            usage
            ;;
    esac
done
shift $((OPTIND - 1))

# Validate required options
if [ -z "${INPUT_FILE}" ]; then
    echo "Error: -f FILE is required" >&2
    usage
fi

# Use parsed options
[ "${VERBOSE}" = true ] && echo "Input: ${INPUT_FILE}, Output: ${OUTPUT_DIR}"
```

---

### Safe File Operations

**Atomic file writes:**
```bash
# WRONG - not atomic, partial writes on error
echo "data" > /important/file.txt

# CORRECT - write to temp, then move (atomic)
TEMP_FILE=$(mktemp)
trap 'rm -f "${TEMP_FILE}"' EXIT

echo "data" > "${TEMP_FILE}"
mv "${TEMP_FILE}" /important/file.txt
```

**Safe file checks:**
```bash
# Avoid TOCTOU (Time-Of-Check-Time-Of-Use) race condition

# WRONG - file could be deleted/changed between check and use
if [ -f "${FILE}" ]; then
    cat "${FILE}"  # FILE might not exist here
fi

# CORRECT - let command fail if file doesn't exist
cat "${FILE}" 2>/dev/null || {
    echo "Error: Could not read ${FILE}" >&2
    exit 1
}

# CORRECT - use command directly
if cat "${FILE}" 2>/dev/null; then
    echo "File read successfully"
fi
```

---

### Command Substitution

Use `$()` instead of backticks:

```bash
# OLD STYLE - backticks (hard to nest, hard to read)
files=`find . -name "*.txt"`
date=`date +%Y%m%d`

# NEW STYLE - $() (can nest, clearer)
files=$(find . -name "*.txt")
date=$(date +%Y%m%d)

# Nesting example
outer=$(echo "inner=$(date)")
```

---

### Process Substitution

Avoid temporary files:

```bash
# WRONG - creates temporary file
find . -name "*.txt" > /tmp/files.txt
while read -r file; do
    process "$file"
done < /tmp/files.txt
rm /tmp/files.txt

# CORRECT - no temporary file needed
while read -r file; do
    process "$file"
done < <(find . -name "*.txt")

# Compare two command outputs
diff <(sort file1.txt) <(sort file2.txt)
```

---

### Shellcheck Integration

Run shellcheck on all scripts:

```bash
# Install shellcheck
# macOS: brew install shellcheck
# Ubuntu: apt install shellcheck

# Run on script
shellcheck script.sh

# Suppress specific warnings
# shellcheck disable=SC2086
intentionally_unquoted=$var

# Check all scripts
find . -name "*.sh" -exec shellcheck {} +
```

**Common shellcheck warnings:**
- SC2086 - Unquoted variable (word splitting)
- SC2046 - Quote command substitution
- SC2006 - Use $() instead of backticks
- SC2181 - Check exit code directly (if cmd; then, not if [ $? -eq 0 ])
- SC2155 - Declare and assign separately (local var=$(cmd) hides cmd exit code)

---

## Write-Script-First Rule

**General principle: If your Bash command is too long to meaningfully review in a permission prompt, it's too long to be inline.**

Long inline commands create unusable permission prompts (the full command appears in the approval dialog), bloat settings.json if the user clicks "don't ask again," bypass review and version control, and are impossible to debug after the fact.

### When a Command Is "Too Long" (Thresholds)

A command crosses the line when it hits ANY of these:

- More than 5 lines of logic (excluding comments and blank lines)
- More than 200 characters of logic content
- A heredoc (`<< 'EOF'`) of any kind
- A piped chain of 3+ commands
- Conditional logic (if/then, case) inside the command
- A loop (for, while)
- Variable definitions followed by their use

### Application 1: File Creation

**NEVER** use `Bash(cat > file << 'EOF' ... EOF)` to create files.

**ALWAYS** use the Write tool to create files, then Bash to execute them.

This applies to ALL file types: scripts, configs, data files, Python files, everything. Using Bash heredocs to write files creates unusable permission prompts (the entire heredoc shows in the approval dialog), pollutes settings.json if the user clicks "don't ask again" (148 lines stored as a permanent allow rule), and bypasses version control, review, and tracking capabilities.

```
BAD:  Bash(cat > script.py << 'EOF'
      [148 lines of Python]
      EOF)

GOOD: Write(script.py, content)
      then Bash(python script.py)
```

### Application 2: Complex Ad-Hoc Scripts

When an ad-hoc Bash command hits the thresholds above, write it as a script file first:

1. Use Write tool to create `.tmp/scripts/{descriptive-name}.sh`
2. Use Bash to `chmod +x` and execute it
3. Script follows ali-bash conventions: shebang, strict mode, proper quoting

```bash
# Step 1 - Write tool creates the file:
# .tmp/scripts/process-json.sh

#!/bin/bash
set -euo pipefail

for f in *.json; do
    name=$(jq -r '.name' "$f")
    echo "$f: $name"
done

# Step 2 - Bash executes it:
chmod +x .tmp/scripts/process-json.sh
bash .tmp/scripts/process-json.sh
```

```
BAD:  Bash(for f in *.json; do jq '.name' "$f" | while read name; do
      echo "$f: $name"
      done
      done)

GOOD: Write(.tmp/scripts/process-json.sh, content)
      then Bash(chmod +x .tmp/scripts/process-json.sh && bash .tmp/scripts/process-json.sh)
```

### Application 3: Git Commands with Multi-Line Messages

Git has native file-input flags for commit messages, tag annotations, and notes. Use them instead of heredoc injection — heredocs hit the threshold and cause the same permission prompt problems.

**Native git flags:**
- `git commit -F <file>` — read commit message from file
- `git tag -F <file>` — read tag annotation from file
- `git notes add -F <file>` — read note from file

```
BAD:  Bash(git commit -m "$(cat <<'EOF'
      ... 20 lines of commit message ...
      EOF
      )")

GOOD: Write(.tmp/scripts/commit-msg.txt, message)
      then Bash(git commit -F .tmp/scripts/commit-msg.txt)

BAD:  Bash(git tag -a v1.0 -m "$(cat <<'EOF'
      ... release notes ...
      EOF
      )")

GOOD: Write(.tmp/scripts/tag-msg.txt, notes)
      then Bash(git tag -a v1.0 -F .tmp/scripts/tag-msg.txt)
```

### Exceptions (Inline Bash is Fine)

- Single read-only commands: `git status`, `git log`, `ls -la`
- Make targets: `make test-structural`
- Simple one-liners under 5 lines AND under 200 chars with no conditionals, loops, or heredocs

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No strict mode | Errors go undetected | `set -euo pipefail` at top |
| Unquoted variables | Word splitting, globbing | Always use `"${var}"` |
| Ignoring exit codes | Silent failures | Check critical commands explicitly |
| Missing trap cleanup | Leaked temp files, locks | `trap cleanup EXIT` |
| `cd` without checking | Operations in wrong directory | `cd /path || exit 1` |
| Using `eval` | Code injection risk | Almost always avoidable |
| `rm -rf $var` | Deletes wrong files if var empty | Check variable first: `[ -n "$var" ]` |
| Backticks | Hard to read, can't nest | Use `$()` |
| `cat file \| grep` | Unnecessary cat | `grep pattern file` |
| Bashisms in `#!/bin/sh` | Not portable | Use `#!/bin/bash` or POSIX syntax |
| `[ $var == "value" ]` | Not POSIX, fails on unset | Use `[[ ]]` or `[ "$var" = "value" ]` |
| Long pipes | Hard to debug | Break into steps with intermediate checks |
| No input validation | Security vulnerabilities | Validate all inputs |
| `cat > file << 'EOF'` in Bash | Bypasses Write tool, pollutes permissions | Use Write tool to create files |
| Complex logic inline | Unusable permission prompts | Write as .tmp/scripts/ file first |
| Multi-line git messages via heredoc | Same heredoc problems; git has native flags | Write message to file, use `git commit -F` / `git tag -F` |

---

## Quick Reference

### Strict Mode Template

```bash
#!/bin/bash
# script-name.sh
# Description
#
# Usage: script-name.sh [OPTIONS]

set -euo pipefail

# Cleanup on exit
cleanup() {
    # Cleanup logic
    :
}
trap cleanup EXIT

# Constants
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"

# Main logic here
main() {
    # Script implementation
    :
}

main "$@"
```

---

### Common Command Patterns

**Check if command exists:**
```bash
if command -v jq &> /dev/null; then
    echo "jq is installed"
else
    echo "jq not found" >&2
    exit 1
fi
```

**Read file line by line:**
```bash
while IFS= read -r line; do
    echo "Line: ${line}"
done < file.txt
```

**Parse CSV:**
```bash
while IFS=, read -r col1 col2 col3; do
    echo "Col1: ${col1}, Col2: ${col2}"
done < data.csv
```

**Find files and process:**
```bash
# Using find with -exec
find . -name "*.txt" -exec grep "pattern" {} +

# Using find with while loop
find . -name "*.txt" -print0 | while IFS= read -r -d '' file; do
    process "$file"
done
```

**Check if variable is set:**
```bash
# Exit if not set
: "${VAR:?Error: VAR must be set}"

# Provide default
VAR="${VAR:-default_value}"

# Set if not set
: "${VAR:=default_value}"
```

**Conditional execution:**
```bash
# Run command if previous succeeded
command1 && command2

# Run command if previous failed
command1 || command2

# Complex conditional
if command1; then
    command2
elif command3; then
    command4
else
    command5
fi
```

---

### POSIX vs Bash

**POSIX-compliant (works in #!/bin/sh):**
```bash
#!/bin/sh
# No arrays
# Use [ ] not [[ ]]
# Use = not ==
# No process substitution
# No $((var++))

if [ "$var" = "value" ]; then
    echo "POSIX compliant"
fi
```

**Bash-specific (requires #!/bin/bash):**
```bash
#!/bin/bash
# Arrays: arr=(1 2 3)
# [[ ]] with regex: [[ $var =~ pattern ]]
# Process substitution: diff <(cmd1) <(cmd2)
# Arithmetic: $((i++))
# String manipulation: ${var^^} (uppercase)

files=( file1 file2 file3 )
if [[ "${files[0]}" =~ ^file ]]; then
    echo "Bash-specific"
fi
```

**When to use which:**
- Use POSIX (`#!/bin/sh`) for maximum portability
- Use Bash (`#!/bin/bash`) when you need arrays, advanced features
- Document requirements if using zsh-specific features

---

### Performance Tips

**Avoid repeated process spawning:**
```bash
# SLOW - spawns grep for each iteration
for file in *.txt; do
    if grep "pattern" "$file" > /dev/null; then
        echo "$file"
    fi
done

# FAST - single grep invocation
grep -l "pattern" *.txt
```

**Use builtins when possible:**
```bash
# SLOW - spawns external command
length=$(echo "$string" | wc -c)

# FAST - bash builtin
length=${#string}

# SLOW - external basename
filename=$(basename "$path")

# FAST - parameter expansion
filename="${path##*/}"
```

**Avoid unnecessary pipes:**
```bash
# UNNECESSARY
cat file.txt | grep pattern

# BETTER
grep pattern file.txt

# UNNECESSARY
ls | wc -l

# BETTER (but see note below)
echo */  # Glob expansion
```

---

### Security Checklist

Before deploying a script:

- [ ] Strict mode enabled (`set -euo pipefail`)
- [ ] All variables quoted
- [ ] No hardcoded secrets
- [ ] Input validation on all user-provided data
- [ ] No `eval` usage
- [ ] No command injection vectors
- [ ] Proper file permissions set (umask, chmod)
- [ ] No TOCTOU race conditions
- [ ] Cleanup on exit (trap)
- [ ] Error messages don't leak sensitive information
- [ ] Shellcheck passes with no warnings
- [ ] Script tested in target environment
- [ ] No `cat > file << 'EOF'` heredocs (use Write tool instead)

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| claude-ali launcher | ~/.claude/bin/claude-ali |
| Sync script | ~/.claude/bin/sync-org-resources.sh |
| Migration runner | ~/.claude/bin/run-pending-migrations.sh |
| Project setup | ~/.claude/bin/init-aliunde-project.sh |
| Environment setup | ~/.claude/bin/setup_dev_environment.sh |
| Git guard hook | ~/ali-ai/hooks/ali-git-guard.sh |
| AWS guard hook | ~/ali-ai/hooks/ali-aws-guard.sh |
| Migration template | ~/ali-ai/migrations/MIGRATION_TEMPLATE.sh |

---

## References

- [Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html)
- [ShellCheck Wiki](https://www.shellcheck.net/wiki/)
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- [Bash Pitfalls](https://mywiki.wooledge.org/BashPitfalls)
- [POSIX Shell Command Language](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html)
