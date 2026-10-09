# RHEL 10 Default Changes Cheatsheet
# RHEL 10 Mastery Series — Episode 2
# youtube.com/@SysTelligence

---

## What's Gone in RHEL 10 Minimal

### Network Tools (net-tools removed)
```bash
# Gone — not available even with dnf
ifconfig    # use: ip addr show
netstat     # use: ss -tulpn
route       # use: ip route show
arp         # use: ip neigh show

# Note: net-tools was not default on RHEL 9 minimal either
# ip and ss have been standard since RHEL 8
```

### Firewall
```bash
# Gone — iptables compatibility layer removed
iptables -L          # use: nft list ruleset
ip6tables -L         # use: nft list ruleset
iptables-save        # use: nft list ruleset > /etc/nftables.conf
```

### Python
```bash
# Gone
python               # use: python3
pip                  # use: pip3

# Still available
python3 --version
pip3 --version
```

### Network Scripts
```bash
# Gone — directory does not exist
ls /etc/sysconfig/network-scripts/    # No such directory
# use: nmcli connection show
```

### Package Modularity
```bash
# Gone
dnf module list      # Command not found
dnf module enable    # Command not found
# use: dnf group list
```

---

## Modern Replacements — Default in RHEL 10

### Network Management (nmcli)
```bash
# Show all connections
nmcli connection show

# Show device status
nmcli device status

# Create new ethernet connection
nmcli connection add \
  type ethernet \
  ifname eth0 \
  con-name eth0-static \
  ipv4.method manual \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns 8.8.8.8

# Modify existing connection
nmcli connection modify eth0-static ipv4.addresses 192.168.1.20/24
nmcli connection modify eth0-static ipv4.dns "8.8.8.8 8.8.4.4"

# Activate and deactivate
nmcli connection up eth0-static
nmcli connection down eth0-static

# Delete connection
nmcli connection delete eth0-static

# Show connection details
nmcli connection show eth0-static
```

### Firewall (nftables)
```bash
# List complete ruleset
nft list ruleset

# List with handles
nft list ruleset -a

# Add allow rule
nft add rule inet filter input tcp dport 80 accept
nft add rule inet filter input tcp dport 443 accept
nft add rule inet filter input tcp dport 22 accept

# Block IP
nft add rule inet filter input ip saddr 1.2.3.4 drop

# Delete rule by handle
nft delete rule inet filter input handle 5

# Load from file
nft -f /etc/nftables.conf

# Enable persistence
systemctl enable --now nftables

# firewalld still works on top of nftables
firewall-cmd --list-all
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

### Socket and Network Diagnosis
```bash
# Replace netstat with ss
ss -tulpn                    # All listening ports
ss -t state established      # Active connections
ss -s                        # Socket summary

# Replace ifconfig with ip
ip addr show                 # All interfaces
ip link show                 # Interface state
ip route show                # Routing table
ip route get 8.8.8.8        # Routing decision
ip neigh show                # ARP table
```

---

## Crypto Policy Changes

### Check and Set Policy
```bash
# Check current policy
update-crypto-policies --show

# List available policies
update-crypto-policies --list

# Set default policy
update-crypto-policies --set DEFAULT

# Temporarily allow SHA1 (migration only)
update-crypto-policies --set DEFAULT:SHA1

# Restore after migration
update-crypto-policies --set DEFAULT
```

### What DEFAULT Policy Blocks
BLOCKED in RHEL 10 DEFAULT:

TLS 1.0 and TLS 1.1
SHA1 signed certificates
MD5 hashes
RC4 cipher
RSA keys below 2048 bits

ALLOWED in RHEL 10 DEFAULT:

TLS 1.2 and TLS 1.3
SHA256 and above
RSA 2048 and above
AES ciphers
Ed25519 and ECDSA

### Check Certificates
```bash
# Check single certificate
openssl x509 -in cert.pem -noout -text | grep "Signature Algorithm"

# Check all certificates in directory
fd -e pem -e crt /etc/pki | \
  xargs -I{} openssl x509 -in {} -noout -text 2>/dev/null | \
  grep "Signature Algorithm"

# Check live TLS connection
openssl s_client -connect hostname:443 2>/dev/null | \
  openssl x509 -noout -text | grep "Signature Algorithm"
```

---

## Python Migration

### Audit Scripts
```bash
# Find all Python scripts
fd -e py /opt /home /etc

# Find scripts with wrong shebang
fd -e py | xargs rg "^#!/usr/bin/python$"
fd -e py | xargs rg "^#!/usr/bin/env python$"

# Fix shebang (replace python with python3)
fd -e py | xargs sed -i 's|#!/usr/bin/python$|#!/usr/bin/python3|g'
fd -e py | xargs sed -i 's|#!/usr/bin/env python$|#!/usr/bin/env python3|g'

# Verify fix
fd -e py | xargs rg "^#!/usr/bin/python3"
```

### Python Package Management
```bash
# Install Python packages
dnf install python3-requests
dnf install python3-pip

# pip3 usage
pip3 install package
pip3 list
pip3 freeze > requirements.txt

# Virtual environment
python3 -m venv myenv
source myenv/bin/activate
pip3 install -r requirements.txt
```

---

## Containers — Podman

### Install Podman (not default in minimal)
```bash
dnf install podman
podman --version
```

### Basic Podman Commands
```bash
# Docker to Podman — commands are identical
podman run -d nginx                    # docker run -d nginx
podman ps                              # docker ps
podman images                          # docker images
podman pull nginx                      # docker pull nginx
podman stop container_id               # docker stop container_id
podman rm container_id                 # docker rm container_id
podman logs container_id               # docker logs container_id

# Create Docker alias (for existing scripts)
echo "alias docker=podman" >> ~/.bashrc

# Rootless container
podman run --user 1000 -d nginx

# Check podman info
podman info
podman system info
```

---

## systemd-oomd

### Check Status
```bash
# Check oomd status
systemctl status systemd-oomd
oomctl status
oomctl dump

# View memory pressure
oomctl dump | grep -i pressure
```

### Protect Critical Services
```bash
# Add to service unit file
[Service]
OOMScoreAdjust=-900

# Apply to running service
systemctl set-property sshd.service OOMScoreAdjust=-900

# Verify
systemctl show sshd -p OOMScoreAdjust
```

---

## cgroup v2

### Verify cgroup Version
```bash
# Check cgroup version
stat -fc %T /sys/fs/cgroup/
# Returns "cgroup2fs" for v2

# View cgroup hierarchy
systemd-cgls

# Check memory limits per service
cat /sys/fs/cgroup/system.slice/nginx.service/memory.max

# Set memory limit via systemd
systemctl set-property nginx.service MemoryMax=1G
```

---

## EPEL — Modern Tool Stack

### Enable EPEL
```bash
# AlmaLinux 10
dnf install epel-release

# RHEL 10
subscription-manager repos \
  --enable codeready-builder-for-rhel-10-x86_64-rpms
dnf install \
  https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm

# Verify
dnf repolist | grep epel
```

### Install Modern Tools
```bash
# All from EPEL
dnf install fd-find ripgrep btop bat duf ncdu

# Available without EPEL (default repos)
# ss, ip, jq, journalctl, nmcli, nft, python3
```

---

## Eight Post-Install Verification Commands
```bash
# 1. OS version
cat /etc/redhat-release

# 2. Kernel version
uname -r

# 3. Crypto policy
update-crypto-policies --show

# 4. Failed services
systemctl --failed

# 5. Firewall state
nft list ruleset

# 6. Network configuration
nmcli connection show

# 7. Podman available (if needed)
podman --version 2>/dev/null || echo "Podman not installed — dnf install podman"

# 8. Boot errors
journalctl -p err -b
```

---

## Quick Reference — Command Mapping

| Old Command | RHEL 10 Command | Package Needed |
|-------------|-----------------|----------------|
| ifconfig | ip addr show | default |
| netstat -tulpn | ss -tulpn | default |
| route | ip route show | default |
| arp | ip neigh show | default |
| iptables -L | nft list ruleset | default |
| python | python3 | default |
| pip | pip3 | default |
| docker | podman | dnf install podman |
| service start | systemctl start | default |
| yum | dnf | default (alias) |

---

## Corrections and Addendums
ifconfig and netstat were not default on RHEL 9 minimal either.
net-tools package is not available in RHEL 10 base repos.
Standard tools are ip and ss since RHEL 8.
Podman is not installed by default in RHEL 10 minimal.
Install with: dnf install podman
Docker is not supported on RHEL 10.

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
