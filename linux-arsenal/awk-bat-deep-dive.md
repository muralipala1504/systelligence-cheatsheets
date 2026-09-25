# awk and bat Deep Dive Cheatsheet
# Linux Arsenal Series — Episode 3
# youtube.com/@SysTelligence

---

## awk — Complete Reference

### Basic Syntax
```bash
awk 'pattern { action }' filename
awk -F: 'pattern { action }' filename    # Set field separator
awk -v var=value 'pattern { action }'   # Pass variable
awk 'pattern { action }' file1 file2    # Multiple files
```

### Field Processing
```bash
awk '{ print $1 }' file                 # Print first field
awk '{ print $1, $4 }' file            # Print fields 1 and 4
awk '{ print $NF }' file               # Print last field
awk '{ print NR, $0 }' file            # Print line number and line
awk -F: '{ print $1 }' /etc/passwd     # Colon separated
awk -F, '{ print $2 }' file.csv        # CSV processing
awk -F'\t' '{ print $3 }' file.tsv     # Tab separated
```

### Pattern Matching
```bash
awk '$9 == 404' access.log              # Exact field match
awk '$9 != 200' access.log             # Not equal
awk '$5 > 1.0' access.log              # Numeric comparison
awk '$5 >= 2.0 && $9 == 500' access.log # Multiple conditions
awk '/ERROR/' logfile                   # Pattern anywhere in line
awk '!/ERROR/' logfile                  # Lines NOT matching
awk '$1 ~ /192\.168/' access.log       # Regex on field
awk '$1 !~ /192\.168/' access.log      # Regex NOT on field
awk 'NR > 10 && NR < 20' file          # Line range
```

### Log Analysis
```bash
# Count HTTP status codes
awk '{ print $9 }' access.log | sort | uniq -c | sort -rn

# Average response time
awk '{ sum += $5; count++ } END { print sum/count }' access.log

# Top 10 IPs by request count
awk '{ print $1 }' access.log | sort | uniq -c | sort -rn | head -10

# Find slow requests above 2 seconds
awk '$5 > 2.0 { print $1, $7, $5 }' access.log

# Count errors by type
awk '$9 >= 400 { errors[$9]++ } END { for (e in errors) print e, errors[e] }' access.log

# Find top endpoints by request count
awk '{ print $7 }' access.log | sort | uniq -c | sort -rn | head -10
```

### BEGIN and END Blocks
```bash
# Complete log report
awk '
BEGIN {
    FS = " "
    print "=== Log Analysis Report ==="
}
{
    total++
    if ($9 >= 400) errors++
    response_time += $5
}
END {
    print "Total Requests:", total
    print "Error Count:", errors
    print "Error Rate:", (errors/total)*100 "%"
    print "Avg Response Time:", response_time/total "s"
}' access.log
```

### Production One-Liners
```bash
# Remove duplicates preserving order
awk '!seen[$0]++' file

# Print lines between two patterns
awk '/START/,/END/' file

# Sum a column
awk '{ sum += $3 } END { print sum }' file

# Print every nth line (every 10th)
awk 'NR % 10 == 0' file

# Add line numbers
awk '{ print NR": "$0 }' file

# Print lines longer than 80 chars
awk 'length > 80' file

# Replace field value
awk '$3 == "old" { $3 = "new" } { print }' file

# Print unique values from column
awk '!seen[$1]++{ print $1 }' file
```

---

## bat — Complete Reference

### Basic Usage
```bash
bat filename                            # View file with highlighting
bat file1 file2                         # View multiple files
bat *.conf                              # View all conf files
bat -n filename                         # Line numbers only
bat -p filename                         # Plain output no decorations
bat -A filename                         # Show non-printable characters
bat --paging never filename             # Disable paging
bat --line-range 50:100 filename        # View specific line range
bat --line-range :50 filename           # First 50 lines
bat --line-range 100: filename          # From line 100 to end
```

### Language and Theme
```bash
bat -l nginx filename                   # Force nginx syntax
bat -l json filename                    # Force JSON syntax
bat -l yaml filename                    # Force YAML syntax
bat --list-languages                    # Show all supported languages
bat --list-themes                       # Show all available themes
bat --theme=TwoDark filename            # Use specific theme
```

### Config File Review
```bash
bat /etc/nginx/nginx.conf               # Nginx config
bat /etc/ssh/sshd_config               # SSH config
bat /etc/fstab                          # Mount table
bat /etc/hosts                          # Hosts file
bat /etc/crontab                        # Cron jobs
bat Dockerfile                          # Dockerfile
bat docker-compose.yml                  # Docker Compose
bat /etc/systemd/system/app.service    # Systemd unit file
```

### Log File Viewing
```bash
bat /var/log/syslog                     # System log
bat /var/log/nginx/error.log           # Nginx errors
bat --line-range 100:150 /var/log/app.log  # Specific lines
bat -A /etc/hosts                       # Show non-printable chars
tail -f /var/log/app.log | bat --paging never  # Live colored log
```

### Integrations
```bash
# Colored git diff
git diff | bat

# Colored man pages
export MANPAGER="sh -c 'col -bx | bat -l man -p'"

# Replace cat permanently
echo "alias cat='bat'" >> ~/.bashrc
source ~/.bashrc

# fzf preview with bat
export FZF_CTRL_T_OPTS="--preview 'bat --color=always {}'"

# bat as PAGER
export PAGER="bat"
```

### Install on Kickstart
```bash
%packages
bat
%end
# Note: On Debian/Ubuntu package is batcat
# Add alias: echo "alias bat='batcat'" >> ~/.bashrc
```

---

## Quick Reference Card

| Task | awk | bat |
|------|-----|-----|
| Print field | awk '{ print $2 }' | — |
| Filter lines | awk '$9 == 404' | — |
| Count occurrences | awk '{ c[$1]++ } END { for (k in c) print c[k], k }' | — |
| View file | — | bat filename |
| Line range | awk 'NR>=10 && NR<=20' | bat --line-range 10:20 |
| No decoration | — | bat -p filename |
| Show hidden chars | — | bat -A filename |
| Live log | — | tail -f log \| bat --paging never |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
