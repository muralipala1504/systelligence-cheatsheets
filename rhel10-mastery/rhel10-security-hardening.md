# RHEL 10 Security Hardening Cheatsheet
# RHEL 10 Mastery Series — Episode 3
# youtube.com/@SysTelligence

---

## 1. Verify Defaults (Out of the Box)
```bash
# SELinux must be Enforcing
getenforce
sestatus

# System-wide Crypto Policies (RHEL 10 uses DEFAULT)
update-crypto-policies --show
cat /etc/crypto-policies/back-ends/openssh.config

# Firewalld active zones
firewall-cmd --get-active-zones
firewall-cmd --list-all

# Audit daemon
systemctl status auditd

2. SSH Hardening (Drop-in Configs)
RHEL 10 uses /etc/ssh/sshd_config.d/ drop-ins.
Edit /etc/ssh/sshd_config.d/50-redhat.conf or create your own:

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no

# Test config before restarting
sshd -t

# Reload if test passes
systemctl reload sshd

3. Sudo Drop-ins

# NEVER edit /etc/sudoers directly
visudo

# Create drop-in files in /etc/sudoers.d/
vi /etc/sudoers.d/90-custom-admins

# Check what a user is allowed to run
sudo -l -U username

4. Kernel Hardening (sysctl)
Create /etc/sysctl.d/99-hardening.conf:

# Reverse path filtering (IP spoofing protection)
net.ipv4.conf.all.rp_filter = 1

# Address Space Layout Randomization
kernel.randomize_va_space = 2

# SYN flood protection
net.ipv4.tcp_syncookies = 1

# Apply changes
sysctl --system

5. Time Synchronization (Chrony)

# Check tracking and offset
chronyc tracking
chronyc sources

6. USBGuard (Physical Security)

# Install
dnf install usbguard

# Generate baseline policy from currently connected devices
usbguard generate-policy > /etc/usbguard/rules.conf

# Enable and start
systemctl enable --now usbguard

7. Compliance Scanning (OpenSCAP)

# Install the security guide
dnf install scap-security-guide

# Run a CIS profile evaluation
oscap xccdf eval --profile cis \
  --results scan-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml

8. Crypto Policies

# Set to FUTURE for stricter cryptography
update-crypto-policies --set FUTURE

# Requires reboot to fully apply
reboot

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence
