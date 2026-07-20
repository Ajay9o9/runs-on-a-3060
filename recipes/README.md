# Quick recipes (copy-paste)

Sanitize paths. Default threads `-t 12` matched the lab **9900X**; adjust. Hybrid MoE assumes **plenty of system RAM** (lab: **64 GB**).

## 1) Qwen 35B — max gen (TQ3)

```bash
# runtime: llama.cpp-tq3
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -c 8192 -b 512 -ub 512 -t 12 \
  -ngl 99 -ncmoe 24 \
  -fa 1 -ctk q4_0 -ctv tq3_0 \
  --host 0.0.0.0 --port 8080
```

## 2) Qwen 35B — Unsloth Q4 hybrid (TurboQuant KV)

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -c 4096 -b 512 -ub 512 -t 12 \
  -ngl 999 --cpu-moe \
  -fa 1 -ctk turbo4 -ctv turbo3 \
  --host 0.0.0.0 --port 8080
```

## 3) Gemma 4 12B — full GPU daily driver

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q5_K_XL.gguf \
  --mmproj $MODEL_DIR/mmproj-F16.gguf \
  -c 32768 -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0 \
  --jinja --host 0.0.0.0 --port 8080
```

## 4) Bench any GGUF the way we did

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/your-model.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

## 5) Quiet GPU (115 W)

```bash
sudo nvidia-smi -pl 115
sudo nvidia-smi -lgc 1550,1550
```

Deep dives: [../models/](../models/) · [../runtimes/comparison.md](../runtimes/comparison.md).
