# Linux Kernel Security - sysctl Hardening Cheatsheet
# Linux Security Hardening Series — Episode 5
# youtube.com/@SysTelligence

---

## 1. Verify Defaults
```bash
# Reverse Path Filtering (RHEL 10 default is 0)
sysctl net.ipv4.conf.all.rp_filter

# SYN Cookies (Default is 1)
sysctl net.ipv4.tcp_syncookies

# Address Space Layout Randomization (Default is 2)
sysctl kernel.randomize_va_space

# Kernel Pointer Restriction (Default is 1)
sysctl kernel.kptr_restrict

# dmesg Restriction (Default is 0)
sysctl kernel.dmesg_restrict

2. The Sysctl Drop-in (Hardening)
Create /etc/sysctl.d/99-hardening.conf:

# Strict Reverse Path Filtering
net.ipv4.conf.all.rp_filter = 1

# Restrict dmesg to root only
kernel.dmesg_restrict = 1

# Hide kernel pointers
kernel.kptr_restrict = 1

# Prevent privileged processes from writing core dumps
fs.suid_dumpable = 0

# Restrict Magic SysRq to sync only (16)
kernel.sysrq = 16

# Apply the drop-in
sysctl --system

3. Disabling Core Dumps (Shell Level)

# Disable core dumps for the current shell session
ulimit -c 0

4. Memory Security (ASLR & NX)

# Ensure full ASLR is enabled (Set to 2)
sysctl -w kernel.randomize_va_space=2

# NX bit is enforced by hardware and the kernel by default.
# Do not disable it.

5. Verification

# Apply and verify a specific file
sysctl -p /etc/sysctl.d/99-hardening.conf

# Query the running kernel state to confirm
sysctl net.ipv4.conf.all.rp_filter
sysctl kernel.dmesg_restrict

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence
