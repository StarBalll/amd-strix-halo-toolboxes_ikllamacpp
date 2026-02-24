# AMD Strix Halo Toolboxes (ik_llama.cpp Fork)

本项目基于 [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes)，主要添加了 **ik_llama.cpp** 版本的镜像构建支持。

ik_llama.cpp 是 llama.cpp 的一个高性能分叉，支持先进的 IQ 量化类型（在同等显存下质量更好）。

---

## 镜像列表

| 镜像 Tag | 后端 | llama.cpp 版本 | 说明 |
|---------|------|---------------|------|
| `rocm-7.2` | ROCm 7.2 (HIP) | ggerganov/llama.cpp | 原版，稳定 |
| `rocm-7.2-ik` | ROCm 7.2 (HIP) | ikawrakow/ik_llama.cpp | 支持 IQ 量化 |
| `rocm-nightlies-ik` | ROCm Nightly (HIP) | ikawrakow/ik_llama.cpp | 最新 ROCm + IQ 量化 |
| `vulkan-radv-ik` | Vulkan (RADV) | ikawrakow/ik_llama.cpp | 支持 IQ 量化 ⚠️ MoE 模型有 bug |

---

## 快速开始

### 创建 Toolbox

```sh
# ROCm 7.2 原版
toolbox create llama-rocm-7.2 \
  --image docker.io/starballl/amd-strix-halo-toolboxes_ikllamacpp:rocm-7.2 \
  -- --device /dev/dri --device /dev/kfd \
  --group-add video --group-add render --group-add sudo --security-opt seccomp=unconfined

# ROCm 7.2 IK (支持 IQ 量化)
toolbox create llama-rocm-7.2-ik \
  --image docker.io/starballl/amd-strix-halo-toolboxes_ikllamacpp:rocm-7.2-ik \
  -- --device /dev/dri --device /dev/kfd \
  --group-add video --group-add render --group-add sudo --security-opt seccomp=unconfined

# ROCm Nightlies IK (最新 ROCm)
toolbox create llama-rocm-nightlies-ik \
  --image docker.io/starballl/amd-strix-halo-toolboxes_ikllamacpp:rocm-nightlies-ik \
  -- --device /dev/dri --device /dev/kfd \
  --group-add video --group-add render --group-add sudo --security-opt seccomp=unconfined

# Vulkan RADV IK (不支持 MoE 模型!)
toolbox create llama-vulkan-radv-ik \
  --image docker.io/starballl/amd-strix-halo-toolboxes_ikllamacpp:vulkan-radv-ik \
  -- --device /dev/dri --group-add video --security-opt seccomp=unconfined
```

### 进入容器

```sh
toolbox enter llama-rocm-7.2-ik
```

### 运行推理

```sh
# 强制使用 GPU (ROCm)
llama-cli -m model.gguf -n 128 -ngl 999 -fa 1 --no-mmap

# Vulkan 模式
llama-cli -m model.gguf -n 128 -ngl 999 -fa 1
```

---

## 重要参数

| 参数 | 说明 |
|------|------|
| `-ngl 999` | 加载所有层到 GPU |
| `-fa 1` | 启用 Flash Attention |
| `--no-mmap` | 禁用内存映射（避免问题） |

---

## 已知问题

### 1. Vulkan RADV IK - MoE 模型不兼容

**问题描述**：ik_llama.cpp 的 Vulkan 后端对 MoE (Mixture of Experts) 模型（如 Qwen3-Coder-30B-A3B）有已知 bug，会导致速度极慢或无限循环。

**参考**：[ik_llama.cpp Issue #641](https://github.com/ikawrakow/ik_llama.cpp/issues/641)

**解决方案**：
- 使用 ROCm 后缀的镜像（推荐）
- 或使用 dense 类型的模型（非 MoE）

### 2. ROCm 路径问题

**问题描述**：ROCm 7.2 安装路径为 `/opt/rocm-7.2.0`，不是传统的 `/opt/rocm`。

**已修复**：所有 Dockerfile 已更新正确的路径。

---

## IQ 量化模型推荐

ik_llama.cpp 的核心优势是支持 IQ 量化类型，以下是 Qwen3-Coder-30B-A3B 的推荐下载：

**来源**：[ubergarm/Qwen3-Coder-30B-A3B-Instruct-GGUF](https://huggingface.co/ubergarm/Qwen3-Coder-30B-A3B-Instruct-GGUF)

| 量化类型 | 大小 | 质量 (PPL) |
|---------|------|-----------|
| IQ5_K | 21.3 GB | 9.59 |
| IQ4_K | 17.9 GB | 9.60 |
| IQ4_KSS | 15.5 GB | 9.64 |
| IQ3_K | 14.5 GB | 9.68 |
| IQ3_KS | 13.6 GB | 9.79 |

> 注意：IQ 量化只能用 ik_llama.cpp 运行，原版 llama.cpp 不支持！

---

## 性能对比

| 镜像 | 后端 | Token 生成速度 |
|------|------|--------------|
| rocm-7.2 | ROCm 7.2 | ~50 t/s |
| rocm-nightlies-ik | ROCm Nightly | ~49 t/s (MoE 模型) |
| vulkan-radv | Vulkan RADV (原版) | ~60+ t/s |
| vulkan-radv-ik | Vulkan RADV (IK) | ~49 t/s |

**说明**：
- Vulkan 原版性能最好
- ik_llama.cpp 优势在于 IQ 量化（同显存下质量更好），而非绝对性能
- MoE 模型建议使用 ROCm 后端

---

## 本地构建

```bash
# 构建特定镜像
docker build -f toolboxes/Dockerfile.rocm-7.2-ik -t my-image .

# 构建所有镜像（需要 GitHub Actions）
```

---

## 项目结构

```
.
├── .github/workflows/
│   ├── build-rocm-7.2.yml          # ROCm 7.2 原版构建
│   ├── build-rocm-7.2-ik.yml       # ROCm 7.2 IK 构建
│   ├── build-rocm-nightlies-ik.yml  # ROCm Nightlies IK 构建
│   └── build-vulkan-radv-ik.yml     # Vulkan RADV IK 构建
├── toolboxes/
│   ├── Dockerfile.rocm-7.2
│   ├── Dockerfile.rocm-7.2-ik
│   ├── Dockerfile.rocm7-nightlies-ik-llama
│   ├── Dockerfile.vulkan-radv-ik-llama
│   └── gguf-vram-estimator.py
└── README.md
```

---

## 参考链接

- 原始项目：[kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes)
- ik_llama.cpp：[https://github.com/ikawrakow/ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp)
- IQ 量化模型：[ubergarm/Qwen3-Coder-30B-A3B-Instruct-GGUF](https://huggingface.co/ubergarm/Qwen3-Coder-30B-A3B-Instruct-GGUF)
