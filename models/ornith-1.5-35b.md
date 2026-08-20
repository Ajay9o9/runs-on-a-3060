# Ornith 1.5 35B-A3B on RTX 3060

**Ornith-1.5-35B-A3B** MoE (~35B total / ~3B active, `qwen35moe`). Hybrid only on 12GB: **`-ngl 999` + `-ncmoe`**.  
Lab dates: **2026-08-20**. Host is **64 GB**; smaller RAM SKUs were enforced with cgroup `memory.max` on `llama-server` (`MemorySwapMax=0`, **4 GiB OS reserve** so 16 GB → **12 GiB** process cap, 8 GB → **5.5 GiB**).

**Runtime:** [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) **b10498** (`5ecbe1ac1`).  
**Bench:** [llama-benchy](https://github.com/eugr/llama-benchy) `0.3.8.dev2+gff162bcfc` — **`--pp 0 --tg 256 --runs 1 --no-warmup`**, depths **8k / 16k / 32k / 64k / 112640**. These rows are **decode vs context depth**, not prefill.  
**Always:** `-ngl 999` · `-fa on` · `-t 12` · `-np 1` · `--jinja` · **`-c 131072`**.  
KV catalog: [../techniques/kv-cache.md](../techniques/kv-cache.md) · offload: [../techniques/moe-offload.md](../techniques/moe-offload.md).

Tokenizer for benchy: `ornith-ai/Ornith-1.5-35B-A3B`. No quality eval in this session.

**Report:** [ajay9o9.github.io/runs-on-a-3060/reports/ornith-1.5-35b](https://ajay9o9.github.io/runs-on-a-3060/reports/ornith-1.5-35b/)

## Model / files

| Item | Value |
|------|--------|
| Arch | qwen35moe · 256 experts / 8 active · Q6/Q4_K_M: **41** blocks · AD mixed: **40** blocks |
| Lab dir | `/media/aj-homeserver/windows/krea2-model/ornith/` |
| Hardware | RTX 3060 12GB · Ryzen 9 9900X · 64 GB RAM · Ubuntu 24.04 |

HF: [ornith-ai/Ornith-1.5-35B-A3B](https://huggingface.co/ornith-ai/Ornith-1.5-35B-A3B) · AD GGUFs [AtomicChat/Ornith-1.5-35B-A3B-GGUF](https://huggingface.co/AtomicChat/Ornith-1.5-35B-A3B-GGUF)

### Quants tried

| File | Disk | Kind |
|------|------|------|
| `Ornith-1.5-35B-Q6_K.gguf` | **~28 GB** | Q6_K |
| `Ornith-1.5-35B-Q4_K_M.gguf` | **~20.2 GB** | Q4_K_M |
| `Ornith-1.5-35B-A3B-AD-Q4_K-IQ4_XS.gguf` | **~18.7 GB** | AtomicChat AD mixed |
| `Ornith-1.5-35B-A3B-AD-IQ4_XS-IQ3_S.gguf` | **~16.4 GB** | AtomicChat AD mixed |

## Headline (what to run)

All **KV q8_0 / q8_0** unless noted. **tg** = llama-benchy tg256 at that **depth**. Peak RSS is the host-RAM number that transfers to smaller PCs (not this box’s page cache).

| Quant | RAM floor | `-ncmoe` | KV | VRAM (load) | peak RSS | tg @8k | tg @32k | tg @112k |
|-------|-----------|----------|-----|-------------|----------|-------:|--------:|---------:|
| **IQ4_XS-IQ3_S** | **16 GB** | **22** | q8_0 / q8_0 | ~10.8 GB | ~11.8 GB | **56.4** | **47.7** | **30.5** |
| **Q4_K_M** | **24 GB** | **24** | q8_0 / q8_0 | ~11.2 GB | ~14.8 GB | **55.4** | **47.6** | **30.6** |
| **Q4_K-IQ4_XS** | **24 GB** | **25** | q8_0 / q8_0 | ~10.6 GB | ~15.0 GB | **53.8** | **44.0** | **29.1** |
| **Q6_K** | **32 GB** | **28** | q8_0 / q8_0 | ~11.4 GB | ~21.8 GB | **47.7** | **41.2** | **27.6** |

**8 GB:** no usable 131k recipe.  
**16 GB:** only **IQ4_XS-IQ3_S** completed **112k**. Q4_K_M (q8 or q4 KV) and Q4_K-IQ4_XS died before 112k on a 12 GiB process cap.  
Lowest `-ncmoe` that **loads** on 12GB at `-c 131072` (more GPU experts = less host RAM, more VRAM): Q6 **28** · Q4_K_M **24** · Q4_K-IQ4_XS **23** · IQ4_XS-IQ3_S **20** (20 GPU-OOM on later retries; **22** is the stable recipe).

`-c 131072` reserved on every start. Benchy max depth **112640**. Did **not** search a higher `-c`.

---

## Server (recipes)

```bash
# 16 GB RAM — IQ4_XS-IQ3_S
./build/bin/llama-server \
  -m $MODEL_DIR/Ornith-1.5-35B-A3B-AD-IQ4_XS-IQ3_S.gguf \
  -ngl 999 -ncmoe 22 -fa on --jinja -np 1 -t 12 \
  --cache-type-k q8_0 --cache-type-v q8_0 -c 131072 \
  --host 127.0.0.1 --port 8080
```

```bash
# 24 GB RAM — Q4_K_M
./build/bin/llama-server \
  -m $MODEL_DIR/Ornith-1.5-35B-Q4_K_M.gguf \
  -ngl 999 -ncmoe 24 -fa on --jinja -np 1 -t 12 \
  --cache-type-k q8_0 --cache-type-v q8_0 -c 131072 \
  --host 127.0.0.1 --port 8080
```

```bash
# 24 GB RAM — Q4_K-IQ4_XS
./build/bin/llama-server \
  -m $MODEL_DIR/Ornith-1.5-35B-A3B-AD-Q4_K-IQ4_XS.gguf \
  -ngl 999 -ncmoe 25 -fa on --jinja -np 1 -t 12 \
  --cache-type-k q8_0 --cache-type-v q8_0 -c 131072 \
  --host 127.0.0.1 --port 8080
```

```bash
# 32 GB RAM — Q6_K
./build/bin/llama-server \
  -m $MODEL_DIR/Ornith-1.5-35B-Q6_K.gguf \
  -ngl 999 -ncmoe 28 -fa on --jinja -np 1 -t 12 \
  --cache-type-k q8_0 --cache-type-v q8_0 -c 131072 \
  --host 127.0.0.1 --port 8080
```

| Flag | Note |
|------|------|
| `-ngl 999` | all non-overridden tensors on GPU (fixed for this whole sweep) |
| `-ncmoe N` | MoE weights of the first N layers on CPU |
| `-c 131072` | KV reserved for 131k (benchy used 112640 depth + 256 gen) |
| `-ctk` / `-ctv` | **q8_0 / q8_0** on recipes; q4_0 poke below |

## llama-benchy

```bash
llama-benchy \
  --base-url http://127.0.0.1:8080 \
  --model <served-alias> \
  --tokenizer ornith-ai/Ornith-1.5-35B-A3B \
  --tg 256 --pp 0 \
  --depth 8192 16384 32768 65536 112640 \
  --no-warmup --runs 1 --skip-coherence --latency-mode none
```

**Config stamp:** ngl **999** · fa on · t 12 · np 1 · **`-c 131072`** · runs **1** · no-warmup · endpoint `127.0.0.1:8080`.

---

## Results — IQ4_XS-IQ3_S (KV q8_0 / q8_0)

Min GPU-fit `-ncmoe` **20** (flaky); recipe **22**.

| ncmoe | RAM cap | VRAM | peak RSS | 8k | 16k | 32k | 64k | 112k |
|------:|--------:|-----:|---------:|---:|----:|----:|----:|-----:|
| **22** | **16** | 10.8 | 11.8 | **56.4** | 52.3 | **47.7** | 37.8 | **30.5** |
| 25 | 16 | 9.6 | 12.0 | 52.4 | 51.6 | 44.5 | 36.0 | 29.4 |
| 30 | 24 | 7.9 | 15.4 | 52.2 | 50.3 | 44.3 | 35.7 | 27.6 |
| 35 | 24 | 6.3 | 17.4 | 50.8 | 47.3 | 42.6 | 35.7 | 28.3 |
| 41 | 24 | 4.7 | 19.3 | 40.1 | 42.9 | 41.5 | 32.6 | 28.3 |

## Results — Q4_K_M (KV q8_0 / q8_0)

Min GPU-fit `-ncmoe` **24**.

| ncmoe | RAM cap | VRAM | peak RSS | 8k | 16k | 32k | 64k | 112k |
|------:|--------:|-----:|---------:|---:|----:|----:|----:|-----:|
| **24** | **24** | 11.2 | 14.8 | **55.4** | 48.1 | **47.6** | 39.4 | **30.6** |
| 26 | 24 | 10.3 | 15.7 | 52.1 | 53.7 | 47.6 | 38.9 | 30.4 |
| 29 | 24 | 9.0 | 17.1 | 52.3 | 52.8 | 46.7 | 37.9 | 29.9 |
| 30 | 24 | 8.6 | 17.5 | 48.4 | 51.9 | 45.7 | 37.1 | 29.3 |
| 35 | 24 | 6.3 | 19.9 | 51.2 | 49.8 | 39.7 | 27.2 | 28.8 |
| 41 | 32 | 3.9 | 22.7 | 49.8 | 47.9 | 42.9 | 35.6 | 28.5 |

### Q4_K_M + KV **q4_0 / q4_0** (16 GB poke)

Same `-ngl 999 -c 131072`. q4 KV frees VRAM → min `-ncmoe` **22–23**.

| ncmoe | RAM cap | result |
|------:|--------:|--------|
| 23 | 16 GB (12 GiB cap) | **fail** — 8k ~41 t/s then server died at 16k |
| 23 | 24 GB | full 112k: 56.3 / 54.3 / 47.1 / 36.4 / **27.0** · peak RSS ~13.7 GB |

q4 KV does **not** make Q4_K_M a 16 GB 131k recipe.

## Results — Q4_K-IQ4_XS (KV q8_0 / q8_0)

Min GPU-fit `-ncmoe` **23**. Recipe for full 112k: **25** on **24 GB**.  
**ncmoe 41** load started (RSS ~18.8 GB, VRAM ~4.5 GB); **no full bench** (host reboot).

| ncmoe | RAM cap | VRAM | peak RSS | 8k | 16k | 32k | 64k | 112k |
|------:|--------:|-----:|---------:|---:|----:|----:|----:|-----:|
| 23 | 16 | 11.4 | ~11.9 | 50.8 | 50.8 | 45.3 | 29.1 | **died** |
| 23 | 8 | 11.5 | — | 10.0 | 7.1 | 5.6 | 4.4 | **no 112k** (thrash) |
| **25** | **24** | 10.6 | 15.0 | **53.8** | 49.4 | **44.0** | 35.8 | **29.1** |
| 28 | 24 | 9.4 | 16.3 | 52.0 | 48.9 | 43.3 | 36.1 | 28.5 |
| 30 | 24 | 8.6 | 17.2 | 51.5 | 48.8 | 43.2 | 35.4 | 28.4 |
| 35 | 24 | 6.5 | 19.4 | 49.4 | 47.0 | 42.1 | 32.8 | 23.4 |

## Results — Q6_K (KV q8_0 / q8_0)

Min GPU-fit `-ncmoe` **28**. 16/24 GB: peak RSS ~22 GB → **no**.

| ncmoe | RAM cap | VRAM | peak RSS | 8k | 16k | 32k | 64k | 112k |
|------:|--------:|-----:|---------:|---:|----:|----:|----:|-----:|
| **28** | **32** | 11.4 | 21.8 | **47.7** | 46.9 | **41.2** | 34.6 | **27.6** |
| 30 | 32 | 10.2 | 23.1 | 47.8 | 45.6 | 41.1 | 34.2 | 27.6 |
| 33 | 32 | 8.3 | 25.0 | 46.4 | 44.0 | 39.7 | 33.6 | 27.1 |
| 35 | 32 | 7.1 | 26.3 | 45.2 | 42.8 | 39.5 | 32.7 | 26.8 |
| 41 | 64 | 4.0 | 29.5 | 42.9 | 41.0 | 37.6 | 31.8 | 26.1 |

## Lab takeaways

- **`-ngl 999` is the whole sweep** — only `-ncmoe` (and once KV type) changed.
- **16 GB + 3060 + 131k:** **IQ4_XS-IQ3_S `-ncmoe 22` q8 KV**. Not Q4_K_M, not q4 KV on Q4_K_M, not Q4_K-IQ4_XS at 112k.
- **24 GB:** Q4_K_M `-ncmoe 24` or Q4_K-IQ4_XS `-ncmoe 25` (q8 KV).
- **32 GB:** Q6_K `-ncmoe 28`.
- Hybrid mmap: host **Cached** on 64 GB is the GGUF in page cache; **peak RSS** is what a 16/24/32 GB box must hold.
- Single-run benchy; no ± spread; no quality score.
