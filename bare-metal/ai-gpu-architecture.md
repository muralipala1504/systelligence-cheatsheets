# Inside the AI GPU: Blackwell vs Rubin Cheatsheet
# Bare Metal Mastery Series — Part 20
# youtube.com/@SysTelligence

---

## 1. Identify the Hardware
```bash
# List all GPUs on the system
nvidia-smi -L

# Query the GPU name, compute capability, and driver version
nvidia-smi --query-gpu=name,compute_cap,driver_version --format=csv

2. Check the Host Link (PCIe)

# Check the current PCIe link generation and width
nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.width.current --format=csv

# Verify the physical link capabilities (LnkCap) vs current status (LnkSta)
sudo lspci -vv | rg 'LnkCap|LnkSta'

(Rule: Compare current against maximum. If the link is training at a lower generation or width than the slot supports, your GPU is starved.)

3. Check the GPU to GPU Link (NVLink)

# Show the topology matrix
nvidia-smi topo -m

# Check the status of NVLink connections
nvidia-smi nvlink --status

The Topology Matrix Legend
When you run nvidia-smi topo -m, you will see a grid. Here is how to read it:

X: The GPU itself.

NV#: Connected by that many NVLinks (This is what you want to see).

PIX: Connected via at most one PCIe bridge.

PXB: Connected via multiple PCIe bridges.

PHB: Connected through the PCIe Host Bridge.

NODE: Within one NUMA node.

SYS: Across NUMA nodes (Traffic has to cross the CPU, which is slow).

4. Deep Hardware Details & Power

# Full query of GPU hardware details
nvidia-smi -q

# Check power draw and power limits
nvidia-smi --query-gpu=power.draw,power.limit --format=csv

5. The Software Stack Alignment
If nvidia-smi fails or performance is low, verify the stack:

# Check running kernel
uname -r

# Check if the kernel development headers match the running kernel
rpm -q kernel-devel-$(uname -r)

# Check the Nvidia kernel module information
modinfo nvidia

# Check Secure Boot status (if on, modules must be signed)
mokutil --sb-state

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
