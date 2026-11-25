# Post-Quantum Cryptography (PQC) on RHEL 10 Cheatsheet
# RHEL 10 Mastery Series — Episode 11
# youtube.com/@SysTelligence

---

## 1. Prerequisites (Not installed by default)
```bash
# Install OpenSSL and the crypto policy tools
sudo dnf install -y openssl

# Verify the version (must be 3.5+)
openssl version
# Output: OpenSSL 3.5.9 (or newer)

# Verify the packages
rpm -q openssl openssl-libs crypto-policies

2. Check for PQC Algorithms (Default Policy)
Even on the DEFAULT policy, OpenSSL 3.5+ ships with the NIST Post-Quantum algorithms.

# Check Key Encapsulation Mechanisms (ML-KEM)
openssl list -kem-algorithms | grep -i ml

# Check Digital Signatures (ML-DSA, SLH-DSA)
openssl list -signature-algorithms | grep -i ml

3. The Crypto Policy Trap (The Gotcha)
The FUTURE policy enforces stricter cryptography (e.g., minimum RSA 3072). This breaks legacy mirrors.

# Check current policy
update-crypto-policies --show

# Switch to the FUTURE policy (Requires reboot or restart of services)
sudo update-crypto-policies --set FUTURE

# Try to update packages (This will fail if the mirror uses weak RSA)
sudo dnf makecache
# Error: SSL certificate problem: EE certificate key too weak

4. The Rollback (The Fix)
If dnf or SSH breaks, revert to DEFAULT.

# Revert to DEFAULT
sudo update-crypto-policies --set DEFAULT

# Verify it is applied
update-crypto-policies --is-applied
update-crypto-policies --check

# Test dnf again
sudo dnf makecache
# Output: Metadata cache created

5. Testing PQC Signatures (ML-DSA)

# Generate an ML-DSA-65 key
openssl genpkey -algorithm ML-DSA-65 -out mldsa65.key

# Sign a message
echo "pq test" > msg.txt
openssl pkeyutl -sign -rawin -inkey mldsa65.key -in msg.txt -out msg.sig

# Verify the signature
openssl pkeyutl -verify -rawin -inkey mldsa65.key -in msg.txt -sigfile msg.sig
# Output: Signature Verified Successfully

6. Testing the Handshake (Hybrid Groups)

# Generate a self-signed ML-DSA certificate
openssl req -x509 -newkey ML-DSA-65 -keyout srv.key -out srv.crt -days 1 -subj "/CN=localhost" -nodes

# Start a test server
openssl s_server -accept 4433 -cert srv.crt -key srv.key -www -quiet

# Connect using the hybrid PQC group
echo | openssl s_client -connect localhost:4433 -groups X25519MLKEM768 -brief
# Look for: Signature type: mldsa65, Negotiated TLS1.3 group: X25519MLKEM768

7. Checking SSH Support

# Check if SSH supports Post-Quantum key exchange
ssh -Q kex | grep -i mlkem
# Output: mlkem768x25519-sha256, mlkem768nistp256-sha256, mlkem1024nistp384-sha384

8. The SSH Policy Trap
If you set update-crypto-policies --set FUTURE, older SSH clients (like PuTTY < 0.83) will be locked out. Test with the latest PuTTY (0.85+) before rolling out fleet-wide.

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
