# eBPF and bpftrace Deep Dive Cheatsheet
# Linux Arsenal Series — Episode 8
# youtube.com/@SysTelligence

---

## 1. Install bpftrace
```bash
# Not installed by default on RHEL 10 or Alma 9
sudo dnf install -y bpftrace

# Verify version
bpftrace --version

2. Trace Process Execution (execve)
# Watch new processes start
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s\n", comm); }'

3. Trace File Opens (openat)

# Watch who reads the passwd file
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat /str(args->filename) == "/etc/passwd"/ { printf("%s opened passwd\n", comm); }'

4. Trace Signals (signal_generate)

# Watch signals being sent to processes
sudo bpftrace -e 'tracepoint:signal:signal_generate { printf("Signal %d sent to %s by %s\n", args->sig, args->comm, comm); }'

5. Trace Network Connections (tcp_connect)

# Watch outgoing TCP connections
sudo bpftrace -e 'kprobe:tcp_connect { printf("%s connected to port %d\n", comm, ((struct sock *)arg0)->__sk_common.skc_dport); }'

6. Production Safety Rules
Test on Staging First. Never run a new script directly on a production server.

Use Filters. Do not trace all events. Filter by process name, file, or port.

Limit Execution Time. Use the timeout command (e.g., timeout 10s sudo bpftrace ...) to prevent buffer floods.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
