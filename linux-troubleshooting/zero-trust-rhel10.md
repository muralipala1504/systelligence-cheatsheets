# Zero Trust Architecture on RHEL 10 Cheatsheet
# Linux Troubleshooting Series — Episode 17
# youtube.com/@SysTelligence

---

## 1. Identity (SSH Hardening)
```bash
# Generate a hardware-backed SSH key (FIDO2)
ssh-keygen -t ed25519-sk -O verify-required

# Check current sshd configuration
sudo sshd -T | grep -i -E 'pubkeyauthoptions|authenticationmethods'

# Enforce key-based auth in /etc/ssh/sshd_config or sshd_config.d/
# PasswordAuthentication no
# PubkeyAuthOptions verify-required

# Check authselect profile
authselect current

2. Least Privilege (Sudo)

# Check for blanket wheel access
sudo grep -rn wheel /etc/sudoers /etc/sudoers.d

# Check what a user can run
sudo -l -U USERNAME

# Check sudo logs
sudo journalctl _COMM=sudo --since today

3. Network Segmentation (nftables)

# Test ruleset syntax before applying
sudo nft -c -f segment.nft

# Save current ruleset as backup
sudo nft list ruleset > /root/ruleset.bak

# Schedule automatic rollback after 5 minutes (safety net)
sudo systemd-run --on-active=5m sh -c 'nft flush ruleset; nft -f /root/ruleset.bak'

4. SELinux (Workload Isolation)

# Check context of sensitive files
ls -Z /etc/shadow

# Check running process contexts
ps -eZ | grep httpd

# Search for allow rules (requires setools-console)
sesearch --allow -s httpd_t -t shadow_t

# Check recent AVC denials
sudo ausearch -m AVC -ts recent

5. Container Isolation (Rootless Podman)

# Verify rootless mode is active
podman info --format '{{.Host.Security.Rootless}}'

# Check user namespace mapping
podman unshare cat /proc/self/uid_map

# Check subuid allocations
grep USER /etc/subuid

6. Audit Trail (auditd)

# Example rules in /etc/audit/rules.d/zt.rules:
# -w /etc/sudoers -p wa -k sudoers
# -a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=unset -F euid=0 -k priv-exec

# Load rules
sudo augenrules --load

# Search audit logs
sudo ausearch -k priv-exec -i

7. Secret Management (systemd-creds)

# Encrypt a secret
printf %s "$SECRET" | sudo systemd-creds encrypt --name=dbpass - /etc/myapp/dbpass.cred

# In the systemd unit file:
# LoadCredentialEncrypted=dbpass:/etc/myapp/dbpass.cred
# The service reads from $CREDENTIALS_DIRECTORY/dbpass

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence

