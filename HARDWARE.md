# Test hardware

All numbers in this repo were measured on **one machine**. Offload recipes that put large MoE experts on CPU depend heavily on system RAM and CPU — a 3060 alone is not enough context.

## Specs

| Component | Spec |
|-----------|------|
| **GPU** | NVIDIA GeForce **RTX 3060 12GB** |
| **VRAM** | ~11887–11889 MiB (as reported by llama.cpp) |
| **Compute capability** | 8.6 (Ampere) |
| **CPU** | AMD **Ryzen 9 9900X** (12 cores / 24 threads) |
| **System RAM** | **64 GB** DDR5 @ **6000 MT/s** |
| **OS** | Ubuntu 24.04 |
| **Board** | B650M (desktop) |

## Why RAM matters for these results

Many of the “big” models here do **not** fit fully in 12GB VRAM:

| Weight size (approx) | Fits fully on 3060? | How we ran it |
|----------------------|---------------------|---------------|
| Gemma 4 12B Q5 (~8 GiB) | Yes | Full GPU (`-ngl -1` / `999`) |
| Gemma 4 12B Q6 (~10 GiB) | Barely (KV eats headroom) | Full GPU, watch context |
| Qwen3.6 35B Q4 (~21 GiB) | No | Hybrid: attention GPU, experts mostly **CPU** |
| Qwen3.6 35B Q5 (~25 GiB) | No | Same |
| Qwen3.6 35B Q6 (~27–30 GiB) | No | Same |
| Qwen3.6 TQ3_4S (~12.4 GiB weights) | Borderline | Hybrid MoE + compressed KV |

**Rule of thumb from this box:**

- Expect **tens of GB of system RAM** to be used when experts live on CPU (`-ncmoe`, `-ot exps=CPU`, `--cpu-moe`).
- Example memory breakdown (Qwen3.6 MTP Q6, short prompt, llama.cpp): Host side ~**27–31 GiB** self-reported for model/context/compute — you need headroom beyond the weight file size.
- Threads used in benches: **`-t 12`** (one per physical core on the 9900X). More threads did not always help (see Gemma 26B tests).

## If your RAM is smaller

| Your RAM | Expectation vs this repo |
|----------|---------------------------|
| **32 GB** | Many hybrid Qwen Q5/Q6 configs will be tight or OOM; prefer smaller quants / more aggressive CPU expert offload / lower context |
| **64 GB** (this machine) | Comfortable for Qwen 35B Q4–Q6 hybrid + long context experiments |
| **16 GB** | Stick to full-GPU 7–12B class; hybrid 35B is unlikely to be pleasant |

## GPU power notes

- Stock peaks observed ~**140–142 W** on heavy TQ3 / long-context runs.
- Underclock recipe used: **power limit 115 W**, core locked **1550 MHz** → roughly **~9%** slower gen, much cooler (see [techniques/power-underclock.md](techniques/power-underclock.md)).

## Path convention in this repo

Commands use placeholders:

```bash
export MODEL_DIR=/path/to/your/ggufs
export LLAMA_BIN=./build/bin   # from the runtime you built
```

Replace with your paths. Original lab paths were local SSD mounts and are intentionally not required to reproduce the ideas.
