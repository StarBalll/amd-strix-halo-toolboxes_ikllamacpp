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
# ROCm 后端
llama-cli -m model.gguf -n 128 -ngl 999 -fa 1 --no-mmap

# Vulkan 模式
llama-cli -m model.gguf -n 128 -ngl 999 -fa 1
```

### 推荐运行参数 (llama-server)

```sh
llama-server \
  --alias qwen3-coder-next \
  --jinja \
  --flash-attn on \
  -ngl 999 \
  -c 65535 \
  --no-mmap \
  --host 0.0.0.0 \
  --port 8080 \
  -b 16384 \
  -ub 4096 \
  --mlock \
  --parallel 2 \
  --seed 3407 \
  --temp 1.0 \
  --top-p 0.95 \
  --top-k 40 \
  --min-p 0.01 \
  -m /models/Qwen3-Coder-Next-Q4_K_M.gguf
```

---

## 重要参数说明

| 参数 | 说明 |
|------|------|
| `-ngl 999` | 加载所有层到 GPU |
| `--flash-attn on` 或 `-fa 1` | 启用 Flash Attention |
| `--no-mmap` | 禁用内存映射（避免 ROCm 问题） |
| `-c 65535` | 最大上下文长度 |
| `-b 16384` | batch size |
| `-ub 4096` | ubatch size |
| `--mlock` | 锁定内存，防止换出 |

---

## 已知问题

### 1. Vulkan RADV IK - MoE 模型不兼容 ⚠️

**问题描述**：ik_llama.cpp 的 Vulkan 后端对 MoE (Mixture of Experts) 模型（如 Qwen3-Coder-30B-A3B）有已知 bug，会导致速度极慢或无限循环。

**参考**：[ik_llama.cpp Issue #641](https://github.com/ikawrakow/ik_llama.cpp/issues/641)

**解决方案**：
- 使用 ROCm 后缀的镜像（推荐）
- 或使用 dense 类型的模型（非 MoE）

### 2. ROCm 路径问题

**问题描述**：ROCm 7.2 安装路径为 `/opt/rocm-7.2.0`，不是传统的 `/opt/rocm`。

**已修复**：所有 Dockerfile 已更新正确的路径。

### 3. ik_llama.cpp Vulkan 性能

**说明**：ik_llama.cpp 的 Vulkan 后端性能不如原版 ggerganov/llama.cpp，这是设计差异。ik 的优势在于 CPU 优化和 IQ 量化，而不是 Vulkan。

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

## 构建要点与难点

### 1. ROCm 路径问题

- **问题**：ROCm 7.2 安装路径为 `/opt/rocm-7.2.0`，不是传统的 `/opt/rocm`
- **解决**：在 Dockerfile 中设置正确的环境变量：
  ```dockerfile
  ENV ROCM_PATH=/opt/rocm-7.2.0
  ENV HIP_PATH=/opt/rocm-7.2.0
  ENV LD_LIBRARY_PATH=/opt/rocm-7.2.0/lib:/opt/rocm-7.2.0/lib64:$LD_LIBRARY_PATH
  ```

### 2. ROCm Nightly 构建

- **问题**：ROCm Nightly 使用 TheRock 提供的 tarball，需要动态获取最新版本
- **解决**：在 Dockerfile 中使用 curl 动态获取最新 tarball：
  ```dockerfile
  BASE="https://therock-nightly-tarball.s3.amazonaws.com"
  PREFIX="therock-dist-linux-${GFX}-${ROCM_MAJOR_VER}"
  KEY="$(curl -s "${BASE}?list-type=2&prefix=${PREFIX}" ... | sort -V | tail -n1)"
  aria2c -x 16 -s 16 -j 16 "${BASE}/${KEY}" -o therock.tar.gz
  ```

### 3. 运行时环境变量

- **问题**：容器运行时需要正确的 ROCm 环境变量
- **解决**：在 Dockerfile 中添加 profile 脚本：
  ```dockerfile
  RUN printf '%s\n' \
    'export ROCM_PATH=/opt/rocm-7.2.0' \
    'export HIP_PATH=/opt/rocm-7.2.0' \
    > /etc/profile.d/rocm.sh \
    && chmod +x /etc/profile.d/rocm.sh
  ```

### 4. ik_llama.cpp 与原版的差异

- **问题**：ik_llama.cpp 的 Vulkan 后端性能不如原版，且对 MoE 模型有 bug
- **说明**：这是 ik_llama.cpp 本身的设计取向，主要优化 CPU 性能和 IQ 量化
- **建议**：根据需求选择合适的镜像

### 5. Docker Hub 权限

- **问题**：构建时需要 Docker Hub 推送权限
- **解决**：在 GitHub Secrets 中配置：
  - `DOCKER_USERNAME`：Docker Hub 用户名
  - `DOCKER_PASSWORD`：具有 Write 权限的 Access Token

---

## 本地构建

```bash
# 构建特定镜像
docker build -f toolboxes/Dockerfile.rocm-7.2-ik -t my-image .

# 构建并推送（需要登录）
docker login
docker build -f toolboxes/Dockerfile.rocm-7.2-ik -t starballl/amd-strix-halo-toolboxes_ikllamacpp:rocm-7.2-ik .
docker push starballl/amd-strix-halo-toolboxes_ikllamacpp:rocm-7.2-ik
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
