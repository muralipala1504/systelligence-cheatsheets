# The $30,000 AI Server Build

Enterprise AI server parts selection — CPU, memory, PCIe, GPU, storage, power, and cooling. The choices that decide whether your GPUs run at full speed or sit idle.

📺 Playlist: [Bare Metal Mastery Series](https://youtube.com/@SysTelligence)

---

## 1. The Budget

| Item | Approx Cost |
|------|-------------|
| CPU | $3,000 |
| Motherboard | $1,000 |
| RAM | $2,000 |
| GPUs (4x) | $16,000 |
| NVMe drives | $2,000 |
| Power | $1,000 |
| Chassis | $1,000 |
| **Subtotal** | **$26,000** |
| Networking + spares | $4,000 |
| **Total** | **$30,000** |

> Prices are approximate. Always get a real vendor quote.

---

## 2. CPU Selection — PCIe Lanes First

| Spec | AMD EPYC | Intel Xeon |
|------|----------|------------|
| PCIe lanes | 128 | 80 |
| Memory channels | 12 | 8 |

**Rule:** You need PCIe lanes to feed the GPUs. Lanes beat core count in an AI build.

---

## 3. Memory Channels

- Populate **all** memory channels. Both sockets.
- Missing a stick cuts memory bandwidth in half.
- Match sticks per channel across both CPUs.

---

## 4. PCIe Slot Layout

| Slot | Use |
|------|-----|
| PCIe Gen 4 x16 (x4) | GPUs |
| PCIe Gen 4 x4 (x2) | NVMe |

**Warning:** Installing an NVMe drive can drop a GPU slot to x8. Check the board lane map before you populate.

---

## 5. GPU Selection

| Card | Memory | Class |
|------|--------|-------|
| RTX 4090 | 24GB | Consumer |
| L40S | 48GB | Enterprise, NVLink support |

**Rule:** Consumer cards are fine for single-node learning. Enterprise cards with NVLink are for real inference clusters.

---

## 6. Storage Layout

- Use **local NVMe Gen 4 x4** for OS and model cache.
- A SAN or remote array will bottleneck the GPU on every model load.
- RAID controller is for bulk data, cold storage, logs — not model drives.
- Keep the fast path local. Keep the slow path on the controller.

---

## 7. Power Delivery

- Example GPU draw: 350W each.
- Example PSU: 1600W Titanium.
- **Rule:** Sum total draw, then add 20% headroom.
- One PSU failure can take down the whole node. Size for redundancy.

---

## 8. Cooling and Airflow

- Front to back airflow. Match fan direction to chassis design.
- Air cooling is fine for single nodes.
- Liquid cooling is for dense racks.

---

## 9. Bare Metal OS

```bash
cat /etc/redhat-release
# Red Hat Enterprise Linux release 10.2

lscpu | grep NUMA
# NUMA node(s): 2

10. Build Verification

# Confirm all GPUs are seen
nvidia-smi

# Confirm PCIe link speed and width
lspci -vv | grep LnkSta

# Confirm NUMA topology
numactl --hardware

Verify the hardware before you run the AI workload.

11. Reference Card
Layer	Minimum
CPU	128 PCIe lanes
Memory	12 channels
GPU	Enterprise, NVLink
Storage	Local NVMe
Power	20 percent headroom

Disclaimer
All prices are approximate and move with the market. AMD, Intel, Nvidia, and Red Hat did not sponsor this content. These are practical picks from enterprise experience. Always get a real vendor quote before you buy.


