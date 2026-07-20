# Qwen3.6 27B MTP on RTX 3060

Thinner notes than 35B-A3B, but present in the lab log.

## Model

| File | Notes |
|------|--------|
| `Qwen3.6-27B-MTP-Q4_K_M.gguf` | MTP-capable Q4 |

## Bench command (experts CPU)

**KV:** `q4_0` / `q4_0`

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-27B-MTP-Q4_K_M.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ot exps=CPU \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv q4_0
```

## CLI with draft-MTP (long prompt budget)

**KV:** `q8_0` / `q8_0`

```bash
./build/bin/llama-cli \
  -m $MODEL_DIR/Qwen3.6-27B-MTP-Q4_K_M.gguf \
  -ngl 999 -fa 1 -t 12 \
  -p 65512 \
  -ctk q8_0 -ctv q8_0 \
  --spec-type draft-mtp --spec-draft-n-max 4
```

## Hardware context

Same box: **RTX 3060 12GB** + **64 GB RAM** + 9900X.  
Prefer the [35B-A3B page](qwen3.6-35b-a3b.md) for detailed MoE / MTP tables — methodology is the same class of experiment.
