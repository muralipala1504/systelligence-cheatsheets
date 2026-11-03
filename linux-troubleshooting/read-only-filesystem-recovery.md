# Read-Only Filesystem Emergency Recovery Cheatsheet
# Linux Troubleshooting Series — Episode 12
# youtube.com/@SysTelligence

---

## 1. Detect the Problem
```bash
# Test if the filesystem is read-only
touch test.log
# (Output: touch: cannot touch 'test.log': Read-only file system)

# Check the current mount options for root
mount | grep ' / '

# Check kernel logs for the I/O error that caused the lock
dmesg -T | tail -n 50

2. The Quick Fix (Temporary)

# Remount the filesystem as read-write (only works if hardware is okay)
mount -o remount,rw /

# Verify the change
mount | grep ' / '

3. Hardware Check

# Check for SCSI or NVMe errors in dmesg
dmesg -T | grep -i -E 'scsi|ata|nvme'

# Check disk health with smartctl (if installed)
smartctl -a /dev/sda

4. Boot into Rescue Mode (GRUB)
Reboot and press e at the GRUB menu.

Find the line starting with linux and change ro to rw.

Append init=/sysroot/bin/sh to the end of the line.

Press Ctrl+X to boot.

5. XFS Repair (Must be unmounted)

# From rescue shell, remount sysroot read-write
mount -o remount,rw /sysroot

# Dry run first (no changes made)
xfs_repair -n /dev/mapper/rhel-root

# Actual repair
xfs_repair /dev/mapper/rhel-root

# If dirty log is an issue, mount and unmount to replay
mount /dev/mapper/rhel-root /mnt
umount /mnt

6. EXT4 Repair (Must be unmounted)

# Run fsck with automatic yes to all prompts
fsck -y /dev/mapper/rhel-root

# Force fsck on next boot via GRUB
# Append fsck.mode=force to the kernel line

7. Post-Recovery Verification

# Confirm root is mounted read-write
mount | grep ' / '

# Create and remove a test file
touch /test.log && rm /test.log

# Check dmesg for new errors
dmesg -T | tail -n 20

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
