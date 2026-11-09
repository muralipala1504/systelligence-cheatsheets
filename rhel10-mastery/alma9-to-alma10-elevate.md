# Alma 9 to Alma 10 Upgrade (ELevate) Cheatsheet
# RHEL 10 Mastery Series — Episode 8
# youtube.com/@SysTelligence

---

## 1. The Pre-Flight Checklist
```bash
# MUST take a VM snapshot or bare metal backup before starting
# Ensure you have console access (SSH will drop during reboot)
# Check free disk space
df -h

# Update the current system to the latest Alma 9.x packages
sudo dnf update -y
sudo reboot

2. Install ELevate and Leapp

# Install the ELevate repository
sudo dnf install -y http://repo.almalinux.org/elevate/elevate-release-latest-el9.noarch.rpm

# Install the Leapp upgrade engine and AlmaLinux data
sudo dnf install -y leapp-upgrade leapp-data-almalinux

3. Run the Pre-Upgrade Assessment

# This does not change the system, it only scans for issues
sudo leapp preupgrade

Read the report overview at the bottom of the output. Look for "Inhibitors". If any exist, you must fix them before proceeding.

4. Read the Report

# View the full report
sudo less /var/log/leapp/leapp-report.txt

# Check for specific warnings (e.g., Berkeley DB deprecation)
grep -i "Risk Factor" /var/log/leapp/leapp-report.txt

(If you have inhibitors, you might need to answer questions using sudo leapp answer --section <section_name>.confirm=True)

5. Execute the Upgrade

# Run the upgrade (prepares the packages)
sudo leapp upgrade

# Reboot to start the actual upgrade process
sudo reboot

(The system will boot into the ELevate initramfs. This takes 15-20 minutes. Do NOT interrupt the reboot.)

6. Verify the Upgrade

# Check the new OS version
cat /etc/redhat-release
# Expected: AlmaLinux release 10.x

# Check the new kernel
uname -r
# Expected: 6.12.0-...

# Check for failed services
systemctl --failed

# Check for leftover Alma 9 packages
rpm -qa | grep el9

7. Post-Upgrade Cleanup (Optional)

# Remove the Leapp tools (after confirming the system is stable)
sudo dnf remove leapp-upgrade leapp-data-almalinux

# Remove old Alma 9 kernels (after confirming the new kernel boots)
sudo dnf remove kernel-core-5.14.0-...

Always check the official ELevate documentation before removing packages.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence

