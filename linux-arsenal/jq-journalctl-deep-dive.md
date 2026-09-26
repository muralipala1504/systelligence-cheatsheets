# jq and journalctl Deep Dive Cheatsheet
# Linux Arsenal Series — Episode 4
# youtube.com/@SysTelligence

---

## jq — Complete Reference

### Basic Syntax
```bash
cat file.json | jq .                    # Pretty print entire JSON
cat file.json | jq .fieldname          # Extract specific field
cat file.json | jq -r .fieldname       # Raw output without quotes
cat file.json | jq .a.b.c             # Nested field access
cat file.json | jq .[]                 # Iterate array elements
cat file.json | jq '.[0]'             # First array element
cat file.json | jq keys               # List all field names
cat file.json | jq length             # Count array elements
```

### Field Extraction
```bash
# Single field
jq .level app.log

# Multiple fields as new object
jq '{time: .timestamp, lvl: .level, msg: .message}' app.log

# Multiple fields as array
jq '[.timestamp, .level, .message]' app.log

# Raw multiple fields
jq -r '.timestamp + " " + .level + " " + .message' app.log

# Nested field
jq .request.headers.host app.log

# All keys in object
jq keys app.log

# All values in object
jq values app.log
```

### select Filters
```bash
# Filter by exact field value
jq 'select(.level == "ERROR")' app.log

# Numeric comparison
jq 'select(.response_time > 1.0)' app.log

# Greater than or equal
jq 'select(.status_code >= 400)' app.log

# Multiple conditions AND
jq 'select(.level == "ERROR" and .service == "auth")' app.log

# Multiple conditions OR
jq 'select(.level == "ERROR" or .level == "CRITICAL")' app.log

# Regex match on field
jq 'select(.message | test("timeout"))' app.log

# Field exists check
jq 'select(.user_id != null)' app.log

# Not equal
jq 'select(.status_code != 200)' app.log
```

### Log Analysis
```bash
# Count errors by service
jq -r '.service' app.log | sort | uniq -c | sort -rn

# Average response time
jq '.response_time' app.log | jq -s 'add/length'

# Top 10 error messages
jq -r 'select(.level=="ERROR") | .message' app.log | sort | uniq -c | sort -rn | head -10

# Unique user IDs from errors
jq -r 'select(.level=="ERROR") | .user_id' app.log | sort -u

# Count by status code
jq -r '.status_code' app.log | sort | uniq -c | sort -rn

# Find slowest requests
jq 'select(.response_time > 2.0) | {endpoint: .path, time: .response_time}' app.log

# Error rate calculation
jq -s 'length as $total | map(select(.level=="ERROR")) | length / $total * 100' app.log
```

### Transforms and Output
```bash
# Reshape JSON — keep only needed fields
jq '{timestamp, level, message}' app.log

# Export as CSV
jq -r '[.timestamp, .level, .message] | @csv' app.log

# Export as TSV
jq -r '[.timestamp, .level, .message] | @tsv' app.log

# Add calculated field
jq '. + {slow: (.response_time > 1.0)}' app.log

# Process multiple files
jq '.level' file1.log file2.log file3.log

# Slurp multiple lines into array
jq -s '.' app.log

# Compact output no whitespace
jq -c '.' app.log
```

### Pipeline Integrations
```bash
# Kubernetes pod logs
kubectl logs pod-name | jq .

# Kubernetes resource status
kubectl get pods -o json | jq '.items[].status.phase'

# Docker container logs
docker logs container-name | jq 'select(.level=="ERROR")'

# AWS CLI output
aws ec2 describe-instances | jq '.Reservations[].Instances[] | {id: .InstanceId, state: .State.Name}'

# AWS S3 list
aws s3api list-buckets | jq '.Buckets[].Name'

# curl API response
curl -s https://api.endpoint.com | jq .data.items

# Terraform output
terraform output -json | jq .instance_ip.value

# Explore unknown JSON structure
curl -s api-endpoint | jq keys
curl -s api-endpoint | jq '.data | keys'
```

---

## journalctl — Complete Reference

### Basic Usage
```bash
journalctl                              # All journal entries
journalctl -f                           # Follow live
journalctl -e                           # Jump to end
journalctl -n 50                        # Last 50 lines
journalctl -r                           # Reverse order newest first
journalctl --no-pager                   # No paging output
journalctl -o json                      # JSON output format
journalctl -o json-pretty              # Pretty JSON output
```

### Service Filters
```bash
journalctl -u nginx                     # Nginx service logs
journalctl -u nginx -f                  # Follow nginx logs live
journalctl -u nginx --since today       # Nginx logs since midnight
journalctl -u nginx -u php-fpm         # Multiple services
journalctl _SYSTEMD_UNIT=app.service   # Exact unit match
```

### Priority Filters
```bash
journalctl -p err                       # Error and above
journalctl -p warning                   # Warning and above
journalctl -p debug                     # All including debug
journalctl -p 0..3                      # Emergency to error range
# Priorities: 0=emerg 1=alert 2=crit 3=err 4=warning 5=notice 6=info 7=debug
```

### Time Filters
```bash
journalctl --since today                # Since midnight
journalctl --since yesterday            # Since yesterday midnight
journalctl --since "2026-09-20 14:00"  # Specific timestamp
journalctl --since "2026-09-20 14:00" --until "2026-09-20 15:00"  # Time window
journalctl --since "1 hour ago"        # Relative time
journalctl --since "10 minutes ago"    # Last 10 minutes
```

### Boot Filters
```bash
journalctl -b                           # Current boot
journalctl -b -1                        # Previous boot
journalctl -b -2                        # Two boots ago
journalctl --list-boots                 # List all boot sessions
journalctl -k                           # Kernel messages current boot
journalctl -k -b -1                     # Kernel messages previous boot
```

### Process and User Filters
```bash
journalctl _PID=1234                    # Specific process ID
journalctl _UID=1000                    # Specific user ID
journalctl _GID=1000                    # Specific group ID
journalctl _COMM=nginx                  # By process name
journalctl _EXE=/usr/bin/python3       # By executable path
```

### Crash and Boot Analysis
```bash
# Unexpected reboot diagnosis
journalctl -b -1 -p err

# Service crash analysis
journalctl -u app -b | tail -50

# Kernel panic previous boot
journalctl -k -b -1

# OOM killer activity
journalctl -k | grep -i "oom\|killed process"

# Failed systemd units
systemctl --failed
journalctl -u failed-service -b
```

### Disk Management
```bash
journalctl --disk-usage                 # Show journal size

# Vacuum by size
journalctl --vacuum-size=500M          # Keep under 500MB

# Vacuum by time
journalctl --vacuum-time=30d           # Keep last 30 days

# Vacuum by file count
journalctl --vacuum-files=5            # Keep 5 most recent files
```

### journald.conf — Permanent Configuration
```bash
# Edit /etc/systemd/journald.conf
SystemMaxUse=500M           # Max journal disk usage
SystemKeepFree=1G           # Min free space to maintain
MaxRetentionSec=25d         # Max log retention
MaxFileSec=1week            # Max per journal file
Compress=yes                # Compress journal files

# Apply changes
systemctl restart systemd-journald
```

### jq plus journalctl Combined
```bash
# Filter critical entries from service
journalctl -u app -o json | jq 'select(.PRIORITY == "3")'

# Extract messages only
journalctl -o json -b | jq -r '.MESSAGE'

# Regex on journal messages
journalctl -u nginx -o json | jq 'select(.MESSAGE | test("upstream"))'

# Export errors as CSV
journalctl -p err -o json | jq -r '[.__REALTIME_TIMESTAMP, .SYSLOG_IDENTIFIER, .MESSAGE] | @csv'

# Count errors by service
journalctl -p err -o json | jq -r '.SYSLOG_IDENTIFIER' | sort | uniq -c | sort -rn
```

---

## Quick Reference Card

| Task | jq | journalctl |
|------|-----|------------|
| Pretty print | jq . file | journalctl -e |
| Extract field | jq .field file | — |
| Filter by value | jq 'select(.f=="v")' | journalctl -u service |
| Errors only | jq 'select(.level=="ERROR")' | journalctl -p err |
| Time window | — | journalctl --since --until |
| Previous boot | — | journalctl -b -1 |
| Live follow | — | journalctl -f -u service |
| Disk usage | — | journalctl --disk-usage |
| Cleanup | — | journalctl --vacuum-size=500M |
| JSON output | jq . | journalctl -o json |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
