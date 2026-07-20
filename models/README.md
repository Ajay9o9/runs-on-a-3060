# models/ (LLM)

LLM / multimodal text workloads. Image gen: [../image/](../image/).

**Machine:** RTX 3060 12GB · Ryzen 9 9900X · 64 GB RAM.

| Family | File | VRAM fit (weights) | Typical flags |
|--------|------|--------------------|---------------|
| Qwen3.6 35B-A3B (MoE) | [qwen3.6-35b-a3b.md](qwen3.6-35b-a3b.md) | no — hybrid | `-ngl 99` + `-ncmoe` / `exps=CPU` |
| Qwen3.6 27B MTP | [qwen3.6-27b.md](qwen3.6-27b.md) | partial | MTP flags |
| Gemma 4 12B | [gemma4-12b.md](gemma4-12b.md) | yes (Q5/Q6) | full GPU; Q8 partial `-ngl` |
| Gemma 4 26B-A4B (MoE) | [gemma4-26b-a4b.md](gemma4-26b-a4b.md) | hybrid | `-ncmoe` |
| Diffusion Gemma 26B-A4B | [diffusion-gemma.md](diffusion-gemma.md) | hybrid | diffusion CLI, `-ngl 15` |
| Ternary Bonsai 27B | [bonsai-ternary-27b.md](bonsai-ternary-27b.md) | full GPU in log | llama-server + benchy |

## Flag reference

| Flag | Meaning |
|------|---------|
| `-ngl N` | layers on GPU |
| `-ncmoe N` / `--n-cpu-moe N` | MoE expert layers on CPU |
| `-ot exps=CPU` | ik_llama: expert tensors on CPU |
| `--cpu-moe` | server: experts on CPU |
| `-t 12` | CPU threads (this machine) |
| `-ctk` / `-ctv` | K/V cache type |
| `-fa 1` | flash attention |

Also: [../techniques/moe-offload.md](../techniques/moe-offload.md), [../runtimes/comparison.md](../runtimes/comparison.md).
