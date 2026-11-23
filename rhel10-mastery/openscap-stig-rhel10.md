# OpenSCAP and STIG Compliance on RHEL 10 Cheatsheet
# RHEL 10 Mastery Series — Episode 10
# youtube.com/@SysTelligence

---

## 1. Installation (Not installed by default)
```bash
sudo dnf install -y openscap-scanner scap-security-guide

2. The RHEL vs Alma Trap (Find the Path)
The file path differs by distribution. Do not guess it.

# RHEL 10 path
DS=/usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml

# AlmaLinux 10 path
DS=/usr/share/xml/scap/ssg/content/ssg-almalinux10-ds.xml

# Verify the file exists
ls -l "$DS"

3. List Available Profiles

oscap info --profiles "$DS"
# Look for: stig, cis, cis_server_l1, pci-dss, hipaa

4. Run the Scan (Read-Only)

# Set the profile (e.g., STIG)
P=xccdf_org.ssgproject.content_profile_stig

# Run the scan (takes about 2 minutes)
sudo oscap xccdf eval --profile $P --results /tmp/stig-results.xml --report /tmp/stig-report.html "$DS" > /tmp/stig-scan.txt 2>&1

# Exit code 2 means the scan finished, but rules failed.
echo "Exit code: $?"

5. Count the Results

# Count how many rules passed vs failed
rg -o '<result>[a-z]+</result>' /tmp/stig-results.xml | sort | uniq -c

6. The HTML Report

# Check the file sizes
ls -lh /tmp/stig-report.html /tmp/stig-results.xml

(The HTML is for the auditor. The XML is for automated tools).

7. The Remediation Trap

# Generate a bash script to fix the failures
sudo oscap xccdf generate fix --profile $P --fix-type bash --output /tmp/stig-fix.sh "$DS"

# WARNING: Read the script before running it! It is root-owned.
sudo less /tmp/stig-fix.sh

8. Targeted Fixes (One Rule at a Time)

# Find a specific failing rule
rg -B3 '^Result\s+fail' /tmp/stig-scan.txt | rg -i 'Title' | rg -i 'aide'

# Fix the rule manually (example: Install AIDE)
sudo dnf install -y aide

# Rescan just that one rule to prove it passed
sudo oscap xccdf eval --profile $P --rule xccdf_org.ssgproject.content_rule_package_aide_installed "$DS" | rg -i 'Title|Result'

9. Automation (Ansible)

# The security guide ships with Ansible playbooks
ls /usr/share/scap-security-guide/ansible/
# RHEL: rhel10-playbook-stig.yml
# Alma: almalinux10-playbook-stig.yml

# Run in check mode first
ansible-playbook -i inventory --check --diff /usr/share/scap-security-guide/ansible/rhel10-playbook-stig.yml

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence

