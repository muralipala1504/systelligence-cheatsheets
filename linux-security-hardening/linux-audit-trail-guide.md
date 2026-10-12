# Linux Audit Trail Production Guide
# Linux Security Hardening Series — Episode 3
# youtube.com/@SysTelligence
# Verified on RHEL 10.2 and AlmaLinux 10.2

---

## Prerequisites — Install Before Using

### RHEL 10 and AlmaLinux 10
```bash
# auditd, ausearch, aureport — installed by default
# auditctl and augenrules require separate package on RHEL 10
sudo dnf install audit-rules

# audit2why and audit2allow require separate package
sudo dnf install policycoreutils-python-utils

# Verify all tools available
which auditd auditctl ausearch aureport augenrules audit2why audit2allow
```

### RHEL 8 and RHEL 9
```bash
# auditctl and augenrules included in audit package on RHEL 8/9
# Only policycoreutils-python-utils needed separately
sudo dnf install policycoreutils-python-utils
```

---

## auditd Status and Configuration

### Service Management
```bash
# Check auditd status
systemctl status auditd

# auditd is enabled and running by default on RHEL 8, 9, 10 and AlmaLinux

# IMPORTANT: Use service command to restart — not systemctl
# systemctl restart auditd FAILS by design (RefuseManualStop=yes)
# The kernel requires audit credentials when stopping auditd
sudo service auditd restart    # CORRECT
sudo service auditd reload     # CORRECT
# sudo systemctl restart auditd  # FAILS — do not use

# Check auditd version
auditctl -v    # correct flag
```

### auditd.conf Key Settings
```bash
# /etc/audit/auditd.conf — requires sudo to read on RHEL 10
sudo cat /etc/audit/auditd.conf
# or
sudo bat /etc/audit/auditd.conf

log_file = /var/log/audit/audit.log
max_log_file = 500              # MB before rotation
max_log_file_action = rotate    # rotate, keep_logs, or SYSLOG
num_logs = 5                    # number of rotated files to keep
space_left = 75                 # MB free before action
space_left_action = SYSLOG
admin_space_left = 50
admin_space_left_action = HALT
disk_full_action = SUSPEND
```

### Check Audit State
```bash
# Check current rules loaded (requires audit-rules package on RHEL 10)
sudo auditctl -l

# Check auditd statistics
sudo auditctl -s

# Check audit log size
duf /var/log/audit/
sudo du -sh /var/log/audit/audit.log*

# List rules directory (requires sudo on RHEL 10)
sudo ls /etc/audit/rules.d/
```

---

## Audit Rules

### Prerequisites
```bash
# Install audit-rules package first (RHEL 10 and AlmaLinux 10)
sudo dnf install audit-rules

# Verify auditctl available
sudo auditctl -l
```

### Audit Rule Syntax
```bash
# File watch rule
sudo auditctl -w /path/to/file -p permissions -k key_name

# Permissions:
# r = read
# w = write
# x = execute
# a = attribute change

# Syscall rule
sudo auditctl -a action,filter -S syscall -k key_name
```

### Essential File Watch Rules
```bash
# User account files
sudo auditctl -w /etc/passwd -p wa -k user-accounts
sudo auditctl -w /etc/shadow -p wa -k user-accounts
sudo auditctl -w /etc/group -p wa -k user-accounts
sudo auditctl -w /etc/gshadow -p wa -k user-accounts

# Privilege escalation files
sudo auditctl -w /etc/sudoers -p rwa -k sudoers-watch
sudo auditctl -w /etc/sudoers.d/ -p rwa -k sudoers-watch

# SSH configuration
sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd-config

# Audit log access
sudo auditctl -w /var/log/audit/ -p rwa -k audit-log-access

# Scheduled tasks
sudo auditctl -w /etc/cron.d/ -p wa -k cron-changes
sudo auditctl -w /etc/crontab -p wa -k cron-changes
sudo auditctl -w /var/spool/cron/ -p wa -k cron-changes

# System binaries
sudo auditctl -w /usr/bin/passwd -p x -k passwd-exec
sudo auditctl -w /usr/bin/sudo -p x -k sudo-exec
sudo auditctl -w /usr/sbin/useradd -p x -k user-mgmt
sudo auditctl -w /usr/sbin/userdel -p x -k user-mgmt
sudo auditctl -w /usr/sbin/usermod -p x -k user-mgmt

# Network configuration
sudo auditctl -w /etc/hosts -p wa -k network-config
sudo auditctl -w /etc/resolv.conf -p wa -k network-config
```

### Syscall Rules
```bash
# Permission changes
sudo auditctl -a always,exit -F arch=b64 -S chmod -S fchmod -k perm-changes
sudo auditctl -a always,exit -F arch=b64 -S chown -S fchown -k perm-changes

# Failed file access
sudo auditctl -a always,exit -F arch=b64 -S open -F exit=-EACCES -k access-denied
sudo auditctl -a always,exit -F arch=b64 -S open -F exit=-EPERM -k access-denied

# Module loading
sudo auditctl -a always,exit -F arch=b64 -S init_module -S delete_module -k modules

# Time changes
sudo auditctl -a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
```

### Make Rules Permanent
```bash
# Write rules to permanent file
sudo cat > /etc/audit/rules.d/production.rules << 'RULES'
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

# Load permanent rules (requires audit-rules package)
sudo augenrules --load

# Verify rules loaded
sudo auditctl -l
```

---

## ausearch — Query Audit Events

### Basic Queries
```bash
# Search by file path
sudo ausearch -f /etc/passwd
sudo ausearch -f /etc/sudoers

# Search by username
sudo ausearch -ua root
sudo ausearch -ua murali

# Search by UID
sudo ausearch -ui 1000

# Search by event type
sudo ausearch -m EXECVE
sudo ausearch -m USER_LOGIN
sudo ausearch -m SYSCALL
sudo ausearch -m AVC            # SELinux denials

# Search by rule key
sudo ausearch -k sudoers-watch
sudo ausearch -k user-accounts
sudo ausearch -k sshd-config

# Search by time
sudo ausearch --start today
sudo ausearch --start yesterday
sudo ausearch --start 10/01/2026
sudo ausearch --start 10/01/2026 --end 10/07/2026

# Search failed events
sudo ausearch -m USER_LOGIN --success no
sudo ausearch -m USER_AUTH --success no
```

### Advanced Queries
```bash
# Combine filters
sudo ausearch -f /etc/passwd --start today
sudo ausearch -ua root -m EXECVE --start yesterday

# Human readable output
sudo ausearch -f /etc/passwd -i

# Pipe to aureport
sudo ausearch -k sudoers-watch | sudo aureport -i --file

# Search SELinux denials
sudo ausearch -m AVC
sudo ausearch -m AVC --start today | sudo audit2why
```

---

## aureport — Summary Reports

```bash
# Overall summary
sudo aureport --summary

# Login report
sudo aureport -l

# Authentication report
sudo aureport -au

# Executable report
sudo aureport -x

# File access report
sudo aureport -f

# Failed events
sudo aureport --failed

# Date range
sudo aureport --start 10/01/2026 --end 10/07/2026 -l
```

---

## Forensic Workflow — Who Changed a File

```bash
# Step 1 — Find all events for the file
sudo ausearch -f /etc/nginx/nginx.conf -i

# Step 2 — Filter for write events
sudo ausearch -f /etc/nginx/nginx.conf -i | grep -E "open|write|truncate"

# Step 3 — Find auid (original login user — key accountability field)
sudo ausearch -f /etc/nginx/nginx.conf --raw | grep "auid="
# auid tracks original SSH login user even after sudo

# Step 4 — Find uid (effective user at time of action)
sudo ausearch -f /etc/nginx/nginx.conf --raw | grep "uid="
# uid=0 with auid=1000 means regular user sudoed to root

# Step 5 — Human readable interpretation
sudo ausearch -f /etc/nginx/nginx.conf -i --start today
```

---

## Privileged Command Tracking

```bash
# Who ran sudo today
sudo ausearch -k sudo-exec --start today -i

# Who switched users with su
sudo ausearch -m USER_AUTH --start today -i

# Who changed file permissions
sudo ausearch -k perm-changes --start today -i

# Who added or deleted users
sudo ausearch -k user-mgmt -i

# Who modified sudoers
sudo ausearch -k sudoers-watch -i

# Failed sudo attempts
sudo ausearch -k sudo-exec --success no -i
```

---

## SELinux and auditd Integration

```bash
# Requires policycoreutils-python-utils
sudo dnf install policycoreutils-python-utils

# Search SELinux denials
sudo ausearch -m AVC

# Today's SELinux denials with explanation
sudo ausearch -m AVC --start today | sudo audit2why

# Generate custom policy from denials
sudo ausearch -m AVC --start today | sudo audit2allow -M mypolicy
sudo semodule -i mypolicy.pp

# Search with ripgrep (fast — no sudo needed on log if readable)
sudo rg "type=AVC" /var/log/audit/audit.log
sudo rg "denied" /var/log/audit/audit.log
```

---

## Audit Log Management

```bash
# Manual rotation
sudo service auditd rotate

# Check rotated logs
ls -lh /var/log/audit/

# Search across rotated logs
sudo ausearch -f /etc/passwd --start 09/01/2026

# Remote logging plugin
sudo dnf install audispd-plugins

# Restart after config change
sudo service auditd restart    # use service not systemctl
```

### journalctl Integration (RHEL 9 and 10)
```bash
# auditd events in systemd journal
sudo journalctl -t audit

# Today's audit events
sudo journalctl -t audit --since today
```

---

## Quick Reference Card

| Task | Command | Notes |
|------|---------|-------|
| Install rule tools | sudo dnf install audit-rules | RHEL 10 only |
| Install SELinux tools | sudo dnf install policycoreutils-python-utils | All versions |
| Check auditd | systemctl status auditd | No sudo needed |
| List rules | sudo auditctl -l | Needs audit-rules |
| Add file watch | sudo auditctl -w /path -p wa -k key | Needs audit-rules |
| Search by file | sudo ausearch -f /path -i | |
| Search by user | sudo ausearch -ua username -i | |
| Search by key | sudo ausearch -k keyname -i | |
| Failed logins | sudo ausearch -m USER_LOGIN --success no -i | |
| Login summary | sudo aureport -l | |
| Load rules | sudo augenrules --load | Needs audit-rules |
| Restart auditd | sudo service auditd restart | NOT systemctl |

---

## RHEL Version Differences

| Feature | RHEL 8/9 | RHEL 10 |
|---------|----------|---------|
| auditctl | audit package | audit-rules package |
| augenrules | audit package | audit-rules package |
| audit2why | policycoreutils-python-utils | policycoreutils-python-utils |
| Restart method | service auditd restart | service auditd restart |
| Default running | Yes | Yes |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
