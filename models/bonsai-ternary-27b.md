# Ternary Bonsai 27B (LLM) on RTX 3060

> **Name collision:** This is a **text LLM** (GGUF, llama-server).  
> For **image** generation see [../image/bonsai-image-4b.md](../image/bonsai-image-4b.md) (PrismML Bonsai Image 4B).

## Model

| File | Type |
|------|------|
| `Ternary-Bonsai-27B-Q2_0.gguf` | Ternary / low-bit 27B LLM |

## Server command

```bash
./bin/cuda/llama-server \
  -m $MODEL_DIR/Ternary-Bonsai-27B-Q2_0.gguf \
  -ngl 999 -np 1 -fa on \
  -c 51840 \
  --jinja \
  --temp 0.7 --top-p 0.95 --top-k 20 --min-p 0 \
  --host 127.0.0.1 --port 8000
```

**Offload:** `-ngl 999` = try full GPU. Host still has **64 GB RAM** for OS / runtime overhead.  
**KV:** **not recorded** in the lab log (no `-ctk`/`-ctv` in the saved command). Runtime default unknown — do not assume q8_0.

## llama-benchy (against server)

```bash
llama-benchy \
  --base-url http://127.0.0.1:8000 \
  --model Ternary-Bonsai-27B-Q2_0.gguf \
  --tg 256 \
  --pp 24576 36864 49152 \
  --no-warmup --runs 3
```

## Results

| test | t/s | peak t/s | e2e_ttft (ms) |
|------|----:|---------:|--------------:|
| pp24576 | **441.48 ± 4.72** | | ~50193 |
| tg256 | **25.49 ± 0.59** | 31.33 | |
| pp36864 | 415.26 ± 18.03 | | ~80636 |
| tg256 | 25.19 ± 4.22 | 33.00 | |
| pp49152 | 407.08 ± 20.87 | | ~109956 |
| tg256 | **23.37 ± 1.39** | 28.33 | |

## Summary

| Metric | Value |
|--------|-------|
| tg256 | ~23–25 t/s |
| PP sizes tested | 24576, 36864, 49152 |

Related MoE logs: [qwen3.6-35b-a3b.md](qwen3.6-35b-a3b.md).
