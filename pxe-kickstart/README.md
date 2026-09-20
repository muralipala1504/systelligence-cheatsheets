# PXE Boot + Kickstart Config Files

Production config files from the SysTelligence PXE Boot + Kickstart episode.

📺 Video: [PXE Boot + Kickstart: Deploy 100 Linux Servers Automatically](https://youtube.com/@SysTelligence)

---

## Files in this folder

| File | Purpose |
|------|---------|
| `dhcpd.conf` | DHCP server config for PXE boot |
| `ks.cfg` | Kickstart file for RHEL/CentOS unattended install |
| `preseed.cfg` | Preseed file for Debian/Ubuntu unattended install |
| `pxelinux.cfg/default` | PXE boot menu config |

---

## Quick Reference

```bash
# Verify DHCP is running
systemctl status dhcpd

# Verify TFTP is running
systemctl status tftp

# Check PXE boot logs
tail -f /var/log/messages | grep PXE
```

---
*Questions? Drop them in the YouTube comments.*
