# models/ (LLM)

LLM / multimodal text workloads. Image gen: [../image/](../image/).

**Machine:** RTX 3060 12GB · Ryzen 9 9900X · 64 GB RAM.

Every LLM page should list **KV cache** (`-ctk` / `-ctv`). Catalog: [../techniques/kv-cache.md](../techniques/kv-cache.md).

| Family | File | VRAM fit (weights) | Typical offload | Typical KV |
|--------|------|--------------------|-----------------|------------|
| Qwen3.6 35B-A3B (MoE) | [qwen3.6-35b-a3b.md](qwen3.6-35b-a3b.md) | no — hybrid | `-ngl 99` + `-ncmoe` / `exps=CPU` | tq3 / turbo / q8 / q4 (see page) |
| Qwen3.6 27B MTP | [qwen3.6-27b.md](qwen3.6-27b.md) | partial | MTP flags | q4_0/q4_0 or q8_0/q8_0 |
| Gemma 4 12B | [gemma4-12b.md](gemma4-12b.md) | yes (Q5/Q6/QAT Q4) | full GPU; Q8 partial `-ngl`; QAT ± MTP long-ctx | **q8_0 / q8_0** |
| Gemma 4 26B-A4B (MoE) | [gemma4-26b-a4b.md](gemma4-26b-a4b.md) | hybrid | `-ncmoe` | **q8_0 / q8_0** |
| Diffusion Gemma 26B-A4B | [diffusion-gemma.md](diffusion-gemma.md) | hybrid | llama.cpp branch `nvidia-diffusion-gemma`, `-ngl 15` | **not recorded** |
| Ternary Bonsai 27B | [bonsai-ternary-27b.md](bonsai-ternary-27b.md) | full GPU in log | llama-server + benchy | **not recorded** |
| Laguna S-2.1 (118B-A8B MoE) | [laguna-s-2.1.md](laguna-s-2.1.md) | no — hybrid (~46–73 GB) | ngl 999 + ncmoe 44–46 | f16 / q8_0 / q4_0 (see page) |

## Flag reference

| Flag | Meaning |
|------|---------|
| `-ngl N` | layers on GPU |
| `-ncmoe N` / `--n-cpu-moe N` | MoE expert layers on CPU |
| `-ot exps=CPU` | ik_llama: expert tensors on CPU |
| `--cpu-moe` | server: experts on CPU |
| `-t 12` | CPU threads (this machine) |
| **`-ctk` / `-ctv`** | **K/V cache type (required)** |
| `-fa 1` | flash attention |

Also: [../techniques/moe-offload.md](../techniques/moe-offload.md), [../runtimes/comparison.md](../runtimes/comparison.md).
