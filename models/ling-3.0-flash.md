# Ling-3.0-flash Q4_K_M on RTX 3060

**Ling-3.0-flash** MoE (atomic-chat GGUF) hybrid on **RTX 3060 12GB**.  
Lab date: **2026-08-06**.

**Runtime:** llama.cpp build under `krea2-model/ling-3.0-flash` (turboquant-oriented stack per atomic-chat Ling readme).  
**Bench tool:** [llama-benchy](https://github.com/eugr/llama-benchy) `0.3.8.dev2+gff162bcfc`.  
**Always log KV** when possible — this session did **not** pass `-ctk`/`-ctv` (default unknown; do not assume q8_0). Catalog: [../techniques/kv-cache.md](../techniques/kv-cache.md).

## Model

| Item | Value |
|------|--------|
| Quant | **Q4_K_M** (atomic-chat “STOCK” multi-shard) |
| Files | `Ling-3.0-flash-Q4_K_M_STOCK-00001-of-00002.gguf` (+ shard 2 same dir) |
| Lab path | `/media/aj-homeserver/windows/krea2-model/ling-3.0-flash/Q4_K_M_STOCK/` |
| Offload | hybrid: `-ngl 999` + `--n-cpu-moe` / `-ncmoe` |
| Hardware | RTX 3060 12GB · Ryzen 9 9900X · 64 GB RAM |

Serve **shard 1**; remaining shards load from the same directory.

## Server commands

### Bench session (ncmoe 42) — numbers below

```bash
cd /media/aj-homeserver/windows/krea2-model/ling-3.0-flash

./build/bin/llama-server \
  -m Q4_K_M_STOCK/Ling-3.0-flash-Q4_K_M_STOCK-00001-of-00002.gguf \
  --jinja -ngl 999 -ncmoe 42 -fa on -t 12 -np 1 \
  --host 127.0.0.1 --port 8080
```

### Also tried

```bash
# tighter host experts (ncmoe 39) — short 8k-oriented poke; not the benchy matrix
./build/bin/llama-server \
  -m Q4_K_M_STOCK/Ling-3.0-flash-Q4_K_M_STOCK-00001-of-00002.gguf \
  --jinja -ngl 999 -ncmoe 39 -fa on -t 12 -np 1
```

| Flag | Note |
|------|------|
| `-ngl 999` | non-overridden tensors on GPU |
| `-ncmoe 42` | experts of first 42 layer indices on CPU (bench matrix) |
| `-ncmoe 39` | more GPU experts; less host offload |
| `-fa on` | flash attention |
| `-t 12` | CPU threads (this machine) |
| **KV** | **not recorded** in lab log |

Weights live on the Windows SSD (`krea2-model`); prefer that over HDD for hybrid MoE mmap.

## llama-benchy

```bash
llama-benchy \
  --base-url http://127.0.0.1:8080 \
  --model Ling-3.0-flash-Q4_K_M \
  --served-model-name 'Q4_K_M_STOCK/Ling-3.0-flash-Q4_K_M_STOCK-00001-of-00002.gguf' \
  --tg 256 \
  --pp 4096 8192 16384 32768 \
  --no-warmup \
  --runs 1 \
  --save-result ling-3.0-flash-q4_k_m-benchy.md \
  --format md
```

**Config stamp:** ncmoe **42** · fa on · ngl 999 · depth 0 · runs **1** · no-warmup · endpoint `127.0.0.1:8080`.

## Results

| test | t/s | peak t/s | e2e_ttft (ms) |
|------|----:|---------:|--------------:|
| pp4096 | **29.66** | | ~127440 |
| tg256 | **14.36** | 25.00 | |
| pp8192 | **86.70** | | ~86587 |
| tg256 | **20.89** | 27.00 | |
| pp16384 | **119.03** | | ~125607 |
| tg256 | **18.85** | 27.00 | |
| pp32768 | **134.84** | | ~221724 |
| tg256 | **16.13** | 25.00 | |

### Summary

| PP | PP t/s | TG t/s (avg) | TG peak | TTFT (s) |
|---:|-------:|-------------:|--------:|---------:|
| 4096 | 29.66 | 14.36 | 25.00 | 127.4 |
| 8192 | 86.70 | 20.89 | 27.00 | 86.6 |
| 16384 | 119.03 | 18.85 | 27.00 | 125.6 |
| 32768 | 134.84 | 16.13 | 25.00 | 221.7 |

**Read-out:** first **pp4096** is anomalously slow (cold mmap / hybrid warm-up); treat **pp8k–32k (~87–135 t/s)** and **tg ~16–21 t/s** (peaks ~25–27) as the steadier band. Single-run benchy (`--runs 1`) — no ± spread.

## Lab takeaways

- Fits **3060 hybrid** with **ncmoe 42** (and 39 poke).
- Decode usable in the **~15–21 t/s** range after warm-up; not in the same class as Qwen TQ3 (~54 tg) or IQ3 Laguna (~23 tg).
- Log **`-ctk`/`-ctv`** next time; record peak VRAM + `free -h` under load.
- For chatty long runs, keep GGUF on **SSD** (Windows `krea2-model`), not GAMES1 HDD.

Snapshot: [../RESULTS.md](../RESULTS.md#ling-30-flash-q4_k_m). Related MoE offload: [../techniques/moe-offload.md](../techniques/moe-offload.md).
