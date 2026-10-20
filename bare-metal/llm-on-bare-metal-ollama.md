# LLM on Bare Metal - Ollama Cheatsheet
# Bare Metal Mastery Series — Part 16
# youtube.com/@SysTelligence

---

## Install & Run
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Check status
systemctl status ollama

# Pull a model
ollama pull llama3

# Run interactively
ollama run llama3

API & Ports

# Default port is 11434
curl http://localhost:11434

# Generate text via API
curl http://localhost:11434/api/generate -d '{
  "model": "llama3",
  "prompt": "Explain SELinux in one sentence.",
  "stream": false
}'

Monitoring VRAM

# Visual dashboard
btop

# NVIDIA specific monitoring
nvidia-smi

Production Systemd Override

# Create the override directory
mkdir -p /etc/systemd/system/ollama.service.d

# Create the override file
vi /etc/systemd/system/ollama.service.d/override.conf

Add these lines:

[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MODELS=/mnt/nvme/ollama-models"

Apply changes:

systemctl daemon-reload
systemctl restart ollama

Security Rules
NEVER expose port 11434 directly to the public internet.

Bind to localhost or an internal network.

Use nftables to block external access to 11434.

Put a reverse proxy (Nginx/Apache) with authentication in front if external access is needed.

Questions? Drop them in the YouTube comments — I reed every single one.
📺 youtube.com/@SysTelligence

