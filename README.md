# AMD Strix Halo Llama.cpp Toolboxes

This project provides pre-built containers for running LLMs on **AMD Ryzen AI Max "Strix Halo"** integrated GPUs.

---

## Quick Start

### 1. Create Toolbox (ROCm 7.2)

```sh
toolbox create llama-rocm-7.2 \
  --image docker.io/starballl/amd-strix-halo-toolboxes_ikllamacpp:rocm7.2 \
  -- --device /dev/dri --device /dev/kfd \
  --group-add video --group-add render --group-add sudo --security-opt seccomp=unconfined

toolbox enter llama-rocm-7.2
```

### 2. Check GPU Access

```sh
llama-cli --list-devices
```

### 3. Run Inference

```sh
llama-cli -c 8192 -ngl 999 -fa 1 --no-mmap -m models/your-model.gguf
```

> ⚠️ Important: Always use **flash attention** (`-fa 1`) and **no-mmap** (`--no-mmap`) on Strix Halo.

## Image Tags

| Tag | Description |
|-----|-------------|
| `rocm7.2` | ROCm 7.2 with llama.cpp |
| `latest` | Same as rocm7.2 |

## Building Locally

The Docker image is automatically built via GitHub Actions. To build manually:

```bash
docker build -f toolboxes/Dockerfile.rocm-7.2 -t my-image .
```

## References

- Original project: [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes)
