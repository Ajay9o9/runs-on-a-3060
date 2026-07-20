# Power underclock on RTX 3060 (LLM)

## Why

LLM decode is often **memory-bound**. Power limiting can cut heat/noise/power with modest tok/s loss.

## Commands used

```bash
sudo nvidia-smi -pm 1
sudo nvidia-smi -pl 115          # down from ~170W board limit / ~142W observed peak
sudo nvidia-smi -lgc 1550,1550   # lock GPU core clock
```

## Benchmark pair (TQ3_4S, 128k, ncmoe 20)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -p 131072 -n 128 \
  -ngl 99 -ncmoe 20 \
  -b 4096 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv tq3_0
```

| State | Peak power (lab) | pp131072 | tg128 |
|-------|------------------|----------|-------|
| Stock | ~**142 W** | 507.64 | **59.21** |
| 115 W + 1550 MHz | limited | 477.90 | **53.76** |

≈ **9%** generation loss for a large power/thermal cut.

## Related assets

Raw GPU logs/plots from the lab may be added under `data/underclock/` (CSV + charts from nvidia sampling).
