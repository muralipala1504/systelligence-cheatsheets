# HugePages and Memory Tuning for LLMs Cheatsheet
# Bare Metal Mastery Series — Part 19
# youtube.com/@SysTelligence

---

## 1. Check Your Defaults (Never Assume)
```bash
# Check swappiness and dirty ratios
sysctl vm.swappiness vm.dirty_ratio

# Check Transparent HugePages (THP) status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Check Explicit HugePages
grep Huge /proc/meminfo

Verified Defaults: RHEL 10.2 has swappiness=30 and dirty_ratio=30. AlmaLinux 10.2/9.8 have swappiness=60 and dirty_ratio=20.

2. The THP Fix (Runtime & Persistent)

# Temporary: Switch THP to madvise
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

# Verify it worked (should show [madvise])
cat /sys/kernel/mm/transparent_hugepage/enabled

# Persistent: Add to kernel boot parameters
sudo grubby --update-kernel=ALL --args="transparent_hugepage=madvise"
sudo reboot

3. Configuring Explicit HugePages

# Allocate 1024 pages (1024 * 2MB = 2GB)
sudo sysctl -w vm.nr_hugepages=1024

# Verify allocation
grep Huge /proc/meminfo
# Output should show: HugePages_Total: 1024, HugePages_Free: 1024

# Make persistent (add to /etc/sysctl.d/99-hugepages.conf)
echo "vm.nr_hugepages = 1024" | sudo tee /etc/sysctl.d/99-hugepages.conf

4. Overcommit and KSM Checks

# Check overcommit memory setting (0 = heuristic default)
sysctl vm.overcommit_memory

# Check Kernel Samepage Merging (KSM)
cat /sys/kernel/mm/ksm/run
# (0 = off. Leave it off unless running many identical VMs).

5. Swappiness & Dirty Ratios (AI Tuning)

# Lower swappiness for AI workloads (keeps weights in RAM)
sudo sysctl -w vm.swappiness=10

# Verify
cat /proc/sys/vm/swappiness

6. systemd LimitMEMLOCK

# Create a drop-in for your AI service
mkdir -p /etc/systemd/system/inference.service.d/
cat <<EOF > /etc/systemd/system/inference.service.d/memlock.conf
[Service]
LimitMEMLOCK=infinity
EOF

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart inference

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
