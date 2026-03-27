```markdown
<div align="center">

<img src="https://img.shields.io/badge/Epsilon-IDE%20Engine-4f46e5?style=for-the-badge&logoColor=white"/>

# Epsilon IDE Engine
**Potato PC Edition — v1.0**

A local AI coding assistant that runs on weak hardware.  
No internet. No API keys. No cloud. Everything runs on your machine.

![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-WSL2%20Ubuntu-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![RAM](https://img.shields.io/badge/RAM-80%20MB%20Python%20footprint-teal?style=flat-square)
![Part of SEAL](https://img.shields.io/badge/Part%20of-SEAL%20Project-purple?style=flat-square)

</div>

---

## What is this?

Epsilon IDE Engine is an AI coding assistant — similar to GitHub Copilot — that runs
completely offline on your own hardware. You type a function name, it writes the body.
You describe what you want, it generates the code. You paste a broken function, it fixes it.

The difference from Copilot: **it runs on your machine**. No subscription.
No data sent anywhere. Works without internet. Built to fit inside 2 GB of RAM
and 2 GB of GPU memory — hardware that most AI tools refuse to support.

This is **v1 — the working prototype**. The engine generates code, searches your project
files for context, validates syntax, and communicates through a clean JSON interface
that a VS Code extension (coming in v2) will connect to.

---

## How it works — the simple version

When you send a prompt, four agents handle it in sequence:

```
Your prompt
     │
     ▼
 ROUTER ──── figures out what kind of request this is (zero AI)
     │
     ▼
 RECALL ──── searches your project files for relevant code (zero AI)
     │
     ▼
 CODER ───── calls the AI model exactly once (one AI call)
     │
     ▼
 CRITIC ──── checks the generated code for syntax errors (zero AI)
     │
     ▼
JSON response
```

Only ONE agent ever calls the AI model. The other three use classical algorithms
that run in under 5 milliseconds. This is why the whole system fits in 80 MB of RAM.

---

## The five technologies that make it possible

| Technology | Problem it solves | Memory saved |
|---|---|---|
| **BitNet 1.58-bit** | Model weights stored as -1, 0, +1 instead of 16-bit floats | 988 MB → 120 MB |
| **bitnet.cpp** | C++ inference engine — no PyTorch needed | 800 MB → 30 MB |
| **INT8 KV Cache** | Model's short-term memory stored in 1 byte not 2 | 440 MB → 80 MB |
| **Sparse attention** | Model looks at 64 relevant tokens instead of all 512 | 8× memory reduction |
| **TF-IDF search** | Code search without a neural embedding model | 500 MB → 20 MB |

**Total Python footprint: ~80 MB.** The model itself uses ~1.2 GB in RAM (CPU mode).

---

## Hardware requirements

| | Minimum | What this project was built on |
|---|---|---|
| **OS** | Ubuntu 20.04+ or WSL2 | WSL2 Ubuntu 22.04 on Windows 10 |
| **CPU** | Any x86-64 | Intel i7-11850H |
| **RAM** | 2 GB | 16 GB (engine uses ~1.3 GB total) |
| **GPU** | Not required for v1 | NVIDIA RTX A2000 (used in v2) |
| **Disk** | 3 GB free | SSD strongly recommended |
| **Python** | 3.11 | 3.11.x |

> **Windows users:** You need WSL2 (Windows Subsystem for Linux).
> If you have Docker Desktop installed, WSL2 is already on your machine.
> Open Ubuntu from the Start menu and follow the Linux instructions below.

---

## Installation — step by step

Follow every step in order. Do not skip any. Each step builds on the previous one.

### Step 1 — Install system dependencies

Open your Ubuntu terminal and run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git cmake build-essential clang lld \
    python3.11 python3.11-venv python3-pip pybind11-dev
```

**What this installs:**
- `build-essential` + `clang` — C++ compiler for building bitnet.cpp
- `cmake` — build system that compiles the C++ code
- `python3.11` — the Python version this project uses
- `pybind11-dev` — lets C++ and Python talk to each other

Verify everything installed:

```bash
clang --version
cmake --version
python3.11 --version
```

All three should print version numbers without errors.

---

### Step 2 — Clone this repository

```bash
git clone https://github.com/YOUR_USERNAME/epsilon-ide.git
cd epsilon-ide
```

> Replace `YOUR_USERNAME` with your actual GitHub username.

---

### Step 3 — Create the Python virtual environment

A virtual environment keeps this project's packages separate from your system Python.
Think of it as a clean room just for this project.

```bash
python3.11 -m venv epsilon-env
source epsilon-env/bin/activate
```

Your terminal prompt should now start with `(epsilon-env)`. You need to run
`source epsilon-env/bin/activate` every time you open a new terminal to work on this project.

Install the required Python packages:

```bash
pip install --upgrade pip
pip install tinygrad sqlite-vec tree-sitter tree-sitter-python
pip install requests psutil pyyaml huggingface-hub
```

> Notice what is **not** here: no `torch`, no `transformers`, no `numpy` bloat.
> We deliberately avoid heavy ML libraries to keep the memory footprint tiny.

---

### Step 4 — Build bitnet.cpp

bitnet.cpp is Microsoft's open-source C++ inference engine for BitNet models.
It runs the AI model as a local HTTP server on port 8088.
Your Python code never loads the model directly — it just sends HTTP requests to this server.

```bash
# Clone bitnet.cpp next to the epsilon-ide folder
cd ~
git clone --recursive https://github.com/microsoft/BitNet.git
cd BitNet
```

Apply a one-line fix for a known compile error in the source:

```bash
sed -i 's/        int8_t \* y_col = y + col \* by;/        const int8_t * y_col = y + col * by;/' \
    src/ggml-bitnet-mad.cpp
```

Download the model (this downloads ~1.19 GB — takes 1-2 minutes):

```bash
python3 -c "
from huggingface_hub import hf_hub_download
import os
os.makedirs('models/BitNet-b1.58-2B-4T', exist_ok=True)
hf_hub_download(
    repo_id='microsoft/BitNet-b1.58-2B-4T-gguf',
    filename='ggml-model-i2_s.gguf',
    local_dir='models/BitNet-b1.58-2B-4T',
)
print('Model downloaded successfully.')
"
```

Build the inference engine (takes 3-5 minutes):

```bash
python3 setup_env.py -md models/BitNet-b1.58-2B-4T -q i2_s
```

You should see `INFO:root:Build successful` at the end.

Verify the binary was created:

```bash
./build/bin/llama-server --version
```

This should print a version number. If it does, bitnet.cpp is ready.

---

### Step 5 — Configure the engine

Go back to your epsilon-ide folder and create your config file:

```bash
cd ~/epsilon-ide
cp config.example.yaml config.yaml
```

Open `config.yaml` and update the paths:

```bash
nano config.yaml
```

Change `YOUR_USERNAME` to your actual Linux username (run `whoami` if unsure):

```yaml
model_path:    /home/YOUR_USERNAME/BitNet/models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf
context_len:   512
cpu_threads:   4
db_path:       /home/YOUR_USERNAME/epsilon-ide/clara.db
project_dir:   /home/YOUR_USERNAME/epsilon-ide
crawl_on_start: true
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

---

### Step 6 — Verify the installation

Run this final check to make sure everything is connected:

```bash
cd ~/epsilon-ide
source epsilon-env/bin/activate

python3 -c "
import sys
sys.path.insert(0, '.')
import tinygrad
import sqlite_vec
import tree_sitter
print('All Python packages OK')
"
```

Then start the server manually to confirm the model loads:

```bash
~/BitNet/build/bin/llama-server \
  -m ~/BitNet/models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf \
  -c 512 -t 4 --port 8088 -ngl 0 --log-disable &

sleep 15
curl http://localhost:8088/health
```

You should see `{"status":"ok"}`. Kill the server:

```bash
pkill llama-server
```

---

## Running the engine

### Quick test — one prompt, one response

```bash
cd ~/epsilon-ide
source epsilon-env/bin/activate

echo '{"prompt": "write a bubble sort function"}' \
  | python3 backend/main.py 2>/dev/null
```

Expected output:

```json
{
  "ok": true,
  "result": "arr):\n    n = len(arr)\n    for i in range(n):\n        ...",
  "task_type": "CODE_GEN",
  "valid_syntax": true,
  "retried": false
}
```

### Interactive session

Start the engine and keep it running, then send prompts from a second terminal:

**Terminal 1 — start the engine:**

```bash
cd ~/epsilon-ide
source epsilon-env/bin/activate
python3 backend/main.py
```

The engine will boot, index your project files, and print:
```
ENGINE READY — waiting for requests
```

**Terminal 2 — send prompts:**

```bash
# Code generation
echo '{"prompt": "write a fibonacci function"}' | nc localhost 8088

# Or pipe directly to the running engine process
echo '{"prompt": "explain what a decorator does"}' \
  >> /proc/$(pgrep -f "python3 backend/main.py")/fd/0
```

> For the cleanest experience, use the pipe method from the Quick Test above.
> Each `echo | python3 backend/main.py` starts a fresh engine session.

---

## Prompt types and examples

The engine automatically detects what kind of request you are making:

```bash
# Generate a function
echo '{"prompt": "write a quicksort function in Python"}' | python3 backend/main.py 2>/dev/null

# Describe a program
echo '{"prompt": "write a script that reads a CSV and prints the average of each column"}' | python3 backend/main.py 2>/dev/null

# Fix a bug
echo '{"prompt": "fix the bug: def multiply(a, b) return a * b"}' | python3 backend/main.py 2>/dev/null

# Explain code
echo '{"prompt": "explain what a context manager does in Python"}' | python3 backend/main.py 2>/dev/null

# Refactor
echo '{"prompt": "simplify this: if x == True: return True else: return False"}' | python3 backend/main.py 2>/dev/null
```

---

## Monitoring performance

While the engine is running, open a second terminal to watch memory and CPU:

```bash
# Real-time GPU memory (if NVIDIA GPU present)
watch -n 1 nvidia-smi

# CPU and RAM usage
htop

# See exactly how much RAM the model server is using
PID=$(pgrep llama-server)
cat /proc/$PID/status | grep VmRSS
# VmRSS should show ~1200-1400 MB — that is the model in RAM
```

---

## Project structure

```
epsilon-ide/
│
├── backend/
│   ├── tiers/
│   │   └── bitnet_model.py          # AI model wrapper
│   │                                # Starts llama-server, polls /health,
│   │                                # sends HTTP POST, returns generated text
│   │
│   ├── inference/
│   │   └── tinygrad_kv.py           # INT8 KV cache
│   │                                # Stores attention vectors in numpy arrays
│   │                                # Sparse attention: 64 of 512 tokens
│   │
│   ├── clara/
│   │   └── potato_oracle.py         # Code search engine
│   │                                # TF-IDF + sqlite-vec
│   │                                # Indexes project files, finds relevant code
│   │
│   ├── picoclaw/
│   │   └── potato_orchestrator.py   # The four-agent pipeline
│   │                                # ROUTER → RECALL → CODER → CRITIC
│   │
│   ├── aether/
│   │   └── aether_link.py           # stdin/stdout JSON bridge
│   │                                # Reads requests, writes responses
│   │
│   └── main.py                      # Engine entry point
│                                    # Boots all components, starts event loop
│
├── scripts/
│   └── monitor.sh                   # Real-time monitoring script
│
├── config.example.yaml              # Configuration template — copy to config.yaml
└── README.md
```

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'tinygrad'`**
```bash
source ~/epsilon-ide/epsilon-env/bin/activate
# Your prompt should show (epsilon-env) — if not, the venv is not active
```

**`ConnectionRefusedError` when starting the engine**
```bash
# The model is still loading — wait 15-20 seconds after starting
# Check if llama-server is running:
pgrep llama-server
```

**`setup_env.py` fails during build**
```bash
# Check the build log for the exact error
cat ~/BitNet/logs/compile.log | tail -30
```

**Engine starts but generates empty responses**
```bash
# Check the model path in config.yaml is correct
ls -la $(grep model_path config.yaml | awk '{print $2}')
# File should exist and be ~1.19 GB
```

**Fan spins up during inference**
This is normal. The CPU runs at 100% during the ~30 seconds it takes to generate
a response in CPU mode. The fan is doing its job. GPU mode (v2) reduces this significantly.

---

## Roadmap

| Version | Feature | Status |
|---|---|---|
| v1.0 | Working CPU inference engine | **Done — this repo** |
| v2.0 | GPU inference (10-20× faster) | Planned |
| v2.0 | Qwen2.5-Coder model (better code quality) | Planned |
| v2.0 | File writing — engine creates files on disk | Planned |
| v2.0 | Conversation memory | Planned |
| v2.0 | VS Code extension with ghost text | Planned |
| v3.0 | Multi-model routing (1.5B + 7B + 33B) | Future |

---

## Part of the SEAL Project

Epsilon IDE is the coding assistant component of **SEAL** — a personal AI runtime
built to run entirely on local hardware. SEAL runs on the **Aether runtime framework**.

---

## Built on these open source projects

- **[Microsoft BitNet](https://github.com/microsoft/BitNet)** — 1.58-bit quantisation and C++ inference engine
- **[tinygrad](https://github.com/tinygrad/tinygrad)** — Lightweight tensor operations, no PyTorch
- **[sqlite-vec](https://github.com/asg017/sqlite-vec)** — Vector similarity search in SQLite
- **[Tree-sitter](https://github.com/tree-sitter/tree-sitter)** — Fast code parsing for syntax validation
- **[HuggingFace Hub](https://huggingface.co/)** — Model hosting

---

## License

MIT — use it, fork it, build on it.

---

<div align="center">
<b>Built for people who refuse to let hardware limits stop them.</b>
</div>
```

---

After pasting this into `README.md` in VS Code, update **one thing** — find this line:

```
git clone https://github.com/YOUR_USERNAME/epsilon-ide.git
```

And replace `YOUR_USERNAME` with your actual GitHub username.