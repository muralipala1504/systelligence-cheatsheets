# Linux OOM Killer Cheatsheet
# Linux Troubleshooting Series — Episode 6
# youtube.com/@SysTelligence

---

## What is OOM
OOM = Out of Memory
Exit code 137 = process was killed by the OOM Killer (not a crash)

---

## Detect OOM Killer Events
```bash
# Check dmesg for OOM events
dmesg | grep -i oom

# Check kernel journal
journalctl -k | grep -i "killed process"

# Check syslog (RHEL)
grep "Out of memory" /var/log/messages

# Check full OOM sequence
dmesg | grep -A 10 "Out of memory"

# Search by time
dmesg -T | grep -i oom

# ausearch for OOM
ausearch -m anom_abend | grep -i oom
```

---

## Read OOM Kill Message
```bash
# Full OOM message from dmesg
dmesg | grep -B 5 -A 20 "Out of memory"

# Key fields to read:
# - Timestamp: when it happened
# - Triggering process: what requested memory
# - Memory map: all processes at time of kill
# - "Out of memory" line: selected victim
# - "Killed process" line: PID and name killed, memory freed
```

---

## Memory Analysis Commands
```bash
# Overall memory status
free -h
# Focus on "available" column — not "free"

# Detailed memory statistics
vmstat -s

# Top memory consumers
ps aux --sort=-%mem | head -15

# Detailed kernel memory info
cat /proc/meminfo
# Key fields: MemAvailable, SwapFree, SwapTotal

# Memory per process (accurate shared memory)
smem -rs rss | head -10

# Watch memory in real time
watch -n 2 free -h

# Check swap usage
swapon --show
```

---

## OOM Score Management
```bash
# Check process OOM score (higher = killed first)
cat /proc/PID/oom_score

# Check OOM score adjustment
cat /proc/PID/oom_score_adj

# Protect critical process (temporary)
echo -900 > /proc/PID/oom_score_adj

# Make process first target (temporary)
echo 500 > /proc/PID/oom_score_adj

# Protect via systemd (permanent)
# Add to service unit file:
# [Service]
# OOMScoreAdjust=-900

# Apply to running service
systemctl set-property servicename.service OOMScoreAdjust=-900
```

### Processes to Always Protect
```bash
# SSH daemon — never lose remote access
echo -900 > /proc/$(pgrep sshd | head -1)/oom_score_adj

# Monitoring agent
echo -900 > /proc/$(pgrep node_exporter)/oom_score_adj

# Alertmanager
echo -900 > /proc/$(pgrep alertmanager)/oom_score_adj
```

---

## Find Memory Leaks
```bash
# Step 1 — Watch available memory trend
watch -n 5 'free -h | grep Mem'

# Step 2 — Track process RSS over time
watch -n 10 'ps aux --sort=-%mem | head -5'

# Step 3 — Detailed process memory map
pmap -x PID | sort -k3 -rn | head -20

# Step 4 — Track specific process RSS
while true; do
  ps -p PID -o pid,rss,vsz,comm
  sleep 30
done

# Step 5 — Compare RSS snapshots
ps aux --sort=-%mem | awk '{print $2,$6,$11}' | head -10
```

---

## Kernel OOM Parameters
```bash
# Check current values
sysctl vm.swappiness
sysctl vm.overcommit_memory
sysctl vm.overcommit_ratio

# Apply immediately
sysctl -w vm.swappiness=10
sysctl -w vm.overcommit_memory=2
sysctl -w vm.overcommit_ratio=80
sysctl -w vm.panic_on_oom=0
sysctl -w vm.oom_kill_allocating_task=1

# Make permanent — add to /etc/sysctl.conf
vm.swappiness = 10
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
vm.panic_on_oom = 0
vm.oom_kill_allocating_task = 1

# Apply from file
sysctl -p /etc/sysctl.conf
```

### Parameter Reference
vm.swappiness=10 # Less aggressive swap (default 60)
vm.overcommit_memory=2 # Strict allocation (default 0)
vm.overcommit_ratio=80 # 80% of RAM committable (default 50)
vm.panic_on_oom=1 # Reboot instead of kill (use carefully)
vm.oom_kill_allocating_task=1 # Kill requester not highest scorer

---

## cgroups Memory Limits

### systemd Service Limits
```bash
# Set memory limit in service unit
# /etc/systemd/system/myapp.service
[Service]
MemoryMax=2G
MemoryHigh=1.5G        # Soft limit — throttle before hard limit
MemorySwapMax=0        # No swap for this service

# Apply changes
systemctl daemon-reload
systemctl restart myapp

# Verify limits
systemctl show myapp | grep Memory
cat /sys/fs/cgroup/system.slice/myapp.service/memory.max
```

### Docker Memory Limits
```bash
# Run container with memory limit
docker run -m 2g --memory-swap 2g myapp

# Update running container
docker update --memory 2g container_name

# Check container memory usage
docker stats container_name
```

### Kubernetes Memory Limits
```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "2Gi"
```

---

## Ten Step OOM Recovery Sequence
```bash
# Step 1 — SSH in immediately
ssh user@server

# Step 2 — Check current memory
free -h

# Step 3 — Identify what was killed
dmesg | grep -i "killed process"

# Step 4 — Check critical services
systemctl status nginx sshd prometheus

# Step 5 — Restart killed services
systemctl restart killed-service

# Step 6 — Find memory hog
ps aux --sort=-%mem | head -10

# Step 7 — Watch for continued leak
watch -n 5 'ps aux --sort=-%mem | head -5'

# Step 8 — Protect critical processes
echo -900 > /proc/$(pgrep sshd)/oom_score_adj

# Step 9 — Set cgroup limits on offender
systemctl set-property offender.service MemoryMax=1G

# Step 10 — Monitor stability
watch -n 2 free -h
```

---

## Prevention Checklist
```bash
# 1. Monitoring alert at 20% available memory
# Prometheus rule:
# node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.20

# 2. Monitoring alert at 50% swap usage
# node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes < 0.50

# 3. Set MemoryMax on all application services
systemctl set-property app.service MemoryMax=2G

# 4. Protect critical processes in unit files
# OOMScoreAdjust=-900 in [Service] section

# 5. Tune kernel parameters
sysctl -w vm.swappiness=10
echo "vm.swappiness=10" >> /etc/sysctl.conf
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Check for OOM events | dmesg \| grep -i oom |
| Check memory | free -h |
| Top memory consumers | ps aux --sort=-%mem \| head -10 |
| Check process OOM score | cat /proc/PID/oom_score |
| Protect process | echo -900 > /proc/PID/oom_score_adj |
| Set memory limit | systemctl set-property app MemoryMax=2G |
| Reduce swap aggression | sysctl -w vm.swappiness=10 |
| Watch memory live | watch -n 2 free -h |
| Full OOM message | dmesg \| grep -B5 -A20 "Out of memory" |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
