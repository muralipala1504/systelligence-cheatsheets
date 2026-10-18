# Fail2ban SSH Protection Cheatsheet
# Linux Security Hardening Series — Episode 4
# youtube.com/@SysTelligence

---

## Install on RHEL 10
```bash
# Fail2ban is not in default repos. EPEL is required.
dnf install epel-release
dnf install fail2ban

# Enable and start
systemctl enable --now fail2ban
systemctl status fail2ban

Configuration Hierarchy
/etc/fail2ban/jail.conf — Package default. NEVER EDIT.

/etc/fail2ban/jail.local — Your overrides. EDIT HERE.

/etc/fail2ban/jail.d/*.local — Drop-in fragments.

The sshd Jail (Production Baseline)
Edit /etc/fail2ban/jail.local:

[sshd]
enabled = true
port = ssh
logpath = /var/log/secure  # RHEL uses secure, NOT auth.log
maxretry = 3
findtime = 10m
bantime = 24h

Whitelisting (Ignore IP)
In the [DEFAULT] section of jail.local:
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 203.0.113.50 10.0.0.0/8

Status and Verification
# List all active jails
fail2ban-client status

# Check specific jail (banned IPs, counts)
fail2ban-client status sshd

# Get banned IPs in a raw list
fail2ban-client get sshd banip

Manual Ban and Unban

# Ban an IP immediately
fail2ban-client set sshd banip 198.51.100.77

# Unban an IP
fail2ban-client set sshd unbanip 198.51.100.77

# Unban across all jails
fail2ban-client unban 198.51.100.77

# Watch actions live
tail -f /var/log/fail2ban.log

Testing Filters
# Always test before enabling a new jail
fail2ban-regex /var/log/secure /etc/fail2ban/filter.d/sshd.conf

Custom Jail for Web App
Filter (/etc/fail2ban/filter.d/myapp.conf):

[Definition]
failregex = ^.*LOGIN_FAILED ip=<HOST>.*$
ignoreregex =

Jail (/etc/fail2ban/jail.local):

[myapp]
enabled = true
port = http,https
filter = myapp
logpath = /var/log/myapp/auth.log
maxretry = 5
bantime = 1h

Recidive Jail (Repeat Offenders)
Watch the fail2ban log itself to ban repeat offenders.
In /etc/fail2ban/jail.local:

[recidive]
enabled = true
filter = recidive
logpath = /var/log/fail2ban.log
bantime = 1w
findtime = 1d
maxretry = 3

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence

