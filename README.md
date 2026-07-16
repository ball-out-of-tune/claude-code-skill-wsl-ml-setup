# WSL ML Setup — Claude Code Skill

Diagnose and fix environment issues when running ML inference projects (vLLM, nano-vllm, FlashAttention, etc.) on **Windows + WSL**.

## What This Skill Does

This skill teaches Claude how to systematically troubleshoot and resolve the 10 most common pitfalls when setting up ML inference on WSL:

| # | Rule | Problem |
|---|---|---|
| 1 | Python 3.13+ is too new | flash-attn has no wheel for cp313/cp314 |
| 2 | CUDA version mismatch | torch cu130 vs system CUDA 12.4 |
| 3 | Dependency install order | flash-attn build fails without torch |
| 4 | flash-attn ABI fragile | Use pre-built wheels, not source compile |
| 5 | Windows native can't run triton | Triton/flash-attn are Linux-only |
| 6 | PEP 668 "externally managed" | Always use venv |
| 7 | Missing Python.h | Triton JIT needs python3.x-dev |
| 8 | Slow pip in China | Use Tsinghua/Aliyun mirrors |
| 9 | WSL proxy config | Proxy from Windows host |
| 10 | Model path between Win/WSL | /mnt/c/... absolute paths |

## Installation

### Option 1: Git Clone

```bash
git clone https://github.com/ball-out-of-tune/claude-code-skill-wsl-ml-setup.git ~/.claude/skills/wsl-ml-setup
```

### Option 2: Manual Copy

Download `skills/wsl-ml-setup.md` and place it in `.claude/skills/` in any Claude Code project:

```
your-project/
└── .claude/
    └── skills/
        └── wsl-ml-setup.md
```

## Usage

Once installed, Claude automatically detects when you're having ML environment issues and invokes this skill. No manual trigger needed.

## When It Triggers

- `ModuleNotFoundError: No module named 'triton'`
- `RuntimeError: The detected CUDA version mismatches`
- `fatal error: Python.h: No such file or directory`
- `error: externally-managed-environment`
- `undefined symbol: _ZN3c105ErrorC2E...`
- Slow ML inference setup on Windows/WSL

## Real-World Provenance

This skill was battle-tested on 2026-07-17 setting up [nano-vllm](https://github.com/GeeeekExplorer/nano-vllm) with Qwen3-0.6B on Windows 11 + WSL2:

| Environment | Details |
|---|---|
| OS | Windows 11 Home China 10.0.26200 |
| WSL | Ubuntu (kernel 5.15) |
| GPU | NVIDIA RTX (CUDA 12.4, driver 12.090) |
| Python | 3.12 (via deadsnakes PPA) |
| PyTorch | 2.6.0+cu124 |
| flash-attn | 2.7.4.post1 (pre-built wheel) |
| Model | Qwen3-0.6B |

## License

MIT
