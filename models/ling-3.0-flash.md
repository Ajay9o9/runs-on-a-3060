# Ling-3.0-flash Q4_K_M on RTX 3060

**Ling-3.0-flash** MoE (atomic-chat GGUF) hybrid on **RTX 3060 12GB**.  
Lab date: **2026-08-06**.

**Runtime:** **[atomic-llama-cpp-turboquant](https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant)** (TurboQuant-enabled llama.cpp fork, release **b10269-1.5.0+**), installed under `krea2-model/ling-3.0-flash`. **Stock llama.cpp cannot load these GGUFs.** Runtime notes: [../runtimes/README.md](../runtimes/README.md#atomic-llama-cpp-turboquant-atomicchat-ling).  
**Bench tool:** [llama-benchy](https://github.com/eugr/llama-benchy) `0.3.8.dev2+gff162bcfc`.  
**KV:** f16 (this session). Catalog: [../techniques/kv-cache.md](../techniques/kv-cache.md).

## Model

| Item | Value |
|------|--------|
| Card | [AtomicChat/Ling-3.0-flash-GGUF](https://huggingface.co/AtomicChat/Ling-3.0-flash-GGUF) (of `inclusionAI/Ling-3.0-flash`) |
| Arch | **124B total / 5.1B active**, hybrid linear attention, **512-expert** MoE |
| Quant | **Q4_K_M** (atomic-chat “STOCK” multi-shard) |
| Files | `Ling-3.0-flash-Q4_K_M_STOCK-00001-of-00002.gguf` (+ shard 2 same dir) |
| Lab path | `/media/aj-homeserver/windows/krea2-model/ling-3.0-flash/Q4_K_M_STOCK/` |
| Offload | hybrid: `-ngl 999` + `--n-cpu-moe` / `-ncmoe` |
| Hardware | RTX 3060 12GB · Ryzen 9 9900X · 64 GB RAM |

Serve **shard 1**; remaining shards load from the same directory.

> **STOCK is a control file.** Upstream ships `AD-*` as the production rungs; `*_STOCK` / `*_FLAT` exist to show
> what the AD quant strategy buys and are not meant for real use. The card's 64 GB rung is **`AD-IQ3_M`**
> (62.2 GB, KLD 0.0481) — worth a rerun on this box for a fair quality/speed read.

## Runtime install

Prebuilt, no compile (see [../runtimes/README.md](../runtimes/README.md#atomic-llama-cpp-turboquant-atomicchat-ling)):

```bash
wget https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant/releases/download/b10269-1.5.0/llama-turboquant-linux-x64-cuda-13.3.tar.gz
tar xzf llama-turboquant-linux-x64-cuda-13.3.tar.gz && cd llama-turboquant-*
```

Chat template ships **inside** the GGUF (thinking mode + tool calling) — hence `--jinja`.
Author sampling: **temp 0.6 · top_p 0.95 · top_k 20**.

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
| `--jinja` | chat template embedded in the GGUF (thinking + tools) |
| **KV** | **f16** (no `-ctk`/`-ctv` override) |

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

**Config stamp:** ncmoe **42** · fa on · ngl 999 · **KV f16** · depth 0 · runs **1** · no-warmup · endpoint `127.0.0.1:8080`.

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

- Needs the **atomic turboquant fork** — this is not a stock-llama.cpp model; budget the download before the run.
- Fits **3060 hybrid** with **ncmoe 42** (and 39 poke).
- Decode usable in the **~15–21 t/s** range after warm-up; not in the same class as Qwen TQ3 (~54 tg) or IQ3 Laguna (~23 tg).
- KV was **f16** here — try `-ctk q8_0 -ctv q8_0` next to buy context headroom; record peak VRAM + `free -h` under load.
- Next comparison: **`AD-IQ3_M`** (the card's 64 GB rung) vs this STOCK Q4_K_M.
- For chatty long runs, keep GGUF on **SSD** (Windows `krea2-model`), not GAMES1 HDD.

Snapshot: [../RESULTS.md](../RESULTS.md#ling-30-flash-q4_k_m). Related MoE offload: [../techniques/moe-offload.md](../techniques/moe-offload.md).
