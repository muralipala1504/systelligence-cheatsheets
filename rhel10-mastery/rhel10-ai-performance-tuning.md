# RHEL 10 AI Performance Tuning Cheatsheet
# RHEL 10 Mastery Series — Episode 4
# youtube.com/@SysTelligence

---

## 1. CPU and NUMA Awareness
```bash
# Install numactl
dnf install numactl

# Check hardware topology
numactl --hardware

# Pin process to specific node (e.g., node 0)
numactl --cpunodebind=0 --membind=0 <command>

2. Transparent Huge Pages (THP)
# Check current status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Set to madvise (recommended for AI/Databases)
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled

3. CPU Frequency Scaling (Tuned)
# Install tuned
dnf install tuned

# Enable and start
systemctl enable --now tuned

# Check active profile
tuned-adm active

# Set performance profile
tuned-adm profile throughput-performance

4. Kernel Memory (sysctl)
Create /etc/sysctl.d/99-ai-tuning.conf:

vm.swappiness = 10
vm.overcommit_memory = 1

# Apply changes
sysctl --system

5. I/O Schedulers for NVMe

# Check current scheduler
cat /sys/block/nvme0n1/queue/scheduler

# Set to none (lowest latency for NVMe)
echo none > /sys/block/nvme0n1/queue/scheduler

6. Network Tuning (Ring Buffers)

# Install ethtool
dnf install ethtool

# Check current ring buffer sizes
ethtool -g eth0

# Increase ring buffers
ethtool -G eth0 rx 4096 tx 4096

7. Monitoring

# Visual dashboard
btop

# NVIDIA GPU monitoring
nvidia-smi

# Live refresh
watch -n 1 nvidia-smi

8. IRQ Affinity

# Check interrupts
cat /proc/interrupts

# Pin an IRQ to a specific CPU (e.g., IRQ 45 to CPU 1)
echo 2 > /proc/irq/45/smp_affinity

9. C-States and Power Management

# Check idle states
cpupower idle-info

# Disable deep C-states via kernel boot parameter
# Add to /etc/default/grub: intel_idle.max_cstate=1

10. Real-time Scheduling (chrt)

# Run process with FIFO priority 99
chrt -f 99 <command>

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence
