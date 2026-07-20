# recipes/

Short commands used or derived from the logs. Paths sanitized. Lab: `-t 12`, 64 GB RAM host.  
Always set **`-ctk` / `-ctv`** for LLM — see [../techniques/kv-cache.md](../techniques/kv-cache.md).

## Use cases

| | |
|--|--|
| [threejs-game-qwen-mtp.md](threejs-game-qwen-mtp.md) | Three.js game dev: Qwen 35B MTP Q6 server command + ik_llama better prefill |

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

**Runtime:** [TheTom/llama-cpp-turboquant](https://github.com/TheTom/llama-cpp-turboquant)  
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

## Three.js game server (Qwen MTP Q6)

Full write-up: [threejs-game-qwen-mtp.md](threejs-game-qwen-mtp.md)

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf \
  -c 132768 \
  -ngl 99 --n-cpu-moe 36 \
  -fa 1 -t 12 --no-mmap \
  -ctk q8_0 -ctv q8_0 \
  --jinja --chat-template-kwargs '{"preserve_thinking":true}' \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --presence-penalty 0.0 --repeat-penalty 1.0 \
  --spec-stage mtp:n_max=4 --metrics
```

Same model on **ik_llama**: higher prefill in lab benches (pp512 ~821 vs ~473 on llama.cpp e-mtp). Details on the use-case page.

Full logs: [../models/](../models/) · [../runtimes/comparison.md](../runtimes/comparison.md).
