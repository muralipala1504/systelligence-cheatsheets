# Linux Inode Detection — Is It Inodes or Space
# Linux Troubleshooting Series — Episode 8
# youtube.com/@SysTelligence

---

## What Inode Exhaustion Actually Is
Disk space and inode space are two separate resources.
A filesystem can have free blocks and zero free inodes.
Result — every write fails even though df says there is room.
The kernel reports "No space left on device" for both cases.

---

## Install the Modern Stack
```bash
# One command, from EPEL
dnf install duf ncdu bat ripgrep fd-find btop

# Verify
duf --version
ncdu --version
rg --version
fd --version
btop --version

Detection — Is It Inodes or Space

# Classic — human readable inode usage
df -i

# Human readable with sizes
df -ih

# Modern — duf shows inode column alongside space
duf

# duf inodes only view
duf --inodes

# Filesystem level inode totals
tune2fs -l /dev/sda1 | grep -i inode

# Single file inode number
stat /etc/hosts

Finding Where Inodes Went

# ncdu in inode mode — navigate directory by directory
ncdu --inodes /

# ncdu in inode mode on a specific mount
ncdu --inodes /var

# Count files in a directory tree with fd
fd --type f | wc -l

# Count files under a specific path
fd --type f . /var/spool | wc -l

# Count directories under a path
fd --type d . /var/spool | wc -l

# Top 20 dirs by file count (find + sort)
fd --type f . /var | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -20

# Search logs for inode errors with ripgrep
rg "No space left on device" /var/log

# Kernel messages for inode warnings
journalctl -k | rg -i "inode|no space"

The Most Common Inode Killers
Mail spool

# Inspect with ncdu first
ncdu --inodes /var/spool/mail
ncdu --inodes /var/mail

# View config files with bat before touching
bat /etc/aliases

# Count mail files
fd --type f . /var/spool/mail | wc -l

# Remove files older than 30 days (after inspection)
find /var/spool/mail -type f -mtime +30 -delete

Package manager cache

# Check size
duf --inodes /

# View cache directory
ncdu /var/cache/dnf

# Safe cleanup
dnf clean all

Application caches

# Identify cache directory with ncdu
ncdu --inodes /var/cache

# Verify nothing active depends on the files
lsof +D /var/cache/appname

# Only then remove
rm -rf /var/cache/appname/*

Session files and temp dirs

# Count session files
fd --type f . /var/lib/systemd | wc -l
fd --type f . /tmp | wc -l

# Check systemd-tmpfiles policy
bat /usr/lib/tmpfiles.d/tmp.conf

Safe Cleanup — Four Scenarios
1. Mail spool

ncdu --inodes /var/spool/mail     # inspect first
bat /etc/aliases                  # view config
find /var/spool/mail -type f -mtime +30 -delete

2. Package manager cache

dnf clean all                     # always safe, no service impact

3. Application caches
ncdu --inodes /var/cache          # identify directory
lsof +D /var/cache/appname        # verify nothing active
rm -rf /var/cache/appname/*

4. Old rotated logs

journalctl --vacuum-time=7d       # systemd journal
logrotate -f /etc/logrotate.conf  # logrotate config
# NEVER delete active log files
rg "active" /var/log/messages     # verify before removing anything

Prevention
Alert at 80 percent inode usage

# Manual check pattern
df -i | awk 'NR>1 {gsub("%","",$5); if ($5 > 80) print $0}'

# Add to systemd timer or cron
# Alert via mail, slack, or pager

Review filesystem type

# ext4 inode count is fixed at mkfs time
tune2fs -l /dev/sda1 | grep -i "inode count"

# xfs allocates inodes dynamically
xfs_info /mountpoint

# Consider xfs for high-file-count workloads

Automated cleanup with systemd timers
# List existing timers
systemctl list-timers

# Inspect a tmpfiles policy
bat /usr/lib/tmpfiles.d/tmp.conf

# Edit tmpfiles policy
vi /etc/tmpfiles.d/cleanup.conf

# Apply
systemd-tmpfiles --clean

Log retention policy

# Journal size cap
journalctl --vacuum-size=500M

# Permanent cap in journald config
bat /etc/systemd/journald.conf
# Set SystemMaxUse=500M
systemctl restart systemd-journald

# logrotate retention
bat /etc/logrotate.conf
# Set rotate count and maxage

Filesystem Specifics
ext4
# Inode count fixed at mkfs
tune2fs -l /dev/sda1 | grep -i inode

# Check free inode count
dumpe2fs -h /dev/sda1 | grep -i "free inodes"

# Cannot add inodes without reformat

xfs

# Dynamic inode allocation
xfs_info /mountpoint

# Inode usage stats
xfs_db -r -c "freesp -s" /dev/sda1

# No fixed inode limit — space is the only constraint

# Dynamic inode allocation
xfs_info /mountpoint

# Inode usage stats
xfs_db -r -c "freesp -s" /dev/sda1

# No fixed inode limit — space is the only constraint

Emergency Recovery Sequence
# Step 1 — Confirm it is inodes, not space
df -i
df -h

# Step 2 — Find the guilty directory fast
ncdu --inodes /

# Step 3 — If ncdu cannot start, use fd + wc
fd --type f . /var | wc -l

# Step 4 — Identify the biggest count
du --inodes -s /var/* 2>/dev/null | sort -rn | head

# Step 5 — Free inodes without breaking services
dnf clean all
journalctl --vacuum-time=1d

# Step 6 — Verify writes work again
touch /tmp/write-test && rm /tmp/write-test

# Step 7 — Monitor recovery
watch -n 5 'df -i'

Monitoring Inode Usage

# Live dashboard — btop shows inodes alongside CPU, mem, disk
btop

# Periodic inode check
watch -n 10 'df -i'

# Log inode usage to a file
df -i >> /var/log/inode-monitor.log

# Kernel inode warnings
journalctl -k | rg -i "inode"

# Combined view — duf inodes column
duf --inodes
Quick Reference Card
Problem	Command
Is it inodes or space	df -i ; df -h
Human readable inodes	df -ih
Modern inode view	duf --inodes
Find guilty directory	ncdu --inodes /
Count files fast	fd --type f . /path	wc -l
Search inode errors	rg "No space left" /var/log
Filesystem inode totals	tune2fs -l /dev/sda1
Single file inode	stat /etc/hosts
Journal size cap	journalctl --vacuum-size=500M
Journal time cap	journalctl --vacuum-time=7d
Safe cache cleanup	dnf clean all
Verify active file deps	lsof +D /path
Modern Tool Stack for Inode Diagnosis

duf         # replaces df for human readable output
ncdu        # disk usage with inode mode — the killer feature
fd          # replaces find for counting files fast
rg          # replaces grep for log searches
bat         # replaces cat for viewing configs
btop        # replaces top with inode awareness
df -i       # classic, still the standard for inode only view
stat        # single file inode info
tune2fs     # ext4 filesystem inode details
journalctl  # systemd journal, integrated with rg

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence



