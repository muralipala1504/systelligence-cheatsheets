# The 50 Degree Freezer Hack: IoT Hardening Cheatsheet
# Linux Troubleshooting Series — Episode 14
# youtube.com/@SysTelligence

---

## 1. The Attack Vector (Default Credentials & Exposed IPs)
```bash
# Find exposed management interfaces on your network
# (Only scan networks you own or have permission to test)
nmap -p 80,443 192.168.1.0/24

# Check Shodan or Censys for your public IP ranges
# (Look for Danfoss, Copeland, or Dixell interfaces exposed)

Rule: If a controller has a public IP and a default password, it is already compromised.

2. The Mechanism (Abusing Defrost Mode)
The attackers didn't use a zero-day. They logged in and forced the system into continuous defrost mode, disabling cooling and activating heating elements.

Setpoint: Pushed to 50°C.

Defrost Cycle: Extended from 1 hour to 8 hours.

Alarms: Delayed or disabled.

3. The Hardening (Network Segmentation)

[ Corporate IT Network ]
          |
     [ Firewall ]
          |
[ IoT/OT VLAN (Isolated) ]

Rule 1: Put all IoT/OT devices on a separate VLAN.

Rule 2: No direct internet exposure. No port forwarding.

Rule 3: Vendor access must go through a VPN with MFA.

4. Firmware Updates
A fix existed for these controllers since August 2025. Over 2,700 devices were still unpatched in October 2026.

Action: Create a schedule to check vendor firmware updates quarterly.

5. Detection & Response
Alert on anomalies: Set alerts for defrost cycles running outside their schedule, or setpoints jumping to extreme values.

Log review: Check access logs for unknown IPs logging into the controller.

Isolate: If compromised, disconnect the controller from the network immediately.

6. The Lesson
If it has an IP address, it is a target. IT and OT are one network. They need one security model.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
