# RHEL 10 Upgrade Roadmap Cheatsheet
# RHEL 10 Mastery Series — Episode 1
# youtube.com/@SysTelligence

---

## RHEL Version Timeline
RHEL 7 — Released 2014 — EOS June 2024 (EXPIRED)
RHEL 8 — Released 2019 — EOS May 2029
RHEL 9 — Released 2022 — EOS May 2032
RHEL 10 — Released 2025 — EOS ~2035
AlmaLinux 10 — Binary compatible with RHEL 10 — Free

---

## Check Current RHEL Version
```bash
# Check OS version
cat /etc/redhat-release
cat /etc/os-release

# Check kernel version
uname -r

# Check subscription status (RHEL only)
subscription-manager status
subscription-manager release --show

# Check installed packages count
dnf list installed | wc -l
```

---

## RHEL 10 Key Changes Summary

### Kernel
Linux kernel 6.x base
cgroup v2 fully default — no v1 fallback
Improved NVMe multipath support
Better NUMA topology awareness for AI workloads

### Security
DEFAULT crypto policy: TLS 1.3 minimum
SHA1 disabled in DEFAULT crypto policy
SSH default host key: Ed25519
Firewalld: nftables backend exclusively
OpenSSL 3.0 minimum
iptables compatibility layer removed

### Package Management

### Network
NetworkManager only — network-scripts removed
ifcfg files not supported
nftables only — no iptables compatibility
ss replaces netstat (net-tools not available)
ip replaces ifconfig

### Storage
LVM + XFS remains default
Stratis available as modern alternative
XFS improvements for NVMe performance
No change to existing LVM workflows

### Containers
Podman 5 default container runtime
Docker not supported on RHEL 10
Rootless containers improved
Full GPU passthrough support in Podman

### systemd
cgroup v2 fully enforced
systemd-oomd enabled by default
journald persistent logging default
systemd-resolved default DNS resolver

---

## Six Breaking Changes — RHEL 7 to RHEL 10

### 1. Python 2 Removed
```bash
# Check Python version
python3 --version

# Find Python 2 scripts
fd -e py . | xargs rg "#!/usr/bin/python$" 2>/dev/null
fd -e py . | xargs rg "print " 2>/dev/null   # Python 2 print statements

# Python 2 to 3 migration tool
dnf install python3-modernize
python-modernize -w script.py
```

### 2. iptables Removed
```bash
# Check existing iptables rules
iptables -L -n -v                    # Will fail on RHEL 10

# Convert to nftables
iptables-save | iptables-restore-translate > /etc/nftables.conf
nft -f /etc/nftables.conf
systemctl enable --now nftables

# Verify nftables rules
nft list ruleset
```

### 3. network-scripts Removed
```bash
# Check for ifcfg files (RHEL 7/8/9)
ls /etc/sysconfig/network-scripts/

# RHEL 10 — use nmcli instead
nmcli connection show
nmcli connection add type ethernet ifname eth0 con-name eth0
nmcli connection modify eth0 ipv4.addresses 192.168.1.10/24
nmcli connection modify eth0 ipv4.gateway 192.168.1.1
nmcli connection up eth0

# Show connection details
nmcli connection show eth0
```

### 4. SysV Init Scripts Not Supported
```bash
# Check for SysV scripts
ls /etc/init.d/

# Convert to systemd unit
# Create /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
ExecStart=/opt/myapp/bin/start.sh
Restart=always

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable --now myapp
```

### 5. SHA1 Certificates Rejected
```bash
# Check certificate hash algorithm
openssl x509 -in cert.pem -noout -text | grep "Signature Algorithm"

# Check all certs in directory
fd -e pem -e crt /etc/pki | xargs -I{} openssl x509 -in {} -noout -text 2>/dev/null | grep "Signature Algorithm"

# Temporarily allow SHA1 (migration only)
update-crypto-policies --set DEFAULT:SHA1

# Generate new SHA256 certificate
openssl req -newkey rsa:4096 -sha256 -nodes \
  -keyout new.key -out new.csr
```

### 6. yum Plugins Not Compatible
```bash
# yum is now just a dnf alias
which yum
yum --version    # Shows dnf version

# Check for yum plugins
ls /etc/yum/pluginconf.d/
ls /usr/lib/yum-plugins/

# dnf equivalents
dnf install package          # replaces yum install
dnf update                   # replaces yum update
dnf search package           # replaces yum search
dnf repolist                 # replaces yum repolist
dnf history                  # replaces yum history
```

---

## Enable EPEL for Modern Tool Stack
```bash
# RHEL 10
subscription-manager repos --enable codeready-builder-for-rhel-10-x86_64-rpms
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm

# AlmaLinux 10
dnf install epel-release

# Verify EPEL enabled
dnf repolist | grep epel

# Install modern tool stack
dnf install fd-find ripgrep btop bat duf ncdu

# Tools available without EPEL (default repos)
# ss, ip, journalctl, jq, nmcli, nft — all default
```

---

## Upgrade Paths

### RHEL 7 to RHEL 10
```bash
# In-place upgrade NOT supported
# Fresh installation required

# Pre-migration checklist
python3 --version           # Ensure Python 3 apps ready
systemctl list-units --type=service  # Check SysV scripts
ls /etc/sysconfig/network-scripts/   # Check ifcfg files
iptables -L                          # Document firewall rules
openssl x509 -in cert.pem -text | grep "Signature Algorithm"  # Check certs
```

### RHEL 8/9 to RHEL 10 (leapp)
```bash
# Install leapp
dnf install leapp-upgrade

# Pre-upgrade check
leapp preupgrade

# Review report
bat /var/log/leapp/leapp-report.txt

# Run upgrade (requires reboot)
leapp upgrade

# Post-upgrade verification
cat /etc/redhat-release
uname -r
systemctl --failed
```

### AlmaLinux Migration
```bash
# AlmaLinux 10 — same upgrade tooling
dnf install leapp-upgrade
leapp preupgrade
leapp upgrade
```

---

## Security Defaults — Verify and Fix

### Crypto Policy
```bash
# Check current policy
update-crypto-policies --show

# Available policies
update-crypto-policies --list

# RHEL 10 default
update-crypto-policies --set DEFAULT

# If SHA1 needed temporarily
update-crypto-policies --set DEFAULT:SHA1
```

### SSH Defaults
```bash
# Check SSH host key types
ls /etc/ssh/ssh_host_*

# RHEL 10 generates Ed25519 by default
# Check sshd config
bat /etc/ssh/sshd_config | grep -E "HostKey|PubkeyAuth"

# Verify SSH service
systemctl status sshd
```

---

## Quick Reference — RHEL 10 Command Changes

| Old Command | RHEL 10 Command | Notes |
|-------------|-----------------|-------|
| ifconfig | ip addr show | net-tools removed |
| netstat | ss -tulpn | net-tools removed |
| iptables | nft list ruleset | iptables removed |
| service start | systemctl start | SysV removed |
| yum install | dnf install | yum is alias only |
| python | python3 | Python 2 removed |
| ifcfg files | nmcli connection | network-scripts removed |

---

## Upgrade Decision Guide
RHEL 7 → EOS EXPIRED → Migrate NOW → Fresh install to RHEL 10
RHEL 8 → EOS 2029 → Plan migration → leapp upgrade supported
RHEL 9 → EOS 2032 → Monitor → leapp upgrade supported
No subscription → Use AlmaLinux 10 → Binary compatible with RHEL 10

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
