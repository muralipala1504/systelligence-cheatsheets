# NUMA and CPU Pinning for AI Cheatsheet
# Bare Metal Mastery Series — Part 21
# youtube.com/@SysTelligence

---

## 1. Discover the Topology
```bash
# Check the NUMA hardware layout (nodes, CPUs, memory, distances)
numactl --hardware

# Check the current CPU topology
lscpu

# List every CPU and its assigned NUMA node
lscpu -e

# Check the NUMA balancing status (0 = off, 1 = on)
cat /proc/sys/kernel/numa_balancing

(Note: Virtual machines and WSL2 often hide the NUMA topology. numactl --hardware may show "No NUMA available on this system".)

2. Check Device Affinity (GPU & NIC)

# Check the GPU topology and NUMA affinity
nvidia-smi topo -m

# Check the PCI address and NUMA node of a GPU
cat /sys/bus/pci/devices/0000:01:00.0/numa_node

# Check the NUMA node of a network interface
cat /sys/class/net/ens160/device/numa_node

3. Pin CPU and Memory

# Bind a process to a specific CPU node
numactl --cpunodebind=0 <command>

# Bind a process to a specific memory node
numactl --membind=0 <command>

# Bind both CPU and memory (best for AI workloads)
numactl --cpunodebind=0 --membind=0 <command>

# Verify the binding
numactl --show

4. Memory Policies (For --membind)
--membind=N: Strictly allocate memory on node N. (Fails if node N is full).

--preferred=N: Try node N first, fall back to other nodes if full.

--interleave=all: Spread memory evenly across all nodes.

5. Monitor NUMA Statistics

# Check overall NUMA hit/miss statistics
numastat

# Check per-node memory usage
numastat -m

# Check the memory distribution of a specific process
numastat -p <PID>

6. Pin a systemd Service Permanently
Create a drop-in file: /etc/systemd/system/myllm.service.d/numa.conf

[Service]
CPUAffinity=0-7
NUMAPolicy=bind
NUMAMask=0

# Reload systemd and restart the service
sudo systemctl daemon-reload
sudo systemctl restart myllm

# Verify the settings
systemctl show myllm -p CPUAffinity -p NUMAPolicy -p NUMAMask

Tube comments — I read every single one.
📺 youtube.com/@SysTelligence
