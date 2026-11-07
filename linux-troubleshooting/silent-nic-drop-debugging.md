# The Silent NIC Drop: Packet Loss Debugging Cheatsheet
# Linux Troubleshooting Series — Episode 15
# youtube.com/@SysTelligence

---

## 1. Check the Physical Layer (CRC Errors)
```bash
# Check interface statistics for drops, errors, and missed packets
ip -s link show ens160

# Check hardware statistics (CRC errors, physical drops)
sudo ethtool -S ens160

Look for rising CRC errors or physical drops. If these are incrementing, check the cable, transceiver, or switch port. Virtual machines may not expose these counters.

2. Check the Ring Buffer

# Check current vs maximum ring buffer size
ethtool -g ens160

# Increase the receive ring buffer (if drops are proven)
sudo ethtool -G ens160 rx 4096

If the ring buffer is full, the NIC drops packets before the kernel sees them. Increasing the buffer helps with bursts, but adds latency and memory use.

3. Check Kernel Drops (Softnet)

# Check the per-CPU backlog queue (values are in hex)
cat /proc/net/softnet_stat

# Check the maximum backlog size
sysctl net.core.netdev_max_backlog

Column 1 is processed, Column 2 is dropped, Column 3 is time_squeeze. If Column 2 or 3 is increasing, the CPU is choking on network processing.

4. Check TCP Retransmits

# Check global TCP retransmit counters
nstat -az | grep -i retrans

# Check per-connection retransmits
ss -ti

A retransmit is the symptom of loss on the network. Look for the retrans field in ss -ti.

5. The Offload Trap (TSO / GSO / GRO)

# Check offload settings
ethtool -k ens160

Hardware offloads (TSO/GSO/GRO) can make packet captures show frames larger than the MTU. This is normal. Turn them off only to test a theory.

6. The MTU Trap (PMTUD Blackhole)

# Test if large packets are dropping (Do Not Fragment)
ping -M do -s 1472 <destination>

If small pings work but large packets fail, you have an MTU mismatch. Check if ICMP is blocked on the path.

7. Tuning RPS / RFS (Multi-Core Processing)

# Check if Receive Packet Steering is enabled
cat /sys/class/net/ens160/queues/rx-0/rps_cpus

All zeros means RPS is off. Setting a CPU mask allows the kernel to spread network processing across multiple cores.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
