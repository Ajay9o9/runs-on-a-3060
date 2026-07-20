# MoE offload on 12GB: vertical vs horizontal

The single most important LLM lesson from this 3060 lab.

## Hardware assumption

- **GPU:** RTX 3060 **12GB**  
- **RAM:** **64 GB** (experts live here when offloaded)  
- **CPU:** Ryzen 9 9900X, usually `-t 12`

If you only have 16–32 GB RAM, copy the flags but expect OOM or thrashing on Q5/Q6 35B-class weights.

## Vertical split (`-ngl`)

Put **N transformer layers** on GPU; rest on CPU.

```bash
-ngl 20   # example
```

**Qwen3.6 Q4 TurboQuant, 4k:**

| ngl | tg128 |
|-----|------:|
| 16 | 27.4 |
| 18 | 28.8 |
| 20 | **30.3** |

Simple, but leaves gen on the table for MoE models.

## Horizontal split (`-ncmoe` / `--cpu-moe` / `-ot exps=CPU`)

Keep **attention (and non-expert)** on GPU with high `-ngl`, push **expert** tensors/layers to CPU.

```bash
-ngl 99 -ncmoe 48
# or
-ngl 99 -ot exps=CPU          # ik_llama
# or server:
-ngl 999 --cpu-moe
```

**Same Qwen Q4 TurboQuant, 4k:**

| n_cpu_moe | tg128 |
|-----------|------:|
| 64 | 43.0 |
| 56 | 43.3 |
| **48** | **43.5** |
| 40 | 42.8 |

≈ **+40% gen** vs best vertical in that series.

## Why RAM shows up in every hybrid recipe

Weight files:

| Quant | ~GiB on disk | Where it goes when experts are CPU |
|-------|--------------|--------------------------------------|
| Q4_K_XL | ~21 | Mostly **system RAM** + some VRAM for attention/KV |
| Q5_K_M | ~25 | Same |
| Q6_K | ~27–30 | Same |
| TQ3_4S | ~12.4 | Still hybrid for best speed / context |

**Users with 12GB VRAM but 16GB RAM cannot reproduce these hybrid speeds.** Document your RAM when you publish results.

## Practical recipe

1. Start `-ngl 99 -ncmoe 64` (max experts on CPU) — safe VRAM.  
2. Lower ncmoe (48 → 32 → 24 → 16) until VRAM is ~10–11 GB under your target context.  
3. Stop when tg drops (oversaturate) or OOM.  
4. Always log: `free -h`, `nvidia-smi`, and the full command.
