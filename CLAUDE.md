# CLAUDE.md - AI Assistant Guide for fancy-motd

## Project Overview

**fancy-motd** is a modular, colorful Message of the Day (MOTD) system written in Bash. It displays server status information at login, including system stats, services, disk usage, and more.

### Key Characteristics
- **Language**: Pure Bash (no Python, Ruby, etc.)
- **Architecture**: Modular plugin-based system
- **Output**: Formatted terminal output with ANSI colors
- **Execution**: Run-parts style module loading

## Repository Structure

```
fancy-motd/
├── motd.sh              # Main entry point
├── framework.sh         # Core utility functions
├── config.sh            # User configuration (gitignored)
├── modules/             # Plugin directory
│   ├── 10-user         # User/login info
│   ├── 11-os           # OS information
│   ├── 12-ip           # IP addresses
│   ├── 20-uptime       # System uptime
│   ├── 30-load         # CPU load average
│   ├── 32-memory       # Memory usage
│   ├── 33-disk         # Disk usage
│   ├── 34-services     # Service status
│   ├── 35-docker       # Docker containers
│   ├── 36-raid         # RAID status
│   ├── 37-smart        # S.M.A.R.T. disk health
│   └── 41-updates      # Available updates
├── .github/
│   └── workflows/
│       └── lint.yml    # CI/CD linting
└── README.md
```

## Architecture

### Execution Flow

1. **motd.sh** - Entry point that:
   - Sets environment (`LANG="en_US.UTF-8"`)
   - Determines config path (default: `config.sh` in base dir)
   - Sources `framework.sh`
   - Discovers modules matching pattern `^(?<!\d)\d{2}(?!\d)-`
   - Executes each module in order
   - Columnizes and prints output

2. **Modules** execute independently:
   - Source `framework.sh`
   - Gather data from system
   - Use `print_columns()` to format output
   - Exit silently on errors (module won't display)

3. **framework.sh** provides:
   - Configuration loading
   - Color definitions
   - Formatting utilities
   - Text manipulation functions

## Key Conventions

### Module Naming
- **Pattern**: `NN-name` where NN is a two-digit number
- **Order**: Numeric prefix determines execution order
  - 10-19: User/system identity
  - 20-29: Uptime/time
  - 30-39: System resources & services
  - 40-49: Updates/packages
  - 50-59: Miscellaneous (quotes, etc.)
- **Disabling**: Rename to not match pattern (e.g., `30-load.disabled`)

### Module Structure Template

```bash
#!/usr/bin/env bash

set -euo pipefail
# shellcheck source=./framework.sh
source "${BASE_DIR}/framework.sh"

# Check for required commands
type required_command > /dev/null 2>&1 || exit 1

# Gather data
data=$(get_data)

# Format and print
print_columns "Label" "${data}"
```

### Bash Best Practices (REQUIRED)

1. **Shebang**: `#!/usr/bin/env bash`
2. **Error handling**: `set -euo pipefail`
3. **Shellcheck**: Add directives for false positives
4. **Quoting**: Always quote variables: `"${var}"`
5. **Arrays**: Use proper array syntax: `"${array[@]}"`
6. **Conditionals**: Use `[[ ]]` not `[ ]`
7. **Silent failures**: Modules exit 1 on errors (no output)
8. **Command checks**: Use `type cmd > /dev/null 2>&1 || exit 1`

## Framework Functions Reference

### Core Output Functions

#### `print_columns(label, text)`
Primary output function. Prints two-column format.
```bash
print_columns "OS" "Ubuntu 22.04"
# Output: OS:  Ubuntu 22.04
```

#### `print_bar(width, percentage)`
Displays visual progress bar.
```bash
print_bar 50 75
# Output: [====================               ]
```

#### `print_color(text, value, warning_threshold, error_threshold)`
Colors text based on thresholds (requires `bc`).
```bash
print_color "5.2" 5.2 4.0 8.0
# Returns green/yellow/red colored text
```

#### `print_status(label, status)`
Displays status with indicator (`▲` active, `▼` inactive).
```bash
print_status "nginx" "active"
# Output: ▲ nginx (in green)
```

#### `print_wrap(width, ...values)`
Wraps comma-separated values to specified width.
```bash
print_wrap "${WIDTH}" "${services[@]}"
```

#### `print_split(width, left_text, right_text)`
Justifies text left and right.
```bash
print_split 50 "Left text" "Right text"
```

#### `print_truncate(text, width, direction)`
Truncates text with ellipsis.
```bash
print_truncate "Long text here" 10 "end"
# Output: Long text…
```

### Utility Functions

#### `print_n(text, count)`
Repeats text N times.

#### `strip_ansi(text)`
Removes ANSI color codes from string.

#### `columnize(text, col_sep, row_sep)`
Internal function for column formatting (avoid direct use).

## Configuration System

### config.sh Variables

```bash
# Color Codes
CA="\e[34m"  # Accent (blue)
CO="\e[32m"  # OK (green)
CW="\e[33m"  # Warning (yellow)
CE="\e[31m"  # Error (red)
CN="\e[0m"   # None (reset)

# Layout
WIDTH=50     # Max width for second column components

# Module-specific (example: services)
declare -A services
services["nginx"]="Nginx"
services["docker"]="Docker"
```

### Environment Variables

- `BASE_DIR`: Auto-set to script directory
- `CONFIG_PATH`: Path to config file
- `LANG`: Set to "en_US.UTF-8" for predictable output

## Development Workflows

### Adding a New Module

1. **Create file**: `modules/NN-name` (choose appropriate number)
2. **Make executable**: `chmod +x modules/NN-name`
3. **Follow template**:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   source "${BASE_DIR}/framework.sh"

   # Your logic here
   print_columns "Label" "Value"
   ```
4. **Test locally**: `./motd.sh`
5. **Lint**: Ensure passes shellcheck and shfmt

### Modifying Framework Functions

**⚠️ Critical**: Framework changes affect ALL modules.

1. **Preserve backward compatibility**
2. **Test all modules** after changes
3. **Update this documentation**
4. **Consider color stripping** for width calculations

### Testing Changes

```bash
# Run MOTD
./motd.sh

# Test specific module
./modules/30-load

# Test with custom config
./motd.sh /path/to/test-config.sh

# Lint check
shellcheck motd.sh framework.sh modules/*
shfmt -i 4 -bn -ci -sr -d motd.sh framework.sh modules/*
```

## CI/CD Pipeline

### GitHub Actions Workflow (.github/workflows/lint.yml)

**Triggers**:
- Push to `master` or `dev`
- Pull requests
- Manual workflow dispatch

**Jobs**:

1. **Shellcheck**:
   - Options: `--enable avoid-nullary-conditions,require-variable-braces,require-double-brackets --external-sources`
   - Checks: `motd.sh`, `framework.sh`, all modules

2. **shfmt** (formatting):
   - Style: 4-space indent, binary ops at line start, case indent, space redirects
   - Command: `shfmt -i 4 -bn -ci -sr -d motd.sh framework.sh modules/*`

### Pre-commit Checklist

- [ ] shellcheck passes with project options
- [ ] shfmt formatting matches style
- [ ] All modules still work
- [ ] No hardcoded paths (use `${BASE_DIR}`)
- [ ] Error conditions exit cleanly
- [ ] Colors use config variables

## Common Patterns & Examples

### Pattern 1: Simple Data Display
```bash
#!/usr/bin/env bash
set -euo pipefail
source "${BASE_DIR}/framework.sh"

# Get data
value=$(cat /proc/sys/kernel/hostname)

print_columns "Hostname" "${value}"
```

### Pattern 2: Multiple Items with Wrapping
```bash
#!/usr/bin/env bash
set -euo pipefail
source "${BASE_DIR}/framework.sh"

items=()
for thing in $(get_things); do
    items+=("${thing}")
done

text=$(print_wrap "${WIDTH}" "${items[@]}")
print_columns "Label" "${text}"
```

### Pattern 3: Colored Status with Thresholds
```bash
#!/usr/bin/env bash
set -euo pipefail
source "${BASE_DIR}/framework.sh"

value=$(get_metric)
warning=80
error=95

colored=$(print_color "${value}%" "${value}" "${warning}" "${error}")
print_columns "Metric" "${colored}"
```

### Pattern 4: Service Status Check
```bash
#!/usr/bin/env bash
set -euo pipefail
source "${BASE_DIR}/framework.sh"

status=$(systemctl show -p ActiveState --value nginx)
formatted=$(print_status "Nginx" "${status}")
print_columns "Service" "${formatted}"
```

### Pattern 5: Progress Bar Display
```bash
#!/usr/bin/env bash
set -euo pipefail
source "${BASE_DIR}/framework.sh"

used=75
bar=$(print_bar 30 ${used})
print_columns "Usage" "${used}% ${bar}"
```

### Pattern 6: Array Data with IP Formatting (from 10-user)
```bash
users=()
while IFS= read -r line; do
    username=$(echo "$line" | awk '{print $1}')
    ip=$(echo "$line" | grep -oE '\([^)]+\)' | sed 's/[()]//g')
    if [[ -n "$ip" ]]; then
        users+=("${username} (${ip})")
    else
        users+=("${username}")
    fi
done < <(who)

print_columns "Users" "$(print_wrap "${WIDTH}" "${users[@]}")"
```

## Error Handling Guidelines

### Silent Failures
Modules should fail silently - if a module can't run, it simply doesn't display.

```bash
# Check for required command
type smartctl > /dev/null 2>&1 || exit 1

# Check for sudo permissions
sudo -n smartctl --version > /dev/null 2>&1 || exit 1

# Avoid errors from commands
result=$(risky_command 2> /dev/null || true)
```

### Defensive Checks
```bash
# Check if file exists
[[ -f /proc/loadavg ]] || exit 1

# Check if variable is set
[[ -n "${variable}" ]] || exit 1

# Check command output
output=$(command)
[[ -n "${output}" ]] || exit 1
```

## Git Workflow for AI Assistants

### Branch Strategy
- **Main branch**: `master`
- **Development**: Feature branches starting with `claude/`
- **Current branch**: `claude/claude-md-mi41hxrkaleaaolz-01STn4NoVnEukBJMPs4GCqb3`

### Commit Guidelines
1. **Format**: Conventional commits preferred
   - `feat:` - New features
   - `fix:` - Bug fixes
   - `style:` - Formatting (shellcheck/shfmt)
   - `ci:` - CI/CD changes
   - `docs:` - Documentation only

2. **Examples from history**:
   - `feat: Convert users to array with proper IP formatting`
   - `fix: Smart is showing nothing (#31)`
   - `ci: Run lint on PRs`
   - `style: Fix shellcheck in new modules`

### Push Requirements
- Always push to designated `claude/` branch
- Use: `git push -u origin <branch-name>`
- Network failures: Retry up to 4 times with exponential backoff (2s, 4s, 8s, 16s)

## Security Considerations

### Avoiding Command Injection
```bash
# BAD
result=$(ls $user_input)

# GOOD
result=$(ls "${user_input}")
```

### Sudo Usage
Some modules require sudo (e.g., smartctl):
```bash
# Check sudo works without password
sudo -n command --version > /dev/null 2>&1 || exit 1
```

Document sudo requirements in module comments.

## Performance Considerations

1. **Module independence**: Each module runs separately; slow modules don't block others
2. **Caching**: Not implemented; each run is fresh data
3. **Timeouts**: No explicit timeouts; rely on command completion
4. **Width awareness**: Use `${WIDTH}` for wrapping to maintain performance

## Troubleshooting Guide

### Module Not Appearing
1. Check file naming: Must match `^(?<!\d)\d{2}(?!\d)-` pattern
2. Make executable: `chmod +x modules/NN-name`
3. Check dependencies: Required commands available?
4. Test directly: `./modules/NN-name`
5. Check permissions: Sudo required?

### Formatting Issues
1. Ensure using `print_columns()`
2. Check ANSI code handling in width calculations
3. Test with `strip_ansi()` for visible character counting
4. Verify tab separator usage

### CI Failures
1. **Shellcheck**: Run locally with same options
2. **shfmt**: Run `shfmt -i 4 -bn -ci -sr -w <file>` to auto-fix
3. Check shellcheck directives are properly formatted

## Quick Reference Commands

```bash
# Run full MOTD
./motd.sh

# Test single module
./modules/30-load

# Lint everything
shellcheck --enable avoid-nullary-conditions,require-variable-braces,require-double-brackets --external-sources motd.sh framework.sh modules/*

# Format code
shfmt -i 4 -bn -ci -sr -w motd.sh framework.sh modules/*

# Find all modules
ls -1 modules/ | grep -P '^(?<!\d)\d{2}(?!\d)-'

# Check git status
git status

# Create new branch
git checkout -b claude/description

# Commit changes
git add <files>
git commit -m "type: description"

# Push changes
git push -u origin <branch-name>
```

## File-Specific Notes

### motd.sh:794
Main entry point. Sets environment, discovers modules, executes and formats output.

### framework.sh:5545
Core library. Contains all formatting and utility functions. Source this in every module.

### config.sh:343
User configuration. Can be overridden by passing path as argument to motd.sh.

## AI Assistant Best Practices

1. **Always lint**: Run shellcheck and shfmt before committing
2. **Test modules individually**: Don't assume they work without testing
3. **Preserve patterns**: Follow existing module patterns closely
4. **Check dependencies**: Verify required commands exist before using
5. **Document sudo needs**: If module needs sudo, document in comments
6. **Use framework functions**: Don't reinvent formatting functions
7. **Silent failures**: Modules should exit cleanly on errors
8. **Quote everything**: Shell quoting prevents many bugs
9. **Array handling**: Use proper array syntax with `[@]` and quotes
10. **Width awareness**: Use `${WIDTH}` variable for consistent formatting

## Recent Changes (as of latest commit)

- **3b4f859**: Users module converted to array with proper IP formatting
- **975d083**: Merged upstream/master, maintaining module deletions
- **a1a9e80**: Fixed S.M.A.R.T. module display issue
- **444ac47**: CI now runs lint on pull requests
- **b6cb7ab**: Shellcheck fixes for new modules
- **15aff53**: Added RAID and S.M.A.R.T. modules
- Deleted modules: 40-tmux, 50-quote

## Resources

- **ShellCheck**: https://www.shellcheck.net/
- **shfmt**: https://github.com/mvdan/sh
- **Original inspiration**: https://github.com/HermannBjorgvin/MOTD
- **Bash best practices**: Use shellcheck recommendations

---

**Last Updated**: 2025-11-18
**Maintained for**: AI assistants (Claude, GPT, etc.)
**Version**: 1.0
