# WSL ML 环境配置 — Claude Code Skill

在 **Windows + WSL** 上运行 ML 推理项目时，自动诊断并修复环境问题。

## 覆盖的 10 条规则

| # | 规则 | 解决的问题 |
|---|---|---|
| 1 | Python 3.13+ 太新 | flash-attn 无 cp313/cp314 预编译包 |
| 2 | CUDA 版本不匹配 | torch cu130 vs 系统 CUDA 12.4 |
| 3 | 依赖安装顺序很重要 | flash-attn 编译缺 torch 报错 |
| 4 | flash-attn ABI 敏感 | 优先用预编译 wheel，避免源码编译 |
| 5 | Windows 原生跑不了 triton | Triton/flash-attn 仅支持 Linux |
| 6 | PEP 668 "externally managed" | 必须使用 venv 虚拟环境 |
| 7 | 缺少 Python.h | Triton JIT 运行时编译需要 python3.x-dev |
| 8 | 国内下载慢 | 使用清华/阿里 pip 镜像加速 |
| 9 | WSL 代理配置 | 从 Windows 宿主机挂代理 |
| 10 | Windows/WSL 路径映射 | 使用 /mnt/c/... 绝对路径 |

## 安装方式

### 方式一：Git 克隆

```bash
git clone https://github.com/ball-out-of-tune/claude-code-skill-wsl-ml-setup.git ~/.claude/skills/wsl-ml-setup
```

### 方式二：手动复制

将 `skills/wsl-ml-setup.md` 复制到任意 Claude Code 项目的 `.claude/skills/` 目录下：

```
your-project/
└── .claude/
    └── skills/
        └── wsl-ml-setup.md
```

## 使用方式

安装后，当你遇到 ML 环境配置问题，Claude 会自动匹配并调用此 skill，无需手动触发。

开发中也可显式调用：

```
/wsl-ml-setup 帮我排查这个环境问题
```

## 触发条件

当 Claude 检测到以下错误时会自动激活：

- `ModuleNotFoundError: No module named 'triton'`
- `RuntimeError: The detected CUDA version mismatches`
- `fatal error: Python.h: No such file or directory`
- `error: externally-managed-environment`
- `undefined symbol: _ZN3c105ErrorC2E...`
- Windows/WSL 环境下 ML 推理相关的问题

## 实战验证

本 skill 于 2026-07-17 在 Windows 11 + WSL2 上配置 [nano-vllm](https://github.com/GeeeekExplorer/nano-vllm) + Qwen3-0.6B 时实战总结而成：

| 环境 | 详情 |
|---|---|
| 操作系统 | Windows 11 Home China 10.0.26200 |
| WSL | Ubuntu (kernel 5.15) |
| GPU | NVIDIA RTX (CUDA 12.4, 驱动 12.090) |
| Python | 3.12 (通过 deadsnakes PPA) |
| PyTorch | 2.6.0+cu124 |
| flash-attn | 2.7.4.post1 (预编译 wheel) |
| 模型 | Qwen3-0.6B |

## 许可证

MIT
