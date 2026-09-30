# SELinux Production Guide Cheatsheet
# Linux Security Hardening Series — Episode 1
# youtube.com/@SysTelligence

---

## SELinux Status and Modes
```bash
# Check current mode
getenforce

# Full status
sestatus

# Temporary switch to permissive (diagnostic only)
setenforce 0

# Switch back to enforcing
setenforce 1

# Permanent mode change — edit this file
vi /etc/selinux/config
# SELINUX=enforcing  (production)
# SELINUX=permissive (diagnostic)
# SELINUX=disabled   (avoid — requires reboot + relabel)
```

---

## SELinux Context — Check Labels
```bash
# Check file context
ls -laZ /var/www/html/

# Check directory context
ls -dZ /data/web

# Check process context
ps -eZ | grep nginx
ps -eZ | grep httpd

# Check context of specific file
ls -Z /etc/nginx/nginx.conf

# Check your own process context
id -Z
```

---

## Fix SELinux Context — Files and Directories

### Temporary Fix (Testing Only)
```bash
# Change context on single file
chcon -t httpd_sys_content_t /data/web/index.html

# Change context recursively on directory
chcon -R -t httpd_sys_content_t /data/web/

# Warning: chcon changes lost on filesystem relabel
```

### Permanent Fix (Production)
```bash
# Add custom context rule to policy
semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"

# Apply the rule to existing files
restorecon -Rv /data/web/

# Verify context applied
ls -laZ /data/web/

# List all custom context rules
semanage fcontext -l | grep local
```

### Common Context Types
```bash
# Web server content (readable by httpd)
httpd_sys_content_t

# Web server writable content
httpd_sys_rw_content_t

# Web server log files
httpd_log_t

# Web server scripts (CGI)
httpd_sys_script_exec_t

# SSH authorized keys
ssh_home_t

# User home directory
user_home_t

# System configuration
etc_t

# Container files
container_file_t
```

---

## Restore Default Context
```bash
# Restore context on file to policy default
restorecon -v /var/www/html/index.html

# Restore context recursively
restorecon -Rv /var/www/html/

# Restore context on entire filesystem (caution — slow)
restorecon -Rv /

# Trigger full relabel on next reboot
touch /.autorelabel
reboot
```

---

## SELinux Booleans
```bash
# List all booleans
getsebool -a

# List booleans with description
semanage boolean -l

# Check specific boolean
getsebool httpd_can_network_connect

# Enable boolean temporarily (resets on reboot)
setsebool httpd_can_network_connect on

# Enable boolean permanently
setsebool -P httpd_can_network_connect on

# Disable boolean permanently
setsebool -P httpd_can_network_connect off
```

### Common Production Booleans
```bash
# Allow web server outbound network connections
setsebool -P httpd_can_network_connect on

# Allow web server to connect to database
setsebool -P httpd_can_connect_db on

# Allow web server to connect to memcached
setsebool -P httpd_can_network_memcache on

# Allow FTP full access
setsebool -P ftpd_full_access on

# Allow Samba read/write
setsebool -P samba_export_all_rw on

# Allow NFS home directories
setsebool -P use_nfs_home_dirs on

# Allow containers to use devices
setsebool -P container_use_devices on
```

---

## SELinux Port Labels
```bash
# List all port labels
semanage port -l

# Check specific port
semanage port -l | grep http

# Add custom port to existing type
semanage port -a -t http_port_t -p tcp 8080

# Add another custom port
semanage port -a -t http_port_t -p tcp 3000

# Delete custom port
semanage port -d -t http_port_t -p tcp 8080

# Verify port added
semanage port -l | grep 8080
```

### Common Port Types
```bash
http_port_t      # 80, 443, 8008, 8080, 8443
ssh_port_t       # 22
mysqld_port_t    # 3306
postgresql_port_t # 5432
redis_port_t     # 6379
mongod_port_t    # 27017
```

---

## Reading and Diagnosing Denials
```bash
# Read audit log for AVC denials
grep AVC /var/log/audit/audit.log

# Read recent denials
grep AVC /var/log/audit/audit.log | tail -20

# Human readable denial explanation
grep AVC /var/log/audit/audit.log | audit2why

# Search by process name
ausearch -m avc -c nginx

# Search by time
ausearch -m avc --start recent

# Use sealert for detailed analysis
sealert -a /var/log/audit/audit.log

# journalctl for SELinux messages
journalctl -t setroubleshoot
```

---

## Generate Custom Policy with audit2allow
```bash
# View what policy would allow
grep AVC /var/log/audit/audit.log | audit2allow

# Generate policy module
grep AVC /var/log/audit/audit.log | audit2allow -M mypolicy

# Files created:
# mypolicy.te  — type enforcement source
# mypolicy.pp  — compiled policy package

# Install policy module
semodule -i mypolicy.pp

# List installed modules
semodule -l

# Remove module
semodule -r mypolicy

# View module contents
cat mypolicy.te
```

---

## Eight Step Troubleshooting Workflow
```bash
# Step 1 — Confirm SELinux mode
getenforce

# Step 2 — Check audit log for denials
grep AVC /var/log/audit/audit.log | tail -20

# Step 3 — Get human readable explanation
grep AVC /var/log/audit/audit.log | audit2why

# Step 4 — Switch to permissive to confirm SELinux is cause
setenforce 0

# Step 5 — Reproduce issue (if it works now, SELinux is confirmed)

# Step 6 — Switch back to enforcing immediately
setenforce 1

# Step 7 — Fix with semanage, boolean, or audit2allow
semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
restorecon -Rv /data/web/

# Step 8 — Verify fix in enforcing mode
grep AVC /var/log/audit/audit.log | tail -5
```

---

## Six Common Production Scenarios

### 1. nginx serves files from custom directory
```bash
# Symptom: permission denied on /data/web
# Fix:
semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
restorecon -Rv /data/web/
```

### 2. Web app cannot connect to database
```bash
# Symptom: connection refused to MySQL/PostgreSQL
# Fix:
setsebool -P httpd_can_connect_db on
```

### 3. App cannot bind to custom port
```bash
# Symptom: bind() failed on port 8080
# Fix:
semanage port -a -t http_port_t -p tcp 8080
```

### 4. App writes to custom log directory
```bash
# Symptom: permission denied on /var/log/myapp
# Fix:
semanage fcontext -a -t httpd_log_t "/var/log/myapp(/.*)?"
restorecon -Rv /var/log/myapp/
```

### 5. Container accessing host filesystem
```bash
# Symptom: container permission denied on host path
# Fix:
semanage fcontext -a -t container_file_t "/data/shared(/.*)?"
restorecon -Rv /data/shared/
```

### 6. Cron job silently failing
```bash
# Symptom: cron job runs but produces no output
# Fix: check audit log, generate custom policy
ausearch -m avc -c crond | audit2allow -M cronpolicy
semodule -i cronpolicy.pp
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Check mode | getenforce |
| Full status | sestatus |
| Permissive temporarily | setenforce 0 |
| Back to enforcing | setenforce 1 |
| Check file context | ls -laZ /path |
| Check process context | ps -eZ |
| Temp context fix | chcon -R -t type_t /path |
| Permanent context fix | semanage fcontext + restorecon |
| Check boolean | getsebool boolean_name |
| Enable boolean | setsebool -P boolean_name on |
| Add port label | semanage port -a -t type_t -p tcp PORT |
| Read denials | grep AVC /var/log/audit/audit.log |
| Explain denial | audit2why |
| Generate policy | audit2allow -M policyname |
| Install policy | semodule -i policy.pp |

---
*Never disable SELinux. Fix it.*
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
