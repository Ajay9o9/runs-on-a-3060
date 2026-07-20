# MoE offload: `-ngl` vs `-ncmoe`

Flag notes and measured deltas on the lab machine.

## Hardware (this log set)

- GPU: RTX 3060 12GB  
- RAM: 64 GB (expert tensors often on host)  
- CPU: Ryzen 9 9900X, usually `-t 12`

Other RAM sizes will not match hybrid Q5/Q6 35B host footprints here.

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

Delta in this series: horizontal ~43 tg vs vertical ~30 tg on the same weights.

## Weight size vs host (when experts on CPU)

| Quant | ~GiB on disk | Host role |
|-------|--------------|-----------|
| Q4_K_XL | ~21 | experts largely in system RAM |
| Q5_K_M | ~25 | same |
| Q6_K | ~27–30 | same |
| TQ3_4S | ~12.4 | still often hybrid for speed / context |

## Flag sweep used in lab

```text
-ngl 99 -ncmoe 64   # then lower ncmoe while watching nvidia-smi
-ngl 99 -ncmoe 48
-ngl 99 -ncmoe 32
...
```

Log with each point: full command, `nvidia-smi`, `free -h` if possible.
