# Dell iDRAC vs HPE iLO Cheatsheet
# Bare Metal Mastery Series — Part 23
# youtube.com/@SysTelligence

---

## 1. Find the BMC (Out-of-Band Management)
*   **Default IPs/PWs:** Do not assume factory defaults. Check the pull-out tag on the server, the BIOS/LCD menu, or the DHCP lease list. Change the password immediately upon onboarding.
*   **Dell Tool:** RACADM
*   **HPE Tool:** iLO CLI over SSH

## 2. Essential IPMI Commands (Works on both)
```bash
# Check basic chassis and power status
ipmitool -I lanplus -H <BMC_IP> -U <USER> chassis status
ipmitool -I lanplus -H <BMC_IP> -U <USER> power status

# List sensors and System Event Log (SEL)
ipmitool -I lanplus -H <BMC_IP> -U <USER> sdr list
ipmitool -I lanplus -H <BMC_IP> -U <USER> sel list

# Activate Serial over LAN (SOL) console
ipmitool -I lanplus -H <BMC_IP> -U <USER> sol activate

3. Serial Console Setup (Linux)
To see kernel panics when the screen is dead, you must configure the OS to output to the serial port:

# Add console parameter to kernel
sudo grubby --update-kernel=ALL --args="console=tty0 console=ttyS0,115200n8"

# Enable the serial getty service
systemctl enable --now serial-getty@ttyS0.service

4. Redfish API (Modern REST Automation)

# Query the root of the API
curl -sk -u USER:PASS https://<BMC_IP>/redfish/v1/ | jq

# List systems (Note the IDs differ: Dell uses "System.Embedded.1", HPE uses "1")
curl -sk -u USER:PASS https://<BMC_IP>/redfish/v1/Systems | jq '.Members[]."@odata.id"'

# Query power state and health
curl -sk -u USER:PASS https://<BMC_IP>/redfish/v1/Systems/<SYSTEM_ID> | jq '{PowerState, Status}'

5. Automation Tools
Ansible: Install community.general for generic Redfish modules, or dellemc.openmanage for Dell-specific features.

Terraform: Check the registry for current iDRAC and iLO providers.

6. Security Checklist
Put the BMC on a dedicated management VLAN.

Use a jump host to access it. Never expose a BMC directly to the internet.

Disable IPMI over LAN if you only use Redfish.

Use trusted TLS certificates.

Use directory accounts with MFA where supported.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
