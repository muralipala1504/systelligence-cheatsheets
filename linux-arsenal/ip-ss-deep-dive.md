# ip and ss Deep Dive Cheatsheet
# Linux Arsenal Series — Episode 2
# youtube.com/@SysTelligence

---

## ip Command — Complete Reference

### ip addr — Address Management
```bash
ip addr show                                    # Show all interfaces
ip addr show eth0                               # Show specific interface
ip addr add 192.168.1.10/24 dev eth0           # Add IP address
ip addr del 192.168.1.10/24 dev eth0           # Remove IP address
ip addr flush dev eth0                          # Remove all IPs from interface
```

### ip link — Interface Management
```bash
ip link show                                    # Show all interfaces
ip link set eth0 up                             # Bring interface up
ip link set eth0 down                           # Bring interface down
ip link set eth0 mtu 9000                       # Set jumbo frames
ip link set eth0 promisc on                     # Enable promiscuous mode
ip link set eth0 txqueuelen 10000              # Set transmit queue length
```

### ip route — Routing Table
```bash
ip route show                                   # Show routing table
ip route add 192.168.2.0/24 via 192.168.1.1   # Add static route
ip route del 192.168.2.0/24                    # Delete static route
ip route get 8.8.8.8                           # Show exact path to destination
ip route flush cache                            # Flush routing cache
ip route show table all                         # Show all routing tables
```

### ip neigh — ARP Table
```bash
ip neigh show                                   # Show ARP table
ip neigh show dev eth0                          # ARP for specific interface
ip neigh add 192.168.1.1 lladdr AA:BB:CC:DD:EE:FF dev eth0  # Add static ARP
ip neigh del 192.168.1.1 dev eth0             # Delete ARP entry
ip neigh flush dev eth0                         # Flush ARP table
```

### ip netns — Network Namespaces
```bash
ip netns list                                   # List all namespaces
ip netns add ns1                                # Create namespace
ip netns del ns1                                # Delete namespace
ip netns exec ns1 ip addr show                 # Run command in namespace
ip link set veth0 netns ns1                    # Move interface to namespace
```

### Bonding Configuration
```bash
ip link add bond0 type bond                     # Create bond interface
ip link set bond0 type bond mode active-backup # Set bonding mode
ip link set eth0 master bond0                   # Add interface to bond
ip link set eth1 master bond0                   # Add second interface
ip addr add 192.168.1.10/24 dev bond0          # Assign IP to bond
ip link set bond0 up                            # Bring bond up
```

### VLAN Configuration
```bash
ip link add link eth0 name eth0.10 type vlan id 10   # Create VLAN 10
ip link add link eth0 name eth0.20 type vlan id 20   # Create VLAN 20
ip addr add 192.168.10.1/24 dev eth0.10              # Assign IP to VLAN
ip link set eth0.10 up                                # Bring VLAN up
ip link del eth0.10                                   # Delete VLAN
```

### Six Step Network Troubleshooting Workflow
```bash
# Step 1 — Check interface state
ip link show

# Step 2 — Verify IP assignment
ip addr show

# Step 3 — Check default route
ip route show

# Step 4 — Trace exact packet path
ip route get <destination_ip>

# Step 5 — Check ARP resolution
ip neigh show

# Step 6 — Ping each hop
ping <gateway>
ping <next_hop>
ping <destination>
```

---

## ss Command — Complete Reference

### Basic Usage
```bash
ss -tulpn                                       # All listening ports with process
ss -t                                           # All TCP sockets
ss -u                                           # All UDP sockets
ss -l                                           # Listening sockets only
ss -p                                           # Show process information
ss -n                                           # No DNS resolution
ss -s                                           # Socket summary statistics
ss -o                                           # Show timer information
```

### State Filters
```bash
ss -t state established                         # Active TCP connections
ss -t state time-wait                           # TIME_WAIT connections
ss -t state listen                              # Listening sockets
ss -t state close-wait                          # CLOSE_WAIT connections
ss -t state syn-sent                            # Outbound connection attempts
```

### Address and Port Filters
```bash
ss -t dst port 443                              # Connections to HTTPS
ss -t dst port 3306                             # Connections to MySQL
ss -t src 192.168.1.10                         # From specific local IP
ss -t dst 10.0.0.1                             # To specific remote IP
ss -tulpn | grep :80                            # Check port 80
```

### Production Diagnostic Commands
```bash
# Port conflict — find what owns a port
ss -tulpn | grep :8080

# Count established connections
ss -t state established | wc -l

# Count TIME_WAIT sockets
ss -t state time-wait | wc -l

# Live socket monitoring
watch -n 1 ss -s

# All connections to MySQL
ss -t state established dst port 3306

# Security audit — all listeners
ss -tulpn

# Connection to specific remote host
ss -t dst 10.0.0.50
```

### Fix TIME_WAIT Accumulation
```bash
# Check current count
ss -t state time-wait | wc -l

# Enable TCP TIME_WAIT reuse
sysctl net.ipv4.tcp_tw_reuse=1

# Make persistent
echo "net.ipv4.tcp_tw_reuse = 1" >> /etc/sysctl.conf
sysctl -p
```

---

## Kickstart — Install Tools
```bash
%packages
iproute
iproute-tc
%end
# Note: ip and ss are part of iproute — pre-installed on all modern Linux
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Show all IPs | ip addr show |
| Add IP | ip addr add X.X.X.X/24 dev ethX |
| Show routes | ip route show |
| Trace packet path | ip route get X.X.X.X |
| Show ARP | ip neigh show |
| Create VLAN | ip link add link eth0 name eth0.10 type vlan id 10 |
| All listeners | ss -tulpn |
| Active connections | ss -t state established |
| Socket summary | ss -s |
| Find port owner | ss -tulpn \| grep :PORT |
| Live monitor | watch -n 1 ss -s |

---
*Questions? Drop them in the YouTube comments — I read every single one.*
*📺 youtube.com/@SysTelligence*
