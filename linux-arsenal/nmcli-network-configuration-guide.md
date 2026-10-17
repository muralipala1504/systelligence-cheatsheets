# nmcli Network Configuration Cheatsheet
# Linux Arsenal Series — Episode 7
# youtube.com/@SysTelligence

---

## nmcli Essential Commands
```bash
# List all connections
nmcli connection show

# Check device status (interface, state, connection)
nmcli device status

# Create a new static ethernet connection
nmcli connection add type ethernet ifname eth0 con-name eth0-static

# Configure IP, gateway, and DNS
nmcli connection modify eth0-static ipv4.addresses 192.168.1.50/24
nmcli connection modify eth0-static ipv4.gateway 192.168.1.1
nmcli connection modify eth0-static ipv4.dns "8.8.8.8 1.1.1.1"

# Activate the connection
nmcli connection up eth0-static

# Verify details
nmcli connection show eth0-static

Network Bonding
# Create bond0 with active-backup
nmcli connection add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup"

# Add physical interfaces as slaves
nmcli connection add type ethernet slave-type bond con-name bond0-port1 ifname eth0 master bond0
nmcli connection add type ethernet slave-type bond con-name bond0-port2 ifname eth1 master bond0

# LACP (802.3ad) requires switch config
nmcli connection modify bond0 bond.options "mode=802.3ad,miimon=100"

# Verify
ip link show bond0
cat /proc/net/bonding/bond0

VLAN Configuration
# Create VLAN 10 on eth0
nmcli connection add type vlan con-name eth0.10 ifname eth0.10 dev eth0 id 10
nmcli connection modify eth0.10 ipv4.addresses 10.0.10.50/24
nmcli connection up eth0.10

# Verify
ip link show eth0.10

Network Bridges (KVM / Containers)
	# Create br0
nmcli connection add type bridge con-name br0 ifname br0

# Assign IP to bridge
nmcli connection modify br0 ipv4.addresses 192.168.1.100/24 ipv4.method manual

# Add eth0 to the bridge
nmcli connection add type ethernet slave-type bridge con-name br0-port1 ifname eth0 master br0

# Verify (brctl is deprecated, use bridge link)
ip link show master br0
bridge link show

Static Routes & Policy Routing

# Persistent static route via nmcli
nmcli connection modify eth0-static +ipv4.routes "10.10.0.0/16 192.168.1.254"

# Temporary runtime route
ip route add 10.10.0.0/16 via 192.168.1.254 dev eth0

# Verify exact kernel decision
ip route get 10.10.0.55

# Policy routing (source-based)
ip rule add from 192.168.1.50 table 100
ip route add default via 192.168.1.254 table 100

# View all tables
ip route show table all

DNS Configuration
# Persistent via nmcli (RECOMMENDED)
nmcli connection modify eth0-static ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection modify eth0-static ipv4.dns-search "example.com"

# systemd-resolved (RHEL 8/9 requires enabling)
systemctl enable --now systemd-resolved
resolvectl status
resolvectl query google.com

# Note: Do NOT edit /etc/resolv.conf directly

Performance Tuning
# Install required tools
dnf install ethtool

# MTU / Jumbo Frames (end-to-end support required)
ip link set eth0 mtu 9000
nmcli connection modify eth0-static 802-3-ethernet.mtu 9000

# Ring Buffers
ethtool -g eth0            # Check
ethtool -G eth0 rx 4096 tx 4096  # Set

# Hardware Offloading
ethtool -k eth0            # Check TSO, GSO, GRO

# Network Queues
ethtool -l eth0            # Check
ethtool -L eth0 combined 8 # Set to match CPU cores

Bonding vs Teaming
# Bonding (Kernel, default, recommended by Red Hat)
nmcli connection add type bond ...

# Teaming (Userspace, JSON config)
dnf install teamd
nmcli connection add type team ...

Troubleshooting Modern Stack
# Install mtr and tcpdump
dnf install mtr tcpdump

# 1. Exact routing decision
ip route get 8.8.8.8

# 2. Listening ports (replaces netstat)
ss -tulpn

# 3. Live path analysis (replaces traceroute)
mtr 8.8.8.8

# 4. Packet capture
tcpdump -i eth0 port
 80 -n

# 5. Physical link status
ethtool eth0

nftables Integration
# Allow traffic only on management VLAN
nft add rule inet filter input ip saddr 10.0.10.0/24 accept

# Restrict SSH to bond0
nft add rule inet filter input iif bond0 tcp dport 22 accept

# Block inter-VLAN routing
nft add rule inet filter forward ip saddr 10.0.10.0/24 ip daddr 10.0.20.0/24 drop

# View ruleset
nft list ruleset

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence


