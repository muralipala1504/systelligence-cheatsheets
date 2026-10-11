# Linux Audit Trail Production Guide
# Linux Security Hardening Series — Episode 3
# youtube.com/@SysTelligence
# Verified on RHEL 10.2 and AlmaLinux 10.2

---

## auditd Status and Configuration

### Service Management
```bash
# Check auditd status
systemctl status auditd

# auditd is enabled by default on RHEL 8, 9, 10 and AlmaLinux
# Verify it is running on every server
systemctl is-enabled auditd
systemctl is-active auditd

# Restart after config changes
systemctl restart auditd
```

### auditd.conf Key Settings
```bash
# /etc/audit/auditd.conf
log_file = /var/log/audit/audit.log
max_log_file = 500              # MB before rotation
max_log_file_action = rotate    # rotate, keep_logs, or SYSLOG
num_logs = 5                    # number of rotated files to keep
space_left = 75                 # MB free before action
space_left_action = SYSLOG      # what to do when space_left reached
admin_space_left = 50           # MB — critical threshold
admin_space_left_action = HALT  # halt system if critically low
disk_full_action = SUSPEND      # suspend logging if disk full

# Check current config
bat /etc/audit/auditd.conf
```

### Check Audit Log Size
```bash
# Check audit log disk usage
duf /var/log/audit/

# Detailed size
du -sh /var/log/audit/
du -sh /var/log/audit/audit.log*

# Check current rules loaded
auditctl -l

# Check auditd statistics
auditctl -s
```

---

## Audit Rules

### Audit Rule Syntax
```bash
# File watch rule syntax
auditctl -w /path/to/file -p permissions -k key_name

# Permissions:
# r = read
# w = write
# x = execute
# a = attribute change

# Syscall rule syntax
auditctl -a action,filter -S syscall -k key_name
```

### Essential File Watch Rules
```bash
# User account files
auditctl -w /etc/passwd -p wa -k user-accounts
auditctl -w /etc/shadow -p wa -k user-accounts
auditctl -w /etc/group -p wa -k user-accounts
auditctl -w /etc/gshadow -p wa -k user-accounts

# Privilege escalation files
auditctl -w /etc/sudoers -p rwa -k sudoers-watch
auditctl -w /etc/sudoers.d/ -p rwa -k sudoers-watch

# SSH configuration
auditctl -w /etc/ssh/sshd_config -p wa -k sshd-config

# Audit log access
auditctl -w /var/log/audit/ -p rwa -k audit-log-access

# Scheduled tasks
auditctl -w /etc/cron.d/ -p wa -k cron-changes
auditctl -w /etc/crontab -p wa -k cron-changes
auditctl -w /var/spool/cron/ -p wa -k cron-changes

# System binaries
auditctl -w /usr/bin/passwd -p x -k passwd-exec
auditctl -w /usr/bin/sudo -p x -k sudo-exec
auditctl -w /usr/sbin/useradd -p x -k user-mgmt
auditctl -w /usr/sbin/userdel -p x -k user-mgmt
auditctl -w /usr/sbin/usermod -p x -k user-mgmt

# Network configuration
auditctl -w /etc/hosts -p wa -k network-config
auditctl -w /etc/resolv.conf -p wa -k network-config
```

### Syscall Rules
```bash
# Permission changes
auditctl -a always,exit -F arch=b64 -S chmod -S fchmod -k perm-changes
auditctl -a always,exit -F arch=b64 -S chown -S fchown -k perm-changes

# Failed file access
auditctl -a always,exit -F arch=b64 -S open -F exit=-EACCES -k access-denied
auditctl -a always,exit -F arch=b64 -S open -F exit=-EPERM -k access-denied

# Module loading
auditctl -a always,exit -F arch=b64 -S init_module -S delete_module -k modules

# Time changes
auditctl -a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
```

### Make Rules Permanent
```bash
# Write rules to permanent file
cat > /etc/audit/rules.d/production.rules << 'RULES'
# User account monitoring
-w /etc/passwd -p wa -k user-accounts
-w /etc/shadow -p wa -k user-accounts
-w /etc/sudoers -p rwa -k sudoers-watch
-w /etc/sudoers.d/ -p rwa -k sudoers-watch

# SSH monitoring
-w /etc/ssh/sshd_config -p wa -k sshd-config

# Privilege escalation
-w /usr/bin/sudo -p x -k sudo-exec

# Cron monitoring
-w /etc/cron.d/ -p wa -k cron-changes
-w /etc/crontab -p wa -k cron-changes

# Permission changes
-a always,exit -F arch=b64 -S chmod -S chown -k perm-changes
RULES

# Load permanent rules
augenrules --load

# Verify rules loaded
auditctl -l
```

---

## ausearch — Query Audit Events

### Basic Queries
```bash
# Search by file path
ausearch -f /etc/passwd
ausearch -f /etc/sudoers

# Search by username
ausearch -ua root
ausearch -ua murali

# Search by UID
ausearch -ui 1000

# Search by event type
ausearch -m EXECVE
ausearch -m USER_LOGIN
ausearch -m SYSCALL
ausearch -m AVC            # SELinux denials

# Search by rule key
ausearch -k sudoers-watch
ausearch -k user-accounts
ausearch -k sshd-config

# Search by time
ausearch --start today
ausearch --start yesterday
ausearch --start 10/01/2026
ausearch --start 10/01/2026 --end 10/07/2026

# Search failed events
ausearch -m USER_LOGIN --success no
ausearch -m USER_AUTH --success no
```

### Advanced Queries
```bash
# Combine filters
ausearch -f /etc/passwd --start today
ausearch -ua root -m EXECVE --start yesterday

# Human readable output
ausearch -f /etc/passwd -i

# Raw output
ausearch -f /etc/passwd --raw

# Pipe to aureport for interpretation
ausearch -k sudoers-watch | aureport -i --file

# Search SELinux denials
ausearch -m AVC
ausearch -m AVC --start today | audit2why
```

---

## aureport — Summary Reports

### Standard Reports
```bash
# Overall summary
aureport --summary

# Login report
aureport -l
aureport --login

# Authentication report
aureport -au
aureport --auth

# Executable report
aureport -x
aureport --executable

# File access report
aureport -f
aureport --file

# Failed events report
aureport --failed

# Anomaly report
aureport --anomaly
```

### Date Range Reports
```bash
# Report for today
aureport --start today --summary

# Report for specific date range
aureport --start 10/01/2026 --end 10/07/2026 -l

# Last 24 hours authentication
aureport --start yesterday --auth
```

---

## Forensic Workflow — Who Changed a File

### Five Step Process
```bash
# Step 1 — Find all events for the file
ausearch -f /etc/nginx/nginx.conf -i

# Step 2 — Filter for write events
ausearch -f /etc/nginx/nginx.conf -i | grep -E "open|write|truncate"

# Step 3 — Find auid (original login user)
ausearch -f /etc/nginx/nginx.conf --raw | grep "auid="
# auid=0 means root, auid=1000 means regular user

# Step 4 — Find uid (effective user at time of action)
ausearch -f /etc/nginx/nginx.conf --raw | grep "uid="

# Step 5 — Interpret with human readable output
ausearch -f /etc/nginx/nginx.conf -i --start today

# Key: auid tracks original SSH login user even after sudo
# If uid=0 but auid=1000, regular user sudoed to root
```

---

## Privileged Command Tracking

```bash
# Who ran sudo today
ausearch -k sudo-exec --start today -i

# Who switched users with su
ausearch -m USER_AUTH --start today -i

# Who changed file permissions
ausearch -k perm-changes --start today -i

# Who added or deleted users
ausearch -k user-mgmt -i

# Who modified sudoers
ausearch -k sudoers-watch -i

# Failed sudo attempts
ausearch -k sudo-exec --success no -i
```

---

## SELinux and auditd Integration

```bash
# Search SELinux denials
ausearch -m AVC

# Today's SELinux denials
ausearch -m AVC --start today

# Human readable SELinux denial explanation
ausearch -m AVC --start today | audit2why

# Generate custom policy from denials
ausearch -m AVC --start today | audit2allow -M mypolicy
semodule -i mypolicy.pp

# Search with ripgrep (fast)
rg "type=AVC" /var/log/audit/audit.log
rg "denied" /var/log/audit/audit.log
```

---

## Audit Log Management

### Log Rotation
```bash
# Manual rotation
service auditd rotate

# Check rotated logs
ls -lh /var/log/audit/

# Search across rotated logs
ausearch -f /etc/passwd --start 09/01/2026

# Search compressed logs
zcat /var/log/audit/audit.log.3.gz | ausearch -i -f /etc/passwd
```

### Remote Logging (audisp-remote)
```bash
# Install audisp-remote plugin
dnf install audispd-plugins

# Configure remote logging
# /etc/audit/plugins.d/au-remote.conf
active = yes
direction = out
path = /sbin/audisp-remote
type = always

# /etc/audisp/audisp-remote.conf
remote_server = siem.company.com
port = 60

# Restart auditd
systemctl restart auditd
```

### journalctl Integration (RHEL 9 and 10)
```bash
# auditd events in systemd journal
journalctl -t audit

# Today's audit events
journalctl -t audit --since today

# Combined audit and service logs
journalctl -t audit -u nginx --since today
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Check auditd | systemctl status auditd |
| List rules | auditctl -l |
| Add file watch | auditctl -w /path -p wa -k keyname |
| Search by file | ausearch -f /path/to/file -i |
| Search by user | ausearch -ua username -i |
| Search by key | ausearch -k keyname -i |
| Failed logins | ausearch -m USER_LOGIN --success no -i |
| Login summary | aureport -l |
| Auth summary | aureport --auth |
| Command summary | aureport -x |
| SELinux denials | ausearch -m AVC --start today |
| Load rules | augenrules --load |
| Verify auid | ausearch -f /path --raw | grep auid |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
