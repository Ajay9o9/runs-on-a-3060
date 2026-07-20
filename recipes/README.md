# recipes/

Short commands used or derived from the logs. Paths sanitized. Lab: `-t 12`, 64 GB RAM host.  
Always set **`-ctk` / `-ctv`** for LLM — see [../techniques/kv-cache.md](../techniques/kv-cache.md).

## Qwen 35B TQ3 (llama.cpp-tq3)

**KV:** `q4_0` / `tq3_0`

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -c 8192 -b 512 -ub 512 -t 12 \
  -ngl 99 -ncmoe 24 \
  -fa 1 -ctk q4_0 -ctv tq3_0 \
  --host 0.0.0.0 --port 8080
```

## Qwen 35B Q4 TurboQuant KV

**KV:** `turbo4` / `turbo3`

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -c 4096 -b 512 -ub 512 -t 12 \
  -ngl 999 --cpu-moe \
  -fa 1 -ctk turbo4 -ctv turbo3 \
  --host 0.0.0.0 --port 8080
```

## Gemma 4 12B Q5 + vision

**KV:** `q8_0` / `q8_0`

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q5_K_XL.gguf \
  --mmproj $MODEL_DIR/mmproj-F16.gguf \
  -c 32768 -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0 \
  --jinja --host 0.0.0.0 --port 8080
```

## Generic llama-bench pattern

**KV:** set explicitly (example `q8_0` / `q8_0`)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/your-model.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

## Power limit (as measured)

```bash
sudo nvidia-smi -pl 115
sudo nvidia-smi -lgc 1550,1550
```

Full logs: [../models/](../models/) · [../runtimes/comparison.md](../runtimes/comparison.md).
