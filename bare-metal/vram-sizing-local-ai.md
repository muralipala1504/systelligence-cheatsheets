# VRAM Sizing for Local AI Cheatsheet
# Bare Metal Mastery Series — Part 22
# youtube.com/@SysTelligence

---

## 1. Measure Your Actual Free VRAM
```bash
# The total is not the free limit. The OS uses some.
nvidia-smi --query-gpu=name,memory.total,memory.used --format=csv

(On a 4GB card, you will likely see 400-600 MiB already used by the display/OS).

2. Check Model File Size vs Loaded Size

# Check the file size on disk
ollama list

# Check the actual loaded size in memory at your specific context
ollama ps

Rule: The file size is the floor. The loaded size includes the KV cache and working buffers, which grow with context.

3. Read the Process Output
Run ollama ps and look at two columns:

PROCESSOR: You want to see 100% GPU. If it says xx%/yy% CPU/GPU, it is spilling to system memory.

CONTEXT: The context length dictates the KV cache size. A larger context uses more memory.

4. The VRAM Formula

Weights + KV Cache + Compute Buffers ≤ Free VRAM

5. Quantization (Saving Memory)

# Check the quantization level of a model
ollama show <model_name>

Q4_K_M: ~4 bits per parameter (Good balance).

Q8_0: ~8 bits per parameter (Higher accuracy, double the RAM).

6. Test with a Custom Context Length

# Run a one-off prompt with a smaller context to fit in VRAM
ollama run <model_name> --verbose "Your prompt here"

(To set context via API, use the num_ctx parameter).

7. Managing the Load (Cold Start)

# The first run takes time to load into memory.
# If you run a second prompt quickly, it is much faster.

# To free memory immediately after you are done:
ollama stop <model_name>

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence
