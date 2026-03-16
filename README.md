# Epsilon IDE Engine 🦭💥

**Status**: Barely holding together.

Look, this isn't another wrapper around OpenAI API painted to look like VS Code. I built this because I am always hardware-constrained, and I wanted to help my friends out who also run on absolute peasant hardware (16GB RAM + 8GB VRAM). I wanted us to be able to run a real multi-tier LLM architecture (70B + 7B + 1.1B) without melting our motherboards.

This is part of my larger goal to create **SEAL**. It runs on my **Aether runtime framework**, built from the ground up by *e*.

To make this work, I had to spend the last week writing raw C++ CUDA semaphores, Windows IoCompletionPorts, and POSIX shared memory pointers just to bypass Python's dogshit overhead. It took me multiple attempts to get the HugePages working without blue-screening my rig. Fuck Windows memory management.

If you don't have a C++ compiler and the CUDA Toolkit, don't even try to run this. It will just crash.

## The Architecture (Why this was a nightmare to build)

We are running "The Swap-Deck" tier system:
1. **The Architect (DeepSeek 33B/70B)**: Sits on the SSD. Too fat for VRAM. We DMA stream its layers directly into RAM using async NVMe `IoCompletionPorts` (bypassing the CPU entirely because Python `read()` is too slow).
2. **The Logic-Gate (Qwen2.5-Coder 7B)**: Lives in VRAM. Handles the actual ghost text generation.
3. **The Foreman (TinyLlama 1.1B)**: Also in VRAM. Routes tasks.

To make them not kill each other over VRAM access, I had to write:
- **`vram_guard.cpp`**: Real C++ atomic semaphores. Python `asyncio.Lock` does literally nothing to the GPU hardware driver. Had to learn this the hard way after 40 CUDA OOM crashes.
- **`extreme_hardware.cpp`**: Bypasses the OS 4KB page allocator using Windows `SEC_LARGE_PAGES`. If the translation lookaside buffer (TLB) misses during an AST handoff, the 7B model drops frames.
- **0-Copy Token Pump**: FastAPI was too bloated to stream the tokens to the frontend, so there's a raw C++ Windows Named Pipe (`\\.\pipe\sealmega_tokens`) screaming the tokens directly into the VS Code memory space. God help you if the pipe breaks.

## Features

- **Real Filesystem Access**: Actually reads and writes your local files. Not sandboxed. Don't let the 70B model write `rm -rf`.
- **Perplexity Rollback**: The system uses Cauchy-Schwarz pruning to speed up inference, but sometimes the model goes blind and hallucinates. I wrote a Shanon entropy safety net. If logit entropy spikes 1.8x, the layer recomputes. 
- **sqlite-vec Context Oracle**: Ripped out ChromaDB because it ate 2GB of RAM just sitting there. Replaced with bare `sqlite3` using `sqlite-vec` to index Python AST nodes. Hard-capped at 512MB RAM.

## How to run (If you hate yourself)

1. You need `pybind11` and MSVC / GCC.
2. Compile the C++ extensions in `backend/core/`.
3. Give your user account `SE_LOCK_MEMORY_NAME` privileges in Windows Group Policy or the HugePages allocation will fail and fallback to slow memory.
4. `pip install -r backend/requirements.txt`
5. `python backend/main.py`
6. Open `http://localhost:8742`

If it crashes on boot, your NVMe drive probably doesn't support unbuffered DMA overlapped I/O (`FILE_FLAG_NO_BUFFERING`). Buy a better SSD.

## License
MIT. Do whatever you want, but don't blame me when the CUDA kernel panics and takes your display driver down with it.
