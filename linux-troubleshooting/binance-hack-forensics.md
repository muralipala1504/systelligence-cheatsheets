# The $6.3 Billion Binance Hack: Endpoint Forensics Guide
# Linux Troubleshooting Series — Episode 11
# youtube.com/@SysTelligence

---

## 1. The Attack Vector (Compromised Endpoint)
The breach did not start in a datacenter. It started on a DevOps engineer's laptop.
```bash
# Attackers target the workstation to steal active session tokens and keys.
# Always check for unauthorized background processes.
ps aux | grep -E 'curl|wget|python|ssh'
# Check for unexpected outbound connections
ss -tulpn

2. Lateral Movement (SSH Agent Abuse)
If your SSH keys are loaded in memory, an attacker with laptop access can use them to jump to production.

# List keys currently loaded in the SSH agent (attacker does this)
ssh-add -l

# Check auth logs on production for logins from the compromised laptop IP
sudo journalctl -u sshd | grep -i "Accepted publickey"

3. The Container Escape
The attacker used the stolen API token to hit an internal service container.

# If an API token is compromised, attackers bypass the UI and hit the API directly.
# Ensure internal APIs require Mutual TLS (mTLS), not just Bearer tokens.
curl -X POST https://internal-api.local/withdraw -H "Authorization: Bearer $STOLEN_TOKEN"

4. The EIP-7702 Exploit (The 50 Nonces)
The attacker flooded the Ethereum network with 50 pre-signed authorizations.

# EIP-7702 allowed the attacker to temporarily turn a regular wallet into a smart contract.
# They sent 50 transactions. Only 1 needed to match the wallet's current nonce.
# Result: $6.3 Billion swept in seconds.

5. The Forensic Trail (Investigating the Laptop)
If you suspect an endpoint compromise, preserve the logs immediately.

# 1. Auditd (Process Execution)
sudo ausearch -m EXECVE --start today | grep -E 'curl|wget|base64'

# 2. Bash History (Check for cleared history)
cat ~/.bash_history

# 3. Auth Logs (Unexpected local logins)
sudo journalctl -u systemd-logind --since "1 hour ago"

6. The Glass Terminal Defense (Prevention)
Never store production keys on your host OS.

Host: Windows or macOS (used only as a screen and keyboard).

Workstation: NAT'd VMs or WSL2 with NO direct internet exposure.

Secrets: Use a Vault or hardware key (YubiKey). Never store unencrypted SSH keys on the host filesystem.

Network: Force all admin traffic through a Bastion Host with MFA.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
