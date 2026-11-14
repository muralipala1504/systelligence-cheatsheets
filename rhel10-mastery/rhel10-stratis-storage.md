# Stratis Storage Management on RHEL 10 Cheatsheet
# RHEL 10 Mastery Series — Episode 9
# youtube.com/@SysTelligence

---

## 1. Installation & Prerequisites
```bash
# Install the daemon and CLI
sudo dnf install -y stratisd stratis-cli

# Enable and start the daemon (CRITICAL: stratis commands fail if daemon is not running)
sudo systemctl enable --now stratisd

# Verify the new disk is visible (e.g., /dev/nvme0n2)
lsblk

# Wipe the disk (DESTROYS ALL DATA ON THE DISK)
sudo wipefs --all /dev/nvme0n2

2. Creating a Pool

# Create a pool from an empty block device
sudo stratis pool create my-pool /dev/nvme0n2

# List the pool to verify
stratis pool list

3. Thin Provisioning & Filesystems

# Create a filesystem inside the pool
sudo stratis filesystem create my-pool my-fs

# List filesystems (Notice the 1 TiB size on a 10 GiB disk!)
stratis filesystem list

# Check the filesystem type (It is XFS under the hood)
sudo blkid /dev/stratis/my-pool/my-fs

4. Mounting and Persistence (The Gotcha)

# Create a mount point and mount
sudo mkdir -p /mnt/mystratis
sudo mount /dev/stratis/my-pool/my-fs /mnt/mystratis

# Add to /etc/fstab (CRITICAL: Use the UUID and the systemd flag)
# Find the UUID first:
sudo blkid /dev/stratis/my-pool/my-fs

# Add this line to /etc/fstab:
# UUID=<your-uuid> /mnt/mystratis xfs defaults,x-systemd.requires=stratisd.service 0 0

# Test the mount before rebooting
sudo umount /mnt/mystratis
sudo systemctl daemon-reload
sudo mount -a

5. Instant Snapshots (Copy-on-Write)

# Take a snapshot
sudo stratis filesystem snapshot my-pool my-fs my-fs-snap

# Mount the snapshot (it is read-write by default)
sudo mkdir -p /mnt/mystratis-snap
sudo mount /dev/stratis/my-pool/my-fs-snap /mnt/mystratis-snap

# Verify the file exists in the snapshot
cat /mnt/mystratis-snap/test.txt

6. Adding Storage to a Pool

# If the pool fills up, wipe a new disk and add it
sudo wipefs --all /dev/nvme0n3
sudo stratis pool add-data my-pool /dev/nvme0n3

# Verify the block devices
stratis blockdev list

7. Monitoring & Alerts

# Check pool usage and alerts
stratis pool list

# If you see an alert code (e.g., WS001), explain it
stratis pool explain WS001

# Check filesystem usage
stratis filesystem list

# Check the physical devices making up the pool
stratis blockdev list

Warning: A full thin pool causes write failures. Always monitor pool free space, not just filesystem free space.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
