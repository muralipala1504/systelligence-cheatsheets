# SSH Hardening Production Guide
# Linux Security Hardening Series — Episode 2
# youtube.com/@SysTelligence

---

## SSH Key Generation and Deployment
```bash
# Generate Ed25519 key (recommended over RSA)
ssh-keygen -t ed25519 -C "server-name-2026"

# Generate with custom filename
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_prod -C "prod-server"

# Copy public key to server
ssh-copy-id user@server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Manual authorized_keys setup
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh

# Test key login
ssh -i ~/.ssh/id_ed25519 user@server

# Convert RSA to Ed25519
ssh-keygen -t ed25519 -C "new-key"
# Deploy new key, test, then remove old RSA key
```

---

## Hardened sshd_config
```bash
# /etc/ssh/sshd_config — production hardened settings

# Disable root login
PermitRootLogin no

# Disable password authentication (after key access confirmed)
PasswordAuthentication no
ChallengeResponseAuthentication no

# Change default port
Port 2222

# Reduce authentication attempts
MaxAuthTries 3
MaxSessions 5

# Reduce authentication window
LoginGraceTime 30

# Disable unnecessary features
X11Forwarding no
AllowTcpForwarding no
GatewayPorts no
PermitTunnel no
AllowAgentForwarding no

# Restrict SSH version
Protocol 2

# Set idle timeout (seconds)
ClientAliveInterval 300
ClientAliveCountMax 2

# Use strong ciphers only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512

# Test config before restart
sshd -t

# Restart after confirmed clean
systemctl restart sshd
```

---

## SSH Access Control
```bash
# Restrict to specific users
AllowUsers deploy murali admin

# Restrict to specific group
AllowGroups sshusers

# Create SSH group and add users
groupadd sshusers
usermod -aG sshusers murali
usermod -aG sshusers deploy

# Restrict by IP with Match block
Match Address 192.168.1.0/24,10.0.0.0/8
    AllowUsers murali deploy
    PasswordAuthentication no

# Deny specific users
DenyUsers testuser tempuser

# Restrict root to specific IPs only
PermitRootLogin no
Match Address 192.168.1.100
    PermitRootLogin forced-commands-only
```

---

## Change SSH Port and Firewall
```bash
# Update sshd_config
Port 2222

# Add firewall rule for new port (nftables)
nft add rule inet filter input tcp dport 2222 accept

# Remove old port 22 rule
nft list ruleset -a
nft delete rule inet filter input handle NUMBER

# SELinux — add new port label (RHEL)
semanage port -a -t ssh_port_t -p tcp 2222

# Test new port before closing session
ssh -p 2222 user@server

# Apply sshd config
systemctl restart sshd
```

---

## Fail2ban Setup
```bash
# Install fail2ban
dnf install fail2ban        # RHEL
apt install fail2ban        # Debian/Ubuntu

# Create local jail config
# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
backend = systemd

[sshd]
enabled = true
port = 2222
logpath = %(sshd_log)s

# Start and enable
systemctl enable --now fail2ban

# Check banned IPs
fail2ban-client status sshd

# Unban specific IP
fail2ban-client set sshd unbanip 1.2.3.4

# Watch fail2ban log
journalctl -u fail2ban -f
```

---

## SSH Certificates
```bash
# Create Certificate Authority
ssh-keygen -t ed25519 -f /etc/ssh/ca_key -C "SSH-CA"

# Sign user public key (30 day validity)
ssh-keygen -s /etc/ssh/ca_key \
  -I "murali-2026" \
  -n murali \
  -V +30d \
  ~/.ssh/id_ed25519.pub

# Configure server to trust CA
# Add to /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/ca_key.pub

# Revoke certificate
ssh-keygen -k -f /etc/ssh/revoked_keys -z 1 signed_key.pub
RevokedKeys /etc/ssh/revoked_keys

# View certificate details
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub
```

---

## Two Factor Authentication
```bash
# Install Google Authenticator PAM
dnf install google-authenticator    # RHEL
apt install libpam-google-authenticator  # Debian

# Setup per user
google-authenticator

# Configure PAM
# /etc/pam.d/sshd — add line:
auth required pam_google_authenticator.so

# Configure sshd_config
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive

# Restart sshd
systemctl restart sshd
```

---

## SSH Monitoring
```bash
# Today's SSH events
journalctl -u sshd --since today

# Failed login attempts
rg "Failed password" /var/log/auth.log
rg "Failed password" /var/log/secure        # RHEL

# Failed attempts by IP (top offenders)
rg "Failed password" /var/log/auth.log | \
  awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -10

# Successful logins
rg "Accepted publickey" /var/log/auth.log

# Last login per user
lastlog

# Currently logged in users
who
w

# Login history
last | head -20

# Failed login history
lastb | head -20

# Watch SSH in real time
journalctl -u sshd -f

# Check for unusual login times
rg "Accepted" /var/log/auth.log | awk '{print $1,$2,$3,$9,$11}'
```

---

## Twelve Item Hardening Checklist
```bash
# 1. Disable root login
PermitRootLogin no

# 2. Disable password authentication
PasswordAuthentication no

# 3. Use Ed25519 keys
ssh-keygen -t ed25519

# 4. Change default port
Port 2222

# 5. Reduce MaxAuthTries
MaxAuthTries 3

# 6. Reduce LoginGraceTime
LoginGraceTime 30

# 7. Disable X11 forwarding
X11Forwarding no

# 8. Disable TCP forwarding
AllowTcpForwarding no

# 9. Restrict access
AllowGroups sshusers

# 10. Deploy fail2ban
systemctl enable --now fail2ban

# 11. Enable monitoring
journalctl -u sshd --since today

# 12. Test before restart
sshd -t && systemctl restart sshd
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Generate key | ssh-keygen -t ed25519 |
| Deploy key | ssh-copy-id user@server |
| Test config | sshd -t |
| Restart SSH | systemctl restart sshd |
| Check failed logins | rg "Failed" /var/log/auth.log |
| Check active sessions | who |
| Last logins | lastlog |
| Ban status | fail2ban-client status sshd |
| Unban IP | fail2ban-client set sshd unbanip IP |
| Live SSH log | journalctl -u sshd -f |

---
*Never leave SSH on default config in production.*
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
