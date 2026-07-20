# Test hardware

All numbers in this repo were measured on the machine below.

## Specs

| Component | Spec |
|-----------|------|
| GPU | NVIDIA GeForce RTX 3060 12GB |
| VRAM | ~11887–11889 MiB (llama.cpp report) |
| Compute capability | 8.6 |
| CPU | AMD Ryzen 9 9900X (12 cores / 24 threads) |
| System RAM | 64 GB DDR5 @ 6000 MT/s |
| OS | Ubuntu 24.04 |
| Board | B650M |

## Model weight size vs 12GB VRAM

| Weight size (approx) | Fits in 12GB VRAM alone | How it was run here |
|----------------------|-------------------------|---------------------|
| Gemma 4 12B Q5 (~8 GiB) | yes | full GPU (`-ngl -1` / `999`) |
| Gemma 4 12B Q6 (~10 GiB) | tight (KV) | full GPU, limited context headroom |
| Qwen3.6 35B Q4 (~21 GiB) | no | hybrid; experts largely on CPU/RAM |
| Qwen3.6 35B Q5 (~25 GiB) | no | same |
| Qwen3.6 35B Q6 (~27–30 GiB) | no | same |
| Qwen3.6 TQ3_4S (~12.4 GiB weights) | borderline | hybrid MoE + compressed KV |

## RAM usage notes

- Hybrid MoE (`-ncmoe`, `-ot exps=CPU`, `--cpu-moe`) keeps large expert tensors in **system RAM**.
- Example host footprint (Qwen3.6 MTP Q6, short prompt): llama host breakdown on the order of **~27–31 GiB** self — need free RAM beyond the GGUF file size.
- Bench threads: usually `-t 12`.

| System RAM | Relevance to these logs |
|------------|-------------------------|
| 64 GB | matches this machine |
| 32 GB | hybrid Q5/Q6 35B may be tight |
| 16 GB | hybrid 35B configs here are not a fair comparison |

## GPU power samples

| Condition | Observation |
|-----------|-------------|
| Stock heavy TQ3 / long ctx | peak ~140–142 W |
| `nvidia-smi -pl 115` + core 1550 MHz | lower power; see [techniques/power-underclock.md](techniques/power-underclock.md) |

## Paths in commands

```bash
export MODEL_DIR=/path/to/your/ggufs
export LLAMA_BIN=./build/bin
```

Lab absolute paths are not required.
