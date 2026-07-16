---
name: wsl-ml-setup
description: Diagnose and fix environment issues when running ML inference projects (vLLM, nano-vllm, etc.) on Windows+WSL. Covers Python version selection, CUDA version matching, dependency ordering, flash-attn compilation, pip mirror configuration, and common pitfalls.
---

# WSL ML Inference Environment Setup

Use this skill when a user on Windows+WSL can't run an ML inference project (import errors, CUDA mismatches, flash-attn build failures, etc.). The skill walks through systematic diagnosis and repair.

## Quick Diagnostic Flow

```
1. Identify the platform → Windows/WSL/Linux
2. Check Python version → must be 3.10-3.12 (NOT 3.13+)
3. Check CUDA version (nvidia-smi) → must match torch CUDA
4. Check triton availability → Linux/WSL only, not Windows native
5. Check flash-attn → needs torch pre-installed, right CUDA ABI
6. Check build tools → python3.12-dev for triton JIT compilation
```

## Rule 1: Never use Python 3.13+ (as of 2026-07)

**Problem**: flash-attn has no pre-built wheels for Python 3.13/3.14. Many ML packages lag behind.

**Detection**: `python --version` shows 3.13 or 3.14

**Fix**:
```bash
# On Ubuntu 24.04+ (only ships Python 3.13+)
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install python3.12 python3.12-venv python3.12-dev -y
python3.12 -m venv venv312
source venv312/bin/activate
```

**Why**: deadsnakes PPA provides older Python versions on new Ubuntu. Always install `python3.x-dev` too — triton needs `Python.h` at runtime for JIT compilation.

## Rule 2: CUDA version must match between torch and system

**Problem**: `RuntimeError: The detected CUDA version (12.4) mismatches the version that was used to compile PyTorch (13.0)`

**Detection**:
```bash
nvidia-smi | grep "CUDA Version"
python -c "import torch; print(torch.version.cuda)"
```

**Fix**: Uninstall mismatched torch, install the correct CUDA variant:
```bash
pip uninstall torch triton -y
# Check nvidia-smi for your CUDA version, then pick matching index:
pip install torch --index-url https://download.pytorch.org/whl/cu124   # for CUDA 12.4
# pip install torch --index-url https://download.pytorch.org/whl/cu121 # for CUDA 12.1
```

**Why**: PyTorch wheels are compiled against specific CUDA versions. The minor version matters (12.1 vs 12.4 vs 13.0 are all different ABIs). `nvidia-smi` shows the MAX supported CUDA version; the driver is backward-compatible, so you can install a torch wheel matching or lower than the driver version.

## Rule 3: Dependency installation ORDER matters

**Problem**: flash-attn build fails with `ModuleNotFoundError: No module named 'torch'`

**Correct order**:
```bash
# 1. torch FIRST (flash-attn setup.py imports torch to detect version)
pip install torch --index-url https://download.pytorch.org/whl/cu124

# 2. triton (comes with torch but may need explicit install)
pip install triton

# 3. Other pure-Python deps
pip install transformers xxhash psutil

# 4. flash-attn LAST (needs torch present to compile)
pip install flash-attn --no-build-isolation
```

**Why**: flash-attn's `setup.py` imports torch at build time to detect CUDA version and torch ABI. Without `--no-build-isolation`, pip builds in a clean venv without torch, causing `ModuleNotFoundError`.

## Rule 4: flash-attn ABI matching is fragile — prefer pre-built wheels

**Problem**: `undefined symbol: _ZN3c105ErrorC2E...` (C++ ABI mismatch)

**Detection**: Import succeeds but crashes at runtime with symbol errors.

**Fix**: Use a pre-built wheel matching your exact torch version:
```bash
# Check torch version
python -c "import torch; print(torch.__version__)"  # e.g. 2.6.0+cu124

# Download matching wheel from github releases:
# https://github.com/Dao-AILab/flash-attention/releases
# Naming: flash_attn-{version}+cu{CUDA_VER}torch{TORCH_VER}cxx11abi{TRUE/FALSE}-cp{PYTHON_VER}-cp{PYTHON_VER}-linux_x86_64.whl

# Example for torch 2.6.0+cu124, Python 3.12, cxx11abi=TRUE:
pip install https://github.com/Dao-AILab/flash-attention/releases/download/v2.7.4.post1/flash_attn-2.7.4.post1+cu124torch2.6cxx11abiTRUE-cp312-cp312-linux_x86_64.whl
```

**Why**: flash-attn's C++ extension uses the same C++ ABI as torch. If you built torch from source or got it from a different channel, the ABI flag (OLD vs NEW/CXX11) may differ from what flash-attn's build detects. Pre-built wheels are tested combinations.

## Rule 5: Windows native Python CANNOT run triton/flash-attn

**Problem**: `ModuleNotFoundError: No module named 'triton'` when `pip install triton` fails

**Detection**: Running on Windows (not WSL)

**Fix**: Use WSL. The project files can stay on Windows filesystem:
```bash
# All project files accessible at /mnt/c/Users/...
cd /mnt/c/Users/16874/AIInfraGuide
source venv312/bin/activate
python example.py
```

**Why**: Triton and flash-attn have Linux-only CUDA kernels. They compile `.c` files at runtime. Windows has no GCC, no CUDA runtime headers, and Triton's NVidia driver backend is Linux-only.

## Rule 6: pip "externally-managed-environment" error

**Problem**: `error: externally-managed-environment` (PEP 668)

**Fix**: Always use venv:
```bash
python3.12 -m venv venv312
source venv312/bin/activate
# now pip install works
```

**Why**: Modern Ubuntu/Debian mark the system Python as "externally managed" to prevent pip from breaking system packages.

## Rule 7: Missing Python.h / python3.x-dev

**Problem**: `fatal error: Python.h: No such file or directory` during triton JIT compilation at runtime

**Detection**: The error appears NOT at pip install time, but when the model actually runs (warmup phase compiles triton kernels).

**Fix**:
```bash
sudo apt install python3.12-dev -y
```

**Why**: Triton compiles CUDA kernels at runtime via GCC. It needs Python C headers. `python3.12-venv` includes the venv module but NOT the dev headers.

## Rule 8: China mainland — use mirrors

**Problem**: Slow pip downloads from PyPI

**Fix**:
```bash
# Tsinghua mirror (fastest in China)
pip install <package> -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn

# For torch specifically, PyTorch official index may be needed for CUDA wheels
# But the standard Tsinghua mirror also mirrors PyTorch wheels
pip install torch -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn

# Aliyun mirror (backup)
pip install <package> -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```

## Rule 9: Proxy for WSL

**Problem**: WSL can't reach external network, but Windows host has proxy

**Fix**:
```bash
# Find Windows host IP (usually in /etc/resolv.conf or via ip route)
export http_proxy=http://$(grep nameserver /etc/resolv.conf | awk '{print $2}'):7897
export https_proxy=$http_proxy

# Or use the actual host IP
export http_proxy=http://192.168.1.34:7897
export https_proxy=$http_proxy
```

## Rule 10: Model path between Windows and WSL

**Problem**: `~/huggingface/model` in WSL points to Linux home, not Windows home

**Fix**: Either use absolute `/mnt/c/...` paths, or create symlinks:
```bash
# Option 1: direct path
# In Python: "/mnt/c/Users/16874/AIInfraGuide/Qwen3-0.6B"

# Option 2: symlink
ln -s /mnt/c/Users/16874/huggingface ~/huggingface
```

## Complete Setup Script (for reference)

```bash
# === Run in WSL ===
# 1. Install Python 3.12 (if needed)
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install python3.12 python3.12-venv python3.12-dev -y

# 2. Create venv
python3.12 -m venv venv312
source venv312/bin/activate

# 3. Install torch (check nvidia-smi first for CUDA version)
pip install torch --index-url https://download.pytorch.org/whl/cu124

# 4. Install remaining deps
pip install triton transformers xxhash psutil

# 5. Install flash-attn (torch must be installed first)
pip install flash-attn --no-build-isolation

# 6. Run
python example.py
```
