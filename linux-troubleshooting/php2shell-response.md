# PHP2Shell Emergency Response Cheatsheet
# Linux Troubleshooting Series — Episode 10
# youtube.com/@SysTelligence

---

## 1. Detection: Check Your Exposure
```bash
# Check if PHP is installed and what version
rpm -qa | grep -E '^php'

# Check if Apache is using the dangerous mod_php
apachectl -M | grep php

# Check if PHP-FPM is running (safer, but still needs patching)
systemctl status php-fpm

Rule: If mod_php is loaded, PHP is running inside the Apache process. One engine crash kills the web server.

2. The Patch Race (Emergency Update)
The fix is in PHP 8.2.35, 8.3.36, 8.4.27, or 8.5.12.

# Update the runtime packages
sudo dnf update php-*

# CRITICAL: The RPM patch does nothing until the service restarts
sudo systemctl restart httpd   # If using mod_php
sudo systemctl restart php-fpm # If using FPM

3. Verification (Did the patch work?)
Do not rely on php -v (it shows CLI version). Check the RPM changelog and the running service.

# Check the RPM changelog for the CVE fix
rpm -q --changelog php-common | grep -E 'CVE-2026-10961[89]'

# Verify the restart time
systemctl status httpd
systemctl status php-fpm

4. Hunting Indicators of Compromise (IoC)
If you suspect the server was attacked before the patch:

# Check Apache logs for oversized headers (OOM trigger)
grep "Request Header Or Cookie Too Large" /var/log/httpd/error_log

# Check PHP logs for out of memory fatal errors
grep "Allowed memory size" /var/log/php-fpm/error.log

# Check audit logs for unexpected process spawns (web shell execution)
sudo ausearch -m EXECVE --start today | grep -E 'www-data|apache'

5. Emergency Mitigation (If you cannot patch immediately)

# Disable the session upload progress feature (blocks the UAF trigger)
# Edit /etc/php.ini and set:
session.upload_progress.enabled = 0

# Restart the service after editing
sudo systemctl restart httpd

Warning: Mitigation is a band-aid. You must patch eventually.

6. Architecture Hardening (Prevention)
Kill mod_php: Move to PHP-FPM. It isolates the PHP worker processes from the Apache web server.

Patch Discipline: Runtime RCEs require emergency patching in hours, not weeks. The weekend maintenance window is dead.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
