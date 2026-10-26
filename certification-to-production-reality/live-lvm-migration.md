# Live LVM Storage Migration Cheatsheet
# Certification to Production Reality — Episode 2
# youtube.com/@SysTelligence

---

## 1. Discover New Storage
```bash
# Scan for new hardware
rescan-scsi-bus.sh

# Or software scan specific device
echo 1 > /sys/block/sdX/device/rescan

# Verify multipath topology and tree
multipath -ll
lsblk

Rule: Always target the multipath alias (e.g. /dev/mapper/mpatha), never the raw sdX device.

2. Extend the Volume Group

# Initialize the new LUN
pvcreate /dev/mapper/mpathb

# Add it to the existing volume group
vgextend vg_data /dev/mapper/mpathb

# Verify the physical volumes
vgs -o +pv_name

3. Live Block Relocation (pvmove)

# Move data in the background
pvmove -b /dev/mapper/mpatha /dev/mapper/mpathb

# Monitor the progress
lvs -a -o +devices,copy_percent

# If a SAN blip occurs, abort safely
pvmove --abort

4. Inspect I/O During Migration

# Check active disk throughput
iostat -xz 1

# Check kernel messages for path issues
dmesg -T

Warning: Throttle pvmove on busy databases to prevent storage controller saturation.

5. Retire the Legacy Storage

# Remove the old physical volume
vgreduce vg_data /dev/mapper/mpatha
pvremove /dev/mapper/mpatha

# Verify zero extents remain
pvs

# Flush multipath and delete the device
multipath -f mpatha
echo 1 > /sys/block/sdX/device/delete

6. Grow the Logical Volume and Filesystem

# Expand the LV to use all free space
lvextend -l +100%FREE /dev/vg_data/lv_data

# Grow the XFS filesystem online (use mount point)
xfs_growfs /mountpoint

Rule: XFS can only grow forward. It cannot be shrunk.

7. Verify Production Persistence

# Check space and filesystem type
df -Th /mountpoint

# Check XFS geometry
xfs_info /mountpoint

# Verify UUID for /etc/fstab
blkid /dev/vg_data/lv_data

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence
