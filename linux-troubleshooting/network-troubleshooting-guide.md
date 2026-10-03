# Linux Network Troubleshooting Cheatsheet
# Linux Troubleshooting Series — Episode 7
# youtube.com/@SysTelligence

---

## Five Layer Diagnostic Approach
Layer 1: Physical/Link — interface state, errors, cable
Layer 2: IP/Routing — address, default route, gateway
Layer 3: Transport — port binding, socket state
Layer 4: Application — service response, upstream
Layer 5: External — DNS, firewall, load balancer
Always diagnose bottom up — never skip layers


---

## Layer 1 — Interface Diagnosis
```bash
# Check interface state
ip link show

# Check IP address assignment
ip addr show

# Interface with statistics
ip -s link show eth0

# Check physical link status
ethtool eth0

# Bring interface up
ip link set eth0 up

# Check NIC errors
ip -s link show eth0 | grep -E "RX|TX|errors"

# Watch interface stats live
watch -n 1 'ip -s link show eth0'
```

---

## Layer 2 — Routing Diagnosis
```bash
# Show routing table
ip route show

# Most useful routing command — exact kernel decision
ip route get 8.8.8.8
ip route get 192.168.1.100

# Test gateway connectivity
ping -c 4 $(ip route | grep default | awk '{print $3}')

# Hop by hop path analysis
traceroute destination.com

# Live continuous traceroute with packet loss
mtr destination.com
mtr --report destination.com        # Single report mode
mtr --report-cycles 10 destination.com

# Add static route
ip route add 10.0.0.0/8 via 192.168.1.1

# Delete route
ip route del 10.0.0.0/8
```

---

## Layer 3 — Socket Diagnosis with ss
```bash
# All listening ports with process
ss -tulpn

# Check specific port
ss -tulpn | grep :80
ss -tulpn | grep :443
ss -tulpn | grep :22

# All established connections
ss -t state established

# Socket summary
ss -s

# Count TIME_WAIT sockets
ss -t state time-wait | wc -l

# Connections to specific destination port
ss -t dst port 3306

# TCP internal stats (retransmits)
ss -i

# Watch connections live
watch -n 2 'ss -s'
```

---

## DNS Diagnosis
```bash
# Basic DNS lookup
dig hostname.com

# Quick IP only
dig hostname.com +short

# Query specific DNS server
dig @8.8.8.8 hostname.com
dig @1.1.1.1 hostname.com

# Reverse DNS lookup
dig -x 192.168.1.10

# Check all DNS records
dig hostname.com ANY

# Check MX records
dig hostname.com MX

# systemd-resolved status
resolvectl status
resolvectl query hostname.com

# Check resolver config
cat /etc/resolv.conf
rg "nameserver" /etc/resolv.conf

# Test DNS vs direct IP
curl -I http://hostname.com     # DNS
curl -I http://192.168.1.10     # Direct IP
# If IP works but hostname fails = DNS problem
```

---

## Packet Capture with tcpdump
```bash
# Capture on interface by port
tcpdump -i eth0 port 80 -n

# Capture traffic to/from specific IP
tcpdump -i eth0 host 10.0.0.5 -n

# Capture with verbose output
tcpdump -i eth0 port 443 -nv

# Save capture to file
tcpdump -i eth0 port 80 -w /tmp/capture.pcap

# Read saved capture
tcpdump -r /tmp/capture.pcap

# Search capture with ripgrep
tcpdump -r /tmp/capture.pcap -A | rg "GET|POST"

# Capture all interfaces
tcpdump -i any port 8080 -n

# Capture with packet count limit
tcpdump -i eth0 port 80 -n -c 100

# Show SYN packets only (new connections)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0' -n
```

---

## Connection Refused vs Timeout

### Connection Refused
```bash
# Immediate error — network works, service not listening
# Diagnostic:
ss -tulpn | grep :PORT          # Check if port is listening
systemctl status servicename    # Check if service is running
journalctl -u servicename -b    # Check service logs
```

### Connection Timeout
```bash
# No response — firewall dropping or wrong path
# Diagnostic:
traceroute destination           # Find where packets stop
mtr destination                  # Live path analysis
nft list ruleset | rg "drop"    # Check firewall drop rules
tcpdump -i eth0 port PORT -n    # Confirm packets arriving
```

---

## Network Performance Diagnosis
```bash
# Bandwidth test between two servers
# Server side:
iperf3 -s

# Client side:
iperf3 -c server-ip
iperf3 -c server-ip -t 30       # 30 second test
iperf3 -c server-ip -P 4        # 4 parallel streams

# TCP retransmits per connection
ss -i | grep retrans

# Interface drop counters
ip -s link show eth0

# Live network IO in btop
btop

# Check NIC ring buffer
ethtool -g eth0

# Check NIC offload settings
ethtool -k eth0
```

---

## Firewall Diagnosis with nftables
```bash
# List complete ruleset
nft list ruleset

# Find drop rules
nft list ruleset | rg "drop"

# Find rules for specific port
nft list ruleset | rg "8080"

# List with handles for deletion
nft list ruleset -a

# Add temporary test rule
nft add rule inet filter input tcp dport 8080 accept

# Verify connectivity after adding rule
curl -I http://localhost:8080

# Remove temporary rule by handle
nft delete rule inet filter input handle NUMBER

# Check iptables (legacy)
iptables -L -n -v
iptables -L INPUT -n -v
```

---

## Ten Step Troubleshooting Sequence
```bash
# Step 1 — Interface state
ip link show

# Step 2 — IP address
ip addr show

# Step 3 — Default route
ip route show

# Step 4 — Gateway connectivity
ping -c 4 GATEWAY_IP

# Step 5 — Routing decision
ip route get DESTINATION_IP

# Step 6 — Port listening
ss -tulpn | grep :PORT

# Step 7 — Local service response
curl -I http://localhost:PORT

# Step 8 — DNS resolution
dig HOSTNAME +short

# Step 9 — Firewall rules
nft list ruleset | rg "PORT\|drop"

# Step 10 — Packet capture
tcpdump -i eth0 port PORT -n -c 50
```

---

## Quick Reference Card

| Layer | Problem | Command |
|-------|---------|---------|
| Physical | Interface down | ip link show |
| Physical | No link detected | ethtool eth0 |
| IP | No IP address | ip addr show |
| Routing | No default route | ip route show |
| Routing | Wrong path | ip route get DEST |
| Transport | Port not listening | ss -tulpn |
| Application | Service not responding | curl localhost:PORT |
| DNS | Name not resolving | dig @8.8.8.8 hostname |
| Firewall | Packets dropped | nft list ruleset |
| Path | Packets not arriving | tcpdump -i eth0 port N |

---

## Modern Tool Stack for Network Diagnosis
```bash
ss          # replaces netstat
ip          # replaces ifconfig, route, arp
mtr         # replaces traceroute
rg          # replaces grep for searching outputs
btop        # replaces top for network IO view
dig         # replaces nslookup
nft         # replaces iptables
```

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
