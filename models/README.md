# LLM models tried on RTX 3060 12GB

All of these are **text / multimodal LLM** workloads (not image diffusion like SD/Flux). Image gen lives under [`../image/`](../image/).

**Host always:** RTX 3060 12GB · Ryzen 9 9900X · **64 GB RAM**.

| Family | Page | Fits in 12GB? | Typical approach |
|--------|------|---------------|------------------|
| Qwen3.6 35B-A3B (MoE) | [qwen3.6-35b-a3b.md](qwen3.6-35b-a3b.md) | Weights no — hybrid yes | `-ngl 99` + `-ncmoe` / `exps=CPU` |
| Qwen3.6 27B MTP | [qwen3.6-27b.md](qwen3.6-27b.md) | Partial | Dense-ish path + MTP |
| Gemma 4 12B | [gemma4-12b.md](gemma4-12b.md) | Yes (Q5/Q6) | Full GPU; Q8 needs layer split |
| Gemma 4 26B-A4B (MoE) | [gemma4-26b-a4b.md](gemma4-26b-a4b.md) | Hybrid | `-ncmoe` VRAM tradeoffs |
| Diffusion Gemma 26B-A4B | [diffusion-gemma.md](diffusion-gemma.md) | Hybrid | diffusion CLI, ngl 15 |
| Ternary Bonsai 27B | [bonsai-ternary-27b.md](bonsai-ternary-27b.md) | Full GPU in lab run | llama-server + benchy |

## Offload flags cheat sheet

| Flag | Meaning |
|------|---------|
| `-ngl N` | GPU layers (vertical split) |
| `-ncmoe N` / `--n-cpu-moe N` | Force N MoE expert layers onto **CPU** (horizontal) |
| `-ot exps=CPU` | ik_llama: put expert tensors on CPU |
| `--cpu-moe` | server: keep MoE experts on CPU |
| `-t 12` | CPU threads (matched 9900X cores here) |
| `-ctk` / `-ctv` | K/V cache quant (q4_0, q8_0, turbo4/turbo3, tq3_0) |
| `-fa 1` | flash attention on |

See [../techniques/moe-offload.md](../techniques/moe-offload.md) and [../runtimes/comparison.md](../runtimes/comparison.md).
