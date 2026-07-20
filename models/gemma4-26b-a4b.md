# Gemma 4 26B-A4B (MoE) on RTX 3060

MoE 26B with ~4B active-class experts. Weights ~**21.6 GiB** — does **not** fully fit; use **CPU expert offload**. Needs healthy **system RAM** (lab: **64 GB**).

**KV (all benches on this page):** `-ctk q8_0 -ctv q8_0`

## Model

| File | Size | Params |
|------|------|--------|
| `gemma-4-26B-A4B-it-UD-Q6_K.gguf` | 21.57 GiB | 25.23 B |

## Offload vs VRAM (same flags family)

Pattern:

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/gemma-4-26B-A4B-it-UD-Q6_K.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe N \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| ncmoe | Peak VRAM | pp4096 | tg128 | Meaning |
|-------|-----------|--------|-------|---------|
| **32** | **3.81 GB** | 541.12 ± 4.74 | **33.53 ± 0.05** | Most experts on CPU → low VRAM |
| **20** | **10.1 GB** | 691.86 ± 9.60 | **39.48 ± 0.13** | More experts on GPU → faster, hungrier VRAM |

### Thread / batch sweep (ncmoe 22)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/gemma-4-26B-A4B-it-UD-Q6_K.gguf \
  -p 4096,8192 -n 128 \
  -ngl 99 -ncmoe 22 \
  -b 1024 -ub 512 -t 12,16 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| threads | pp4096 | pp8192 | tg128 |
|---------|--------|--------|-------|
| 12 | 644.05 | 634.91 | **37.96** |
| 16 | 655.90 | 632.11 | 37.33 |

→ Extra threads past 12 did not help gen on this 9900X setup.

### mmap off

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/gemma-4-26B-A4B-it-UD-Q6_K.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 22 \
  -b 1024 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0 \
  --mmap 0
```

| test | t/s |
|------|-----|
| pp4096 | **720.91 ± 7.07** |
| tg128 | 37.90 ± 0.23 |

Longer PP:

| test | t/s |
|------|-----|
| pp16384 | 684.75 ± 5.62 |
| pp32768 | 669.26 ± 14.35 |
| pp65536 | 607.41 ± 3.13 |
| tg128 | ~38 |

## Summary

| ncmoe | KV | Peak VRAM | tg128 |
|------:|-----|----------:|------:|
| 32 | q8_0 / q8_0 | 3.81 GB | 33.53 |
| 20 | q8_0 / q8_0 | 10.1 GB | 39.48 |

Host RAM: 64 GB on this machine.
