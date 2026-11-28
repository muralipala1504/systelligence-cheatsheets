# Memory Leaks and Ghost Processes — bpftrace Guide

Finding the invisible memory problem — short lived processes, slow leaks, zombies, and OOM victims — using kernel tracing on RHEL 10.2.

📺 Playlist: [Linux Troubleshooting Series](https://youtube.com/@SysTelligence)

---

## 1. The 3 AM Alert

```bash
free -h
ps aux --sort=-rss | head -5
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
dmesg -T | rg -i 'out of memory|killed process'# Memory Leaks and Ghost Processes — bpftrace Guide

Finding the invisible memory problem — short lived processes, slow leaks, zombies, and OOM victims — using kernel tracing on RHEL 10.2.

📺 Playlist: [Linux Troubleshooting Series](https://youtube.com/@SysTelligence)

---

## 1. The 3 AM Alert

```bash
free -h
ps aux --sort=-rss | head -5
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
dmesg -T | rg -i 'out of memory|killed process'

Rule: No single process is the villain. Memory is spread across many, or it is gone in seconds.

2. Why Snapshot Tools Miss It
Failure Pattern	Why top Misses It
Short lived process	Starts and exits between samples
Slow leak	Rise is small, hidden in normal noise
Proof — sample the same process every two seconds:

for i in 1 2 3 4 5; do echo "t=$((i*2))s rss_kb=$(ps -o rss= -p $PID)"; sleep 2; done

Output: 14040, 40752, 73528, 106304, 139080 — about 16 MiB per second.

3. Check Before You Trace

bpftrace --version
# v0.26.x

cat /sys/kernel/security/lockdown
# [none] integrity confidentiality

ls -l /sys/kernel/btf/vmlinux
# -r--r--r--. 1 root root ... /sys/kernel/btf/vmlinux

Warning: Tracing needs root, adds overhead, and a stricter lockdown mode can block probes.

4. The Toolchain

sudo dnf install -y bpftrace

rpm -q bpftrace bcc-tools

ls /usr/share/bcc/tools/ | rg -i 'memleak|execsnoop|exitsnoop|oomkill'
# execsnoop
# exitsnoop
# memleak
# oomkillsudo dnf install -y bpftrace

rpm -q bpftrace bcc-tools

ls /usr/share/bcc/tools/ | rg -i 'memleak|execsnoop|exitsnoop|oomkill'
# execsnoop
# exitsnoop
# memleak
# oomkill

Probe discovery — confirm each exists:

sudo bpftrace -l 'tracepoint:syscalls:sys_enter_execve'
sudo bpftrace -l 'tracepoint:syscalls:sys_enter_mmap'
sudo bpftrace -l 'tracepoint:syscalls:sys_enter_brk'
sudo bpftrace -l 'tracepoint:syscalls:sys_enter_munmap'
sudo bpftrace -l 'tracepoint:sched:sched_process_exit'

5. The Zombie Hunter
Find zombies:

ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'

Trace the parent that failed to collect:

sudo bpftrace -e 'tracepoint:sched:sched_process_exit /comm == "python3"/ { printf("exit: pid=%d parent=%s ppid=%d\n", pid, curtask->real_parent->comm, curtask->real_parent->tgid); }' -c "python3 /tmp/zomb.py"

Rule: A zombie holds a PID, almost no memory. You cannot kill it. Fix the parent.

6. The OOM Trace

sudo bpftrace -l 'tracepoint:oom:*'

Catch the victim at the moment of choice:

sudo timeout 25 bpftrace -e 'tracepoint:oom:mark_victim { printf("OOM victim: %s pid=%d anon_rss=%lu score_adj=%d\n", str(args->comm), args->pid, args->anon_rss, args->oom_score_adj); }'sudo timeout 25 bpftrace -e 'tracepoint:oom:mark_victim { printf("OOM victim: %s pid=%d anon_rss=%lu score_adj=%d\n", str(args->comm), args->pid, args->anon_rss, args->oom_score_adj); }'

Cross-check from the kernel ring buffer:

dmesg -T | rg -i 'oom|killed process'

Rule: mark_victim fires at the moment the kernel picks its victim. A cgroup kill leaves the host alive.

7. The Memory Leak — Who Asks

sudo bpftrace -e 'tracepoint:syscalls:sys_enter_mmap /comm == "python3"/ { @mmap_bytes = sum(args->len); }' -c "python3 /tmp/leak.py"

Rule: Find the growing process by RSS first. Then sum memory requests in a map. Never print every call.

Warning: A glibc heap leak can grow without a syscall. RSS growth is the first signal.

8. The Memory Leak — Who Fails to Release

sudo bpftrace -e 'tracepoint:syscalls:sys_enter_munmap /comm == "python3"/ { @munmap_bytes = sum(args->len); }' -c "python3 /tmp/leak.py"

Script	@mmap_bytes	@munmap_bytes
leak.py (keeps)	~288 MB	~19 KB
free.py (releases)	~288 MB	~268 MB
Rule: A large allocation never freed never shows up in munmap.

Warning: A missing munmap is not proof of a leak. Exit frees memory without it.

9. The Process Spawn Trap

sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args->filename)); }' -c "sh /tmp/job.sh"

Output:

bpftrace -> /bin/sh
sh -> /bin/ls
sh -> /bin/date

Rule: comm is the caller. filename is what it launched. That catches the two second cron job.

Ready-made tool: execsnoop from bcc-tools.

10. The Audit Trail — Count, Bound, Keep
Layer	Technique
Count	@bytes[comm] = sum(args->len)
Bound	interval:s:5 { exit(); } or timeout
Keep	-o /tmp/trace.log with rotation, or audit framework


Rule: Never print every event to the console on a busy server.

11. Production Safety Checklist
Test on staging first.

Filter by comm or pid. Never trace everything.

Aggregate in maps. Do not print per event.

Bound the run with timeout or interval:s:N { exit(); }.

Avoid high frequency probes on busy systems.

Know your lockdown mode: cat /sys/kernel/security/lockdown.

Scale reality: A tiny python3 startup made 26 mmap calls and 2 munmap calls. A busy server makes millions.

12. Reference Card — Five One-Liners

# 1. Catch process spawns
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args->filename)); }'

# 2. Sum memory requests
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_mmap /comm == "NAME"/ { @mmap_bytes = sum(args->len); }'

# 3. Sum memory released
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_munmap /comm == "NAME"/ { @munmap_bytes = sum(args->len); }'

# 4. Catch OOM victims
sudo bpftrace -e 'tracepoint:oom:mark_victim { printf("%s pid=%d\n", str(args->comm), args->pid); }'

# 5. Catch parents of exiting processes
sudo bpftrace -e 'tracepoint:sched:sched_process_exit /comm == "NAME"/ { printf("exit: pid=%d parent=%s ppid=%d\n", pid, curtask->real_parent->comm, curtask->real_parent->tgid); }'

Always add a timeout, a filter, and a map. Replace NAME with your process.

Verification
Tools, probes, and arguments verified on RHEL 10.2. Step 1 also matched on AlmaLinux 10.2. Always confirm probe names on your own kernel before you trust a one-liner.


