# RAID Implementation on Bare Metal Cheatsheet
# Bare Metal Mastery Series — Part 15B
# youtube.com/@SysTelligence

---

## Pre-RAID Drive Identification
```bash
# List all block devices
lsblk

# Detailed view with type
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# NVMe drive details
fdisk -l | grep nvme

# Check existing RAID arrays
cat /proc/mdstat

# Check existing mdadm arrays
mdadm --examine --scan
```

---

## GPT Partitioning (Required for drives above 2TB)
```bash
# Create GPT partition table
parted /dev/nvme0n1 mklabel gpt

# Create partition spanning full drive
parted /dev/nvme0n1 mkpart primary 0% 100%

# Set RAID flag
parted /dev/nvme0n1 set 1 raid on

# Verify partition
parted /dev/nvme0n1 print

# Repeat for each drive
parted /dev/nvme1n1 mklabel gpt
parted /dev/nvme1n1 mkpart primary 0% 100%
parted /dev/nvme1n1 set 1 raid on
```

---

## mdadm RAID Array Creation

### RAID 0 — Striping (AI Training Scratch Space)
```bash
# Create RAID 0 with two drives
mdadm --create /dev/md0 \
  --level=0 \
  --raid-devices=2 \
  /dev/nvme0n1p1 /dev/nvme1n1p1

# Verify
cat /proc/mdstat
mdadm --detail /dev/md0
```

### RAID 1 — Mirror (OS and Boot Drive)
```bash
# Create RAID 1 with two drives
mdadm --create /dev/md1 \
  --level=1 \
  --raid-devices=2 \
  /dev/nvme0n1p1 /dev/nvme1n1p1

# Verify
cat /proc/mdstat
mdadm --detail /dev/md1
```

### RAID 5 — Striping with Parity (Storage Nodes)
```bash
# Create RAID 5 with three drives
mdadm --create /dev/md2 \
  --level=5 \
  --raid-devices=3 \
  /dev/nvme0n1p1 /dev/nvme1n1p1 /dev/nvme2n1p1

# Add hot spare
mdadm --add /dev/md2 /dev/nvme3n1p1

# Verify
cat /proc/mdstat
mdadm --detail /dev/md2
```

### RAID 6 — Double Parity (Large Storage Arrays)
```bash
# Create RAID 6 with four drives
mdadm --create /dev/md3 \
  --level=6 \
  --raid-devices=4 \
  /dev/nvme0n1p1 /dev/nvme1n1p1 \
  /dev/nvme2n1p1 /dev/nvme3n1p1

# Verify
cat /proc/mdstat
mdadm --detail /dev/md3
```

### RAID 10 — Mirror plus Stripe (AI Model Serving)
```bash
# Create RAID 10 with four drives
mdadm --create /dev/md4 \
  --level=10 \
  --raid-devices=4 \
  /dev/nvme0n1p1 /dev/nvme1n1p1 \
  /dev/nvme2n1p1 /dev/nvme3n1p1

# Verify layout
cat /proc/mdstat
mdadm --detail /dev/md4
```

---

## Filesystem Creation and Mounting
```bash
# Create XFS filesystem (recommended for AI workloads)
mkfs.xfs /dev/md0

# Create mount point
mkdir -p /data/ai-training

# Mount array
mount /dev/md0 /data/ai-training

# Get UUID for fstab
blkid /dev/md0

# Add to /etc/fstab (use UUID not device name)
echo "UUID=<uuid-from-blkid> /data/ai-training xfs defaults 0 0" >> /etc/fstab

# Test fstab entry
mount -a

# Verify
df -h /data/ai-training
```

---

## Save RAID Configuration (CRITICAL — Never Skip)
```bash
# View current array configuration
mdadm --detail --scan

# Save to mdadm.conf
mkdir -p /etc/mdadm
mdadm --detail --scan | tee /etc/mdadm/mdadm.conf

# Rebuild initramfs — Debian/Ubuntu
update-initramfs -u

# Rebuild initramfs — RHEL/CentOS
dracut -f

# Verify mdadm.conf content
cat /etc/mdadm/mdadm.conf
```

---

## RAID Health Monitoring
```bash
# Quick status check
cat /proc/mdstat

# Detailed array status
mdadm --detail /dev/md0

# Check all arrays
mdadm --detail --scan

# Monitor with email alerts
mdadm --monitor --mail root@localhost --delay 60 /dev/md0 &

# Watch rebuild progress
watch -n 2 cat /proc/mdstat

# Check drive health with smartctl
smartctl -a /dev/nvme0n1
```

---

## Drive Failure Recovery
```bash
# Step 1 — Confirm failure
cat /proc/mdstat
mdadm --detail /dev/md1

# Step 2 — Remove failed drive
mdadm /dev/md1 --remove /dev/nvme1n1p1

# Step 3 — Replace physical drive (hot-swap if supported)

# Step 4 — Partition new drive
parted /dev/nvme1n1 mklabel gpt
parted /dev/nvme1n1 mkpart primary 0% 100%
parted /dev/nvme1n1 set 1 raid on

# Step 5 — Add new drive to array
mdadm /dev/md1 --add /dev/nvme1n1p1

# Step 6 — Monitor rebuild
watch -n 2 cat /proc/mdstat
```

---

## RAID Management Commands
```bash
# Stop an array
mdadm --stop /dev/md0

# Assemble array manually
mdadm --assemble /dev/md0 /dev/nvme0n1p1 /dev/nvme1n1p1

# Add spare drive
mdadm /dev/md0 --add /dev/nvme2n1p1

# Mark drive as faulty (testing)
mdadm /dev/md0 --fail /dev/nvme1n1p1

# Grow array (add drives to existing array)
mdadm --grow /dev/md0 --raid-devices=3 --add /dev/nvme2n1p1

# Check array consistency
mdadm --action=check /dev/md0
echo check > /sys/block/md0/md/sync_action

# View sync status
cat /sys/block/md0/md/sync_completed
```

---

## /proc/mdstat — Reading the Output
```bash
# Healthy RAID 10 — all drives up
md4 : active raid10 nvme3n1p1[3] nvme2n1p1[2] nvme1n1p1[1] nvme0n1p1[0]
      2000000000 blocks super 1.2 64K chunks 2 near-copies [4/4] [UUUU]

# Degraded RAID 1 — one drive failed
md1 : active raid1 nvme0n1p1[0]
      1000000000 blocks super 1.2 [2/1] [U_]   ← underscore = failed drive

# Rebuilding RAID 10
md4 : active raid10 nvme3n1p1[3] nvme2n1p1[2] nvme1n1p1[1] nvme0n1p1[0]
      2000000000 blocks super 1.2 [4/3] [UUU_]
      [=======>.............]  recovery = 35.2% (704512/2000000) finish=14.2min
```

---

## AI Bare Metal RAID Selection

| RAID | Use Case | Min Drives | Redundancy | Efficiency |
|------|----------|------------|------------|------------|
| 0 | AI training scratch | 2 | None | 100% |
| 1 | OS and boot drive | 2 | 1 drive | 50% |
| 5 | Storage nodes | 3 | 1 drive | 67% |
| 6 | Large arrays | 4 | 2 drives | 50% |
| 10 | AI inferencing | 4 | 1 per pair | 50% |

---

## Three Rules — Never Break
Always use GPT partitioning on drives above 2TB
Always save mdadm.conf and rebuild initramfs after creating arrays
Always monitor /proc/mdstat — degraded arrays need immediate action

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
