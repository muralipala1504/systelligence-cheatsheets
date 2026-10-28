# Linux PAM Hardening Cheatsheet
# Linux Security Hardening Series — Episode 6
# youtube.com/@SysTelligence

---

## 1. The Legacy Trap (authselect)
```bash
# Check current auth profile
authselect current

# On AlmaLinux 9, if it says "No existing configuration detected":
authselect select minimal --force

# On RHEL 10 and AlmaLinux 10, PAM files are symlinks to /etc/authselect/
# NEVER edit /etc/pam.d/system-auth directly on RHEL 10. Use authselect.

2. Password Complexity (pwquality)

# Check if the package is installed
rpm -q libpwquality

# Test a password before enforcing
echo "password123" | pwscore
# Output: Password quality check failed: The password fails the dictionary check

# Configuration file
vi /etc/security/pwquality.conf
# Set minlen = 12, minclass = 3, etc.

3. Account Lockout (faillock)

# Check a user's lockout status (MUST use sudo)
sudo faillock --user <username>

# Unlock a user
sudo faillock --user <username> --reset

# Configuration file
vi /etc/security/faillock.conf
# deny = 3
# unlock_time = 600
# even_deny_root (WARNING: Can lock root out completely)

4. Password Aging and History

# Check current aging policy for a user
chage -l <username>

# Set maximum days and warning days
chage -M 90 -W 7 <username>

# Prevent password reuse (stores old hashes)
# Configured via pam_pwhistory in /etc/authselect/

5. Restricting su (pam_wheel)

# Edit the su config
vi /etc/pam.d/su

# Uncomment this line to restrict su to the wheel group:
auth required pam_wheel.so use_uid

# Verify wheel group members
getent group wheel

# Add a user to the wheel group
usermod -aG wheel <username>

6. The Service Account Gotcha
If automated jobs (like backups) run as a service user and fail authentication 3 times, faillock will lock the account silently, and the job will fail.

Fix: Add service accounts to the exclude list in /etc/security/faillock.conf.

7. PAM Logging

# RHEL 10 and Alma 10 (check the secure log)
sudo tail -n 20 /var/log/secure | grep pam

# Alma 9 (secure log is missing by default, use journald)
sudo journalctl -u sshd -n 20 | grep pam

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
