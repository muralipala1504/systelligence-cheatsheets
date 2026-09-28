# fd and ripgrep Deep Dive Cheatsheet
# Linux Arsenal Series — Episode 5
# youtube.com/@SysTelligence

---

## fd — Complete Reference

### Install
```bash
# RHEL/CentOS/Fedora
dnf install fd-find

# Debian/Ubuntu
apt install fd-find
# Note: command may be fdfind — create alias
echo "alias fd='fdfind'" >> ~/.bashrc

# Verify
fd --version
```

### Basic Usage
```bash
fd filename                             # Search by name in current dir
fd filename /path                       # Search in specific path
fd "pattern"                           # Regex pattern search
fd --extension log /var/log            # Filter by extension
fd --type f /etc                       # Files only
fd --type d /var                       # Directories only
fd --type l /etc                       # Symlinks only
fd --type x /usr/bin                   # Executables only
```

### Time and Size Filters
```bash
fd --changed-before 7days /var/log     # Older than 7 days
fd --changed-after today /etc          # Modified today
fd --changed-after yesterday /var      # Modified since yesterday
fd --size +100m /var                   # Larger than 100MB
fd --size +1g /home                    # Larger than 1GB
fd --size -10k /etc                    # Smaller than 10KB
```

### Hidden and Ignored Files
```bash
fd --hidden .env                       # Include hidden files
fd --hidden --no-ignore node_modules   # Include gitignored dirs
fd --no-ignore vendor /opt/app         # Search vendor directory
fd -H -I "*.secret"                   # Short flags for hidden+no-ignore
```

### Advanced Filters
```bash
fd --owner root /etc                   # Files owned by root
fd --owner murali /home                # Files owned by specific user
fd "^config" /etc                     # Regex: files starting with config
fd "\.log$" /var                      # Regex: files ending with .log
fd --extension conf --extension yaml   # Multiple extensions
fd --type f --extension sh /opt        # Combine type and extension
```

### Exec and Batch Operations
```bash
fd --extension log --exec gzip {}      # Compress each file (parallel)
fd --extension log --exec-batch gzip   # Compress all at once
fd --extension conf --exec cat {}      # Print each config file
fd --extension py --exec wc -l {}     # Count lines in each Python file
fd --size +500m --exec rm {}          # Delete large files (careful!)
fd --extension log -x tail -5 {}      # Show last 5 lines of each log
```

### Production Sysadmin Tasks
```bash
# Find config files modified today
fd --changed-after today --extension conf /etc

# Find large log files for cleanup
fd --size +500m --extension log /var/log

# Find all shell scripts
fd --extension sh /opt /home

# Find SSL certificates
fd --extension pem /etc

# Find files owned by application user
fd --owner deploy /var/www

# Find broken symlinks
fd --type l --exec test -e {} \; -o -print

# Find executable scripts
fd --type x /opt/app

# Find recently modified configs during incident
fd --changed-after "10 minutes ago" /etc
```

### fd with fzf Integration
```bash
# Install fzf
dnf install fzf  # or apt install fzf

# Interactive file finder
fd | fzf

# Add alias
echo "alias ff='fd | fzf'" >> ~/.bashrc

# Open selected file in vim
vim $(fd | fzf)

# fzf preview with bat
export FZF_CTRL_T_OPTS="--preview 'bat --color=always {}'"
```

---

## ripgrep — Complete Reference

### Install
```bash
# RHEL/CentOS/Fedora
dnf install ripgrep

# Debian/Ubuntu
apt install ripgrep

# Verify
rg --version
```

### Basic Usage
```bash
rg pattern                             # Search in current directory
rg pattern /path                       # Search in specific path
rg pattern file.log                    # Search in specific file
rg -i pattern                          # Case insensitive
rg -n pattern                          # Show line numbers
rg -l pattern                          # List matching files only
rg -c pattern                          # Count matches per file
rg -v pattern                          # Invert match
```

### Type Filters
```bash
rg --type log "ERROR" /var/log         # Search only log files
rg --type py "import" /opt/app         # Search only Python files
rg --type sh "password" /etc           # Search only shell scripts
rg --type yaml "port" /etc             # Search only YAML files
rg --list-types                        # Show all built-in types

# Custom type definition
rg --type-add "conf:*.conf" --type conf "listen" /etc
```

### Context and Output
```bash
rg -A 3 "ERROR" app.log               # 3 lines after match
rg -B 2 "ERROR" app.log               # 2 lines before match
rg -C 3 "ERROR" app.log               # 3 lines before and after
rg --no-filename pattern               # Hide filenames
rg --with-filename pattern             # Always show filenames
rg -o pattern                          # Only print matched text
rg -m 5 pattern                        # Max 5 matches per file
```

### Log Analysis Commands
```bash
# Find all errors across log directory
rg ERROR /var/log

# Count errors per log file
rg -c ERROR /var/log

# Find slow requests above 2 seconds
rg --type log "response_time.*[2-9]\." /var/log

# Search only today's logs
rg ERROR $(fd --changed-after today --extension log /var/log)

# Find OOM killer events
rg "Out of memory|oom_kill" /var/log

# Find authentication failures
rg "Failed password|authentication failure" /var/log/auth.log

# Find service crashes
rg "segfault|core dumped" /var/log
```

### Advanced Features
```bash
# Search compressed files
rg -z "ERROR" /var/log/app.log.gz

# Multiline pattern
rg --multiline "start.*end" app.log

# Whole word match
rg -w "error" app.log

# Search gitignored directories
rg --no-ignore "TODO" /opt/app

# Show search statistics
rg --stats "ERROR" /var/log

# Fixed string search (no regex)
rg -F "192.168.1.1" /var/log

# Multiple patterns
rg "timeout|refused|failed" /var/log
```

### Security Audits
```bash
# Find hardcoded passwords
rg -i "password\s*=" /opt/app

# Find API keys
rg "api_key|apikey|api-key" /opt/app

# Find hardcoded IPs
rg "\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b" /opt/app

# Find sudo usage in scripts
rg "sudo" --type sh /opt

# Find world-writable references
rg "chmod 777|chmod o+w" --type sh /opt
```

---

## fd + ripgrep Combined Workflows

```bash
# Search all config files for SSL paths
fd --extension conf /etc | xargs rg ssl_certificate

# Search recent logs for errors
fd --changed-after yesterday --extension log /var/log | xargs rg ERROR

# Audit Python code for deprecated functions
fd --extension py /opt/app | xargs rg "deprecated_function"

# Security audit for hardcoded credentials
fd --extension sh --extension py --extension conf | xargs rg -i "password="

# Search all YAML configs for specific port
fd --extension yaml --extension yml /etc | xargs rg "port: 8080"

# Find and search modified configs during incident
fd --changed-after "1 hour ago" --extension conf /etc | xargs rg -l "."
```

---

## Aliases — Add to ~/.bashrc
```bash
# Replace find with fd
alias find='fd'

# Quick search
alias ff='fd | fzf'

# Search logs fast
alias logsearch='rg --type log'

# Audit credentials
alias credcheck='fd --extension sh --extension py --extension conf | xargs rg -i "password=\|secret=\|api_key="'
```

---

## Quick Reference Card

| Task | find/grep | fd/rg |
|------|-----------|-------|
| Find by extension | find /path -name "*.log" | fd --extension log /path |
| Find by size | find /path -size +100M | fd --size +100m /path |
| Find by time | find /path -mtime +7 | fd --changed-before 7days /path |
| Search in files | grep -r "pattern" /path | rg "pattern" /path |
| Count matches | grep -rc "pattern" | rg -c "pattern" |
| Context lines | grep -A 3 "pattern" | rg -A 3 "pattern" |
| Type filter | grep -r --include="*.log" | rg --type log |
| Compressed search | zgrep "pattern" file.gz | rg -z "pattern" file.gz |
| Find + search | find \| xargs grep | fd \| xargs rg |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
