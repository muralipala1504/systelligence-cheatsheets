# Modern Linux Commands Cheatsheet
# Linux Arsenal Series — Episode 1
# youtube.com/@SysTelligence

## Replace Deprecated Commands with Modern Alternatives

| Deprecated | Modern | Install |
|------------|--------|---------|
| ifconfig | ip | iproute2 (pre-installed) |
| netstat | ss | iproute2 (pre-installed) |
| top | btop | dnf/apt install btop |
| df | duf | dnf/apt install duf |
| find | fd | dnf/apt install fd-find |
| grep -r | rg (ripgrep) | dnf/apt install ripgrep |
| du | ncdu | dnf/apt install ncdu |
| cat logs | jq | dnf/apt install jq |

---

## ip — Network Interface Management
```bash
ip addr show                    # Show all interfaces and IPs
ip link show                    # Show interface status and MTU
ip route show                   # Show routing table
ip neigh show                   # Show ARP table
ip addr add 192.168.1.10/24 dev eth0   # Add IP to interface
ip link set eth0 up             # Bring interface up
```

## ss — Socket Statistics
```bash
ss -tulpn                       # All listening ports with process
ss -t state established         # All established TCP connections
ss -t dst port 443              # Connections to port 443
ss -s                           # Socket summary statistics
ss -tulpn | grep :80            # Check if port 80 is listening
```

## btop — System Monitor
```bash
btop                            # Launch btop
# Inside btop:
# F2 or o — Options menu
# F9 — Kill process
# m — Toggle memory display
# q — Quit
```

## duf — Disk Usage
```bash
duf                             # Show all mount points
duf /var                        # Show specific path
duf --only local                # Show only local filesystems
duf --hide special              # Hide special filesystems
```

## fd — File Search
```bash
fd filename                     # Find by name
fd --extension log /var/log     # Find by extension
fd --changed-before 7days       # Files older than 7 days
fd --size +100m                 # Files larger than 100MB
fd -x rm {}                     # Find and execute command
```

## rg — ripgrep
```bash
rg ERROR /var/log               # Search for ERROR in logs
rg -i warning /var/log          # Case insensitive search
rg --type log "timeout"         # Search only log files
rg -l "pattern"                 # List files with matches only
rg -c "ERROR"                   # Count matches per file
```

## ncdu — Disk Usage Navigator
```bash
ncdu /                          # Scan entire filesystem
ncdu /var                       # Scan specific directory
# Inside ncdu:
# Enter — drill into directory
# d — delete selected
# q — quit
```

## jq — JSON Log Parser
```bash
cat app.log | jq .              # Pretty print JSON
cat app.log | jq .level         # Extract specific field
cat app.log | jq 'select(.level == "ERROR")'   # Filter by value
cat app.log | jq '{time: .timestamp, msg: .message}'  # Custom output
cat app.log | jq -r .message    # Raw output without quotes
```

## journalctl — Systemd Log Management
```bash
journalctl -u nginx --since today       # Service logs since today
journalctl -p err -b                    # All errors since last boot
journalctl -f -u sshd                   # Follow SSH logs live
journalctl --disk-usage                 # Check journal disk usage
journalctl --vacuum-size=500M           # Clean old journal entries
journalctl -b -1                        # Logs from previous boot
```

## systemd-analyze — Boot Performance
```bash
systemd-analyze                         # Total boot time
systemd-analyze blame                   # Services by startup time
systemd-analyze critical-chain          # Boot dependency chain
systemd-analyze plot > boot.svg         # Visual boot timeline
```

---

## Kickstart — Install All Tools
```bash
%packages
btop
duf
fd-find
ripgrep
ncdu
jq
%end
```

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
