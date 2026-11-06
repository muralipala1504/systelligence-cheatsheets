# RHEL 10 vs AlmaLinux 10 Cheatsheet
# RHEL 10 Mastery Series — Episode 7
# youtube.com/@SysTelligence

---

## 1. Identify the System
```bash
# Check the release version
cat /etc/redhat-release

# Check the OS release metadata
cat /etc/os-release

# Check the kernel version
uname -r

(Verified: Both RHEL 10.2 and Alma 10.2 run the exact same kernel 6.12.0-211.73.1.el10_2.x86_64)

2. Check the Support Lifecycle

# Check the support end date (AlmaLinux includes this)
grep SUPPORT_END /etc/os-release
# Output on Alma 10: SUPPORT_END=2035-06-01
# Output on RHEL 10: (No output. Check the Red Hat lifecycle page).

# Check package base and repos
dnf repolist

3. The Divergence (What Alma Added)
AlmaLinux 10 diverges from RHEL 10 in a few key areas:

Btrfs: Supported on Alma, not supported on RHEL.

Older Drivers: Alma re-adds older storage and network drivers (Adaptec, Dell PERC, Mellanox, QLogic, Emulex, LSI, Broadcom).

i686 Packages: Alma includes i686 user-space packages, RHEL does not.

# Verify Btrfs module availability
modinfo btrfs
# RHEL: modinfo: ERROR: Module btrfs not found.
# Alma: filename: /lib/modules/.../kernel/fs/btrfs/btrfs.ko.xz

# Check for Red Hat specific tools
rpm -q insights-client rhc
# RHEL: Installed.
# Alma: package insights-client is not installed.

4. The Migration (Alma 9 to Alma 10)

# Install the ELevate tool
sudo dnf install -y leapp-upgrade leapp-data-almalinux

# Run the pre-upgrade check
sudo leapp preupgrade

# Review the report (CRITICAL STEP)
cat /var/log/leapp/leapp-report.txt

# Run the upgrade
sudo leapp upgrade
sudo reboot

(Warning: Always take a snapshot first. Never run this on production without a rehearsal.)

5. The Security Patch Race
# Check for available security updates
dnf updateinfo list security
dnf check-update --security

Do not trust marketing claims. Measure the time gap between a Red Hat patch and the AlmaLinux rebuild.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
