# Slow Linux Server Bottleneck Cheatsheet
# Linux Troubleshooting Series — Episode 9
# youtube.com/@SysTelligence

---

## Install the Modern Stack
```bash
dnf install btop duf ncdu ripgrep sysstat

Step 1: CPU and Memory (btop)
# Interactive visual dashboard
btop

# Inside btop:
# F6 = Sort by CPU
# F2 = Setup / Options

Step 2: Disk Space vs Inodes (duf)
# Show disk space
duf

# Show inode usage
duf --inodes

Step 3: Finding Giant Directories (ncdu)
# Navigate directory sizes interactively
ncdu /

# Inside ncdu:
# d = delete (use with caution)
# g = show graph
# ? = help

Step 4: Disk I/O Wait (iostat)
# Extended stats, exclude idle devices, every 1 second
iostat -xz 1

# Look at the %iowait column. High iowait = disk bottleneck.

Step 5: What is Eating the CPU (pidstat)
# Run every 1 second for 5 iterations
pidstat 1 5

# Show history of CPU usage per process

Step 6: The OOM Killer (journalctl + ripgrep)
# Search kernel logs for killed processes
journalctl -k | rg -i "killed process"

# Check general OOM events
journalctl -k | rg -i "out of memory"

Step 7: Network Bottlenecks (ss)

# Summary of socket stats
ss -s

# List all listening ports with process info
ss -tulpn

Emergency Relief (Runaway Processes)

# Find the runaway service
systemctl list-units --type=service --state=running

# Stop it immediately
systemctl stop servicename

# If it is critical, lower its CPU priority instead
renice +10 -p PID

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence

