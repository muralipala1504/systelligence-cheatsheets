# Nginx and Apache Troubleshooting Cheatsheet
# Linux Troubleshooting Series — Episode 5
# youtube.com/@SysTelligence

---

## First Five Commands — Run in This Order
```bash
# 1. Check service state
systemctl status nginx

# 2. Confirm port is listening
ss -tulpn | grep :80

# 3. Test local response
curl -I localhost

# 4. Read error log
tail -50 /var/log/nginx/error.log

# 5. Check service journal
journalctl -u nginx --since "10 minutes ago"
```

---

## nginx — Complete Troubleshooting Reference

### Service Management
```bash
systemctl status nginx              # Check service state
systemctl start nginx               # Start service
systemctl stop nginx                # Stop service
systemctl restart nginx             # Full restart
systemctl reload nginx              # Graceful reload
systemctl enable nginx              # Enable on boot
```

### Configuration Testing
```bash
nginx -t                            # Test configuration
nginx -T                            # Test and dump full config
nginx -s reload                     # Reload after config test
nginx -t && systemctl reload nginx  # Safe reload — test first
nginx -v                            # Show nginx version
nginx -V                            # Show version and compile options
```

### Log Analysis
```bash
# Follow error log live
tail -f /var/log/nginx/error.log

# Last 100 error log lines
tail -100 /var/log/nginx/error.log

# Count status codes from access log
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Find all 502 errors with timestamp
awk '$9 == 502 {print $4, $7}' /var/log/nginx/access.log

# Find slow requests above 2 seconds
awk '$NF > 2.0 {print $7, $NF}' /var/log/nginx/access.log

# Top IPs hitting the server
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

### Port and Process Checks
```bash
# Check what is on port 80
ss -tulpn | grep :80

# Find process holding port 80
lsof -i :80

# Check all nginx processes
ps aux | grep nginx

# Check nginx master and worker PIDs
cat /var/run/nginx.pid
```

### SSL Certificate Troubleshooting
```bash
# Check certificate expiry
openssl x509 -in /path/to/cert.pem -noout -dates

# Check live certificate expiry
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates

# Verify certificate chain
openssl verify -CAfile chain.pem cert.pem

# Test SSL connection
openssl s_client -connect yourdomain.com:443

# Check certificate matches key
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa -noout -modulus -in key.pem | md5sum
# Both md5sum values must match
```

### Upstream Backend Troubleshooting
```bash
# Test backend directly
curl -I http://127.0.0.1:8080

# Check backend port listening
ss -tulpn | grep :8080

# Check backend service
systemctl status your-app

# Follow backend logs
journalctl -u your-app -f

# Test upstream from nginx perspective
curl -v --resolve yourdomain.com:80:127.0.0.1 http://yourdomain.com
```

### Permission Fixes
```bash
# Check nginx log directory ownership
ls -la /var/log/nginx/

# Fix log directory permissions
chown -R nginx:nginx /var/log/nginx/
chmod 755 /var/log/nginx/

# Check SSL certificate permissions
ls -la /etc/nginx/ssl/

# Fix certificate permissions
chown nginx:nginx /etc/nginx/ssl/cert.pem
chmod 640 /etc/nginx/ssl/cert.pem
```

### Resource Exhaustion Fixes
```bash
# Check current file descriptor limit
ulimit -n

# Check nginx worker connections
grep worker_connections /etc/nginx/nginx.conf

# Check OOM killer activity
journalctl -k | grep -i "oom\|killed process"

# Check memory usage
free -h
ps aux --sort=-%mem | head -10
```

---

## Apache — Complete Troubleshooting Reference

### Service Management
```bash
systemctl status apache2            # Debian/Ubuntu
systemctl status httpd              # RHEL/CentOS
apachectl status                    # Apache status
systemctl restart apache2           # Restart
systemctl reload apache2            # Graceful reload
```

### Configuration Testing
```bash
apachectl configtest                # Test configuration
apachectl -S                        # Show virtual host summary
apachectl -M                        # List loaded modules
apache2ctl -t                       # Alternative config test
httpd -t                            # RHEL equivalent
```

### Module Management (Debian/Ubuntu)
```bash
a2enmod rewrite                     # Enable mod_rewrite
a2enmod ssl                         # Enable SSL module
a2dismod autoindex                  # Disable module
a2ensite mysite.conf               # Enable virtual host
a2dissite mysite.conf              # Disable virtual host
a2enconf security                  # Enable config
```

### Virtual Host Troubleshooting
```bash
# Show all virtual hosts and their config files
apachectl -S

# Check which config file defines a site
apachectl -S | grep yourdomain.com

# Test specific virtual host config
apachectl -t -D DUMP_VHOSTS
```

### PHP-FPM Troubleshooting
```bash
# Check PHP-FPM status
systemctl status php-fpm
systemctl status php8.1-fpm         # Version specific

# Check PHP-FPM socket
ls -la /var/run/php/

# Check Apache PHP-FPM proxy config
grep -r "ProxyPassMatch\|SetHandler" /etc/apache2/

# Test PHP processing
echo "<?php phpinfo(); ?>" > /var/www/html/test.php
curl http://localhost/test.php | head -5
rm /var/www/html/test.php
```

### Log Analysis
```bash
# Apache error log
tail -f /var/log/apache2/error.log          # Debian
tail -f /var/log/httpd/error_log            # RHEL

# Apache access log
tail -f /var/log/apache2/access.log

# Count status codes
awk '{print $9}' /var/log/apache2/access.log | sort | uniq -c | sort -rn
```

---

## Ten Step Recovery Sequence
```bash
# Step 1 — Service state
systemctl status nginx

# Step 2 — Port listening
ss -tulpn | grep :80

# Step 3 — Local test
curl -I localhost

# Step 4 — Config test
nginx -t

# Step 5 — Error log
tail -50 /var/log/nginx/error.log

# Step 6 — Service journal
journalctl -u nginx -b

# Step 7 — Port conflict check
lsof -i :80

# Step 8 — Restart if config clean
nginx -t && systemctl restart nginx

# Step 9 — Verify recovery
curl -I localhost

# Step 10 — Monitor for recurrence
watch -n 2 'tail -5 /var/log/nginx/error.log'
```

---

## Prevention Checklist
```bash
# Safe config reload — never reload broken config
nginx -t && systemctl reload nginx

# Certificate expiry check — add to weekly cron
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# Prometheus nginx exporter alert rules
# nginx_up == 0 → page immediately
# rate(nginx_http_requests_total{status=~"5.."}[5m]) > 0.05 → warning

# Keep nginx config in git
cd /etc/nginx && git init && git add . && git commit -m "baseline config"
```

---

## Quick Reference Card

| Problem | First Command | Fix |
|---------|--------------|-----|
| Service down | systemctl status nginx | systemctl start nginx |
| Port not listening | ss -tulpn \| grep :80 | nginx -t && systemctl reload |
| Config error | nginx -t | Fix error at reported line |
| Port conflict | lsof -i :80 | Stop conflicting process |
| Permission error | ls -la /var/log/nginx | chown -R nginx:nginx |
| Upstream down | curl -I 127.0.0.1:8080 | Fix backend service |
| SSL expired | openssl x509 -noout -dates | certbot renew |
| OOM killed | journalctl -k \| grep oom | Free memory or add RAM |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
