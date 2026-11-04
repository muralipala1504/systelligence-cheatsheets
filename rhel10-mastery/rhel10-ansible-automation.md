# Ansible Automation on RHEL 10 Cheatsheet
# RHEL 10 Mastery Series — Episode 6
# youtube.com/@SysTelligence

---

## 1. The RHEL 10 Shift
```bash
# The standard "ansible" package is gone. You must install ansible-core.
sudo dnf install ansible-core

# Verify the binaries provided
rpm -ql ansible-core | grep bin/
# Output includes /usr/bin/ansible, ansible-playbook, etc.

# Install the Red Hat System Roles
sudo dnf install rhel-system-roles

2. The Inventory File

# Default location
sudo vi /etc/ansible/hosts

# Example inventory structure:
[webservers]
web01 ansible_host=192.168.1.10
web02 ansible_host=192.168.1.11

[databases]
db01 ansible_host=192.168.1.20

# Check the parsed inventory
ansible-inventory --list

3. SSH Setup (Passwordless)

# Generate SSH key on the control node
ssh-keygen -t ed25519

# Copy the key to the managed node
ssh-copy-id automation@web01

# On the managed node, configure passwordless sudo
sudo visudo -f /etc/sudoers.d/automation
# Add: automation ALL=(ALL) NOPASSWD: ALL

4. Ad-Hoc Commands

# Ping all hosts (proves SSH + Python work)
ansible all -m ping

# Run a command across a specific group
ansible webservers -m command -a "uptime"

# Run a command as root (become)
ansible webservers -m command -a "id" -b

5. Playbook Anatomy

# site.yml
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  tasks:
    - name: Install Nginx
      ansible.builtin.dnf:
        name: nginx
        state: present
    - name: Start Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

# Run the playbook
ansible-playbook site.yml

# Check syntax first
ansible-playbook site.yml --syntax-check

# Dry run (check mode)
ansible-playbook site.yml --check

6. RHEL System Roles

# View available roles
ls /usr/share/doc/rhel-system-roles/

# Example: Automating Networking (net.yml)
# roles:
#   - rhel-system-roles.network
# vars:
#   network_connections:
#     - name: ens160
#       type: ethernet
#       state: up
#       ip:
#         address: 192.168.1.50/24

# Example: Automating SELinux
# roles:
#   - rhel-system-roles.selinux
# vars:
#   selinux_state: enforcing
#   selinux_booleans:
#     - name: httpd_can_network_connect
#       state: on

7. Idempotency (The Play Recap)

# First run (changed=2)
PLAY RECAP
web01 : ok=3 changed=2 unreachable=0 failed=0

# Second run (changed=0)
PLAY RECAP
web01 : ok=3 changed=0 unreachable=0 failed=0

If the second run shows changed=0, your playbook is idempotent.

8. Ansible Vault

# Create a new encrypted file
ansible-vault create secrets.yml

# Encrypt an existing file
ansible-vault encrypt vars.yml

# View or edit an encrypted file
ansible-vault view secrets.yml
ansible-vault edit secrets.yml

# Run a playbook that requires a vault password
ansible-playbook site.yml --ask-vault-pass
Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
