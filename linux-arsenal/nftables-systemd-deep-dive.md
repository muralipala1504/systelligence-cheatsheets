# nftables and systemd Deep Dive Cheatsheet
# Linux Arsenal Series — Episode 6
# youtube.com/@SysTelligence

---

## nftables — Complete Reference

### Install and Enable
```bash
# Install nftables
dnf install nftables          # RHEL/CentOS
apt install nftables          # Debian/Ubuntu

# Enable and start
systemctl enable --now nftables

# Check status
systemctl status nftables
```

### Basic Commands
```bash
# List complete ruleset
nft list ruleset

# List ruleset with handles (needed for deletion)
nft list ruleset -a

# List specific table
nft list table inet filter

# List specific chain
nft list chain inet filter input

# Flush entire ruleset
nft flush ruleset

# Load from file
nft -f /etc/nftables.conf

# Check syntax without applying
nft -c -f /etc/nftables.conf
```

### Table Management
```bash
# Create inet table (IPv4 + IPv6 combined)
nft add table inet filter

# Create IPv4 only table
nft add table ip filter

# Create IPv6 only table
nft add table ip6 filter

# Delete table
nft delete table inet filter

# List all tables
nft list tables
```

### Chain Management
```bash
# Create input chain with drop policy
nft add chain inet filter input \
  '{ type filter hook input priority 0; policy drop; }'

# Create forward chain
nft add chain inet filter forward \
  '{ type filter hook forward priority 0; policy drop; }'

# Create output chain with accept policy
nft add chain inet filter output \
  '{ type filter hook output priority 0; policy accept; }'

# Delete chain
nft delete chain inet filter input
```

### Rule Management
```bash
# Allow loopback
nft add rule inet filter input iif lo accept

# Allow established connections
nft add rule inet filter input ct state established,related accept

# Allow SSH
nft add rule inet filter input tcp dport 22 accept

# Allow SSH from specific IP only
nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 22 accept

# Allow HTTP and HTTPS
nft add rule inet filter input tcp dport { 80, 443 } accept

# Block specific IP
nft add rule inet filter input ip saddr 10.0.0.5 drop

# Log and drop
nft add rule inet filter input log prefix "DROPPED: " drop

# Delete rule by handle
nft list ruleset -a                          # Find handle number
nft delete rule inet filter input handle 5   # Delete by handle
```

### Sets — Multiple IPs and Ports
```bash
# Create named IP set
nft add set inet filter allowed_ips \
  '{ type ipv4_addr; }'

# Add elements to set
nft add element inet filter allowed_ips \
  '{ 192.168.1.10, 192.168.1.20, 10.0.0.5 }'

# Use set in rule
nft add rule inet filter input \
  ip saddr @allowed_ips tcp dport 22 accept

# Create port set
nft add set inet filter web_ports \
  '{ type inet_service; }'
nft add element inet filter web_ports '{ 80, 443, 8080 }'

# Dynamic set (runtime updates)
nft add set inet filter blocklist \
  '{ type ipv4_addr; flags dynamic, timeout; timeout 1h; }'
```

### Production Server Ruleset
```bash
# /etc/nftables.conf
flush ruleset

table inet filter {
    set mgmt_ips {
        type ipv4_addr
        elements = { 192.168.1.0/24, 10.0.0.0/8 }
    }

    chain input {
        type filter hook input priority 0; policy drop;

        # Allow loopback
        iif lo accept

        # Allow established connections
        ct state established,related accept

        # Allow SSH from management only
        ip saddr @mgmt_ips tcp dport 22 accept

        # Allow HTTP and HTTPS
        tcp dport { 80, 443 } accept

        # Log dropped packets
        log prefix "INPUT-DROP: "
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

### Migrate from iptables
```bash
# Translate single iptables rule
iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT

# Translate entire saved ruleset
iptables-save | iptables-restore-translate

# Save output to nftables config
iptables-save | iptables-restore-translate > /etc/nftables.conf
```

---

## systemd — Complete Reference

### Service Management
```bash
# Check service status
systemctl status nginx

# Start service
systemctl start nginx

# Stop service
systemctl stop nginx

# Restart service (full stop + start)
systemctl restart nginx

# Reload config (no downtime)
systemctl reload nginx

# Enable at boot
systemctl enable nginx

# Enable and start immediately
systemctl enable --now nginx

# Disable at boot
systemctl disable nginx

# Mask service (prevent from starting)
systemctl mask nginx

# Unmask service
systemctl unmask nginx
```

### List and Query
```bash
# List all running services
systemctl list-units --type=service --state=running

# List all failed services
systemctl list-units --type=service --state=failed

# List all enabled services
systemctl list-unit-files --type=service --state=enabled

# Show service properties
systemctl show nginx

# Show specific property
systemctl show nginx -p MemoryMax
```

### Writing Custom Service Unit
```bash
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target
Requires=postgresql.service

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
MemoryMax=2G
OOMScoreAdjust=-500
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

```bash
# After creating unit file
systemctl daemon-reload
systemctl enable myapp
systemctl start myapp
systemctl status myapp
```

### systemd Timers (Replace cron)
```bash
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh

# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily
After=network.target

[Timer]
OnCalendar=daily
OnCalendar=*-*-* 02:00:00    # 2 AM daily
Persistent=true               # Run if missed

[Install]
WantedBy=timers.target
```

```bash
# Enable and start timer
systemctl enable --now backup.timer

# List all active timers
systemctl list-timers

# Check timer status
systemctl status backup.timer
```

### Troubleshooting Failed Services
```bash
# Step 1 — Check status and recent logs
systemctl status myapp

# Step 2 — Full logs current boot
journalctl -u myapp -b

# Step 3 — Follow logs during restart
journalctl -u myapp -f

# Step 4 — Check dependencies
systemctl list-dependencies myapp

# Step 5 — Analyze boot time
systemd-analyze blame
systemd-analyze critical-chain myapp.service

# Check exit code
systemctl show myapp -p ExecMainStatus
```

---

## Quick Reference Card

| Task | nftables | systemd |
|------|----------|---------|
| List current rules | nft list ruleset | systemctl list-units |
| Add allow rule | nft add rule inet filter input tcp dport 80 accept | — |
| Delete rule | nft delete rule inet filter input handle N | systemctl mask service |
| Load from file | nft -f /etc/nftables.conf | systemctl daemon-reload |
| Persist rules | systemctl enable nftables | systemctl enable service |
| Check logs | — | journalctl -u service -b |
| Schedule task | — | systemctl enable timer |
| Boot analysis | — | systemd-analyze blame |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
