# Broken FSTAB Recovery Cheatsheet
# Certification to Production Reality — Episode 1
# youtube.com/@SysTelligence

---

## 1. The Exam Recovery (Emergency Mode)
```bash
# 1. Enter the root password at the prompt
# 2. Check filesystem state
mount | grep " / "

# 3. Remount root as read-write
mount -o remount,rw /

# 4. Edit the broken fstab
vi /etc/fstab

# 5. Reload systemd manager configuration
systemctl daemon-reload

# 6. Reboot
reboot

2. The Production Prevention (Systemd Units)
Create /etc/systemd/system/mnt-data.mount:

[Unit]
Description=Data Mount

[Mount]
What=/dev/vg_data/lv_data
Where=/mnt/data
Type=xfs
Options=defaults

[Install]
WantedBy=multi-user.target

# Enable and start the mount unit
systemctl enable --now mnt-data.mount

3. Modern Validation Tools

# List all UUIDs on the system
blkid

# Verify fstab syntax without rebooting
findmnt --verify

# Check for systemd mount unit errors
systemd-analyze verify mnt-data.mount

4. Auto-Repair on Boot (Kernel Parameters)
Add to /etc/default/grub (inside GRUB_CMDLINE_LINUX):

fsck.mode=force fsck.repair=yes

# Regenerate grub config
grub2-mkconfig -o /boot/grub2/grub.cfg

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence
