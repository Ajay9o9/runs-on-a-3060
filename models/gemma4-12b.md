# Gemma 4 12B on RTX 3060

Dense-ish 12B-class model — **can fully land on 12GB** at Q5/Q6 with careful KV. Best “all VRAM, little RAM offload” LLM in this lab.

**Host:** RTX 3060 12GB · 9900X · **64 GB RAM** (RAM mostly idle for full-GPU configs).  
**KV (almost all benches below):** `-ctk q8_0 -ctv q8_0`

## Quants tried

| File | Size (reported) | Peak VRAM notes |
|------|-----------------|-----------------|
| `gemma-4-12b-it-UD-Q5_K_XL.gguf` | ~8.0–8.8 GiB | Full GPU friendly |
| `gemma-4-12b-it-UD-Q6_K_XL.gguf` | ~9.9 GiB | Peak ~**11261 MB**, no RAM spillover noted |
| `gemma-4-12b-it-Q8_0.gguf` | ~11.8 GiB | Needs partial `-ngl` (~40); peak ~**11171 MB** at ngl 40 |
| `gemma-4-12b-it-UD-Q4_K_XL.gguf` | Unsloth Q4 (non-QAT) | Long-context llama-benchy baseline |
| `gemma-4-12b-it-qat-UD-Q4_K_XL.gguf` | Unsloth **QAT** Q4 | Long context; with/without MTP — full section below |
| `gemma-4-12B-it-MTP-Q8_0.gguf` | MTP **draft** | Used as `--model-draft` with Q5 and QAT bases |

Vision: `mmproj-F16.gguf`.

---

## llama-bench — Q5 full GPU

**KV:** `q8_0` / `q8_0`

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q5_K_XL.gguf \
  -p 4096 -n 256 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| test | t/s |
|------|-----|
| pp4096 | **1152.06 ± 14.63** |
| tg256 | **33.27 ± 0.10** |

Later rebuild / size report variant:

| test | t/s |
|------|-----|
| pp4096 | 1069.79 ± 4.14 |
| tg256 | 30.59 ± 0.11 |

JSON metadata from bench included: `"gpu_info": "NVIDIA GeForce RTX 3060"`, `"cpu_info": "AMD Ryzen 9 9900X 12-Core Processor"`.

---

## llama-bench — Q6 full GPU

**KV:** `q8_0` / `q8_0`

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q6_K_XL.gguf \
  -p 4096 -n 256 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| test | t/s | Peak VRAM |
|------|-----|-----------|
| pp4096 | **1112.55 ± 3.27** | |
| tg256 | **26.03 ± 0.51** | **~11261 MB**, no RAM spillover |

---

## llama-bench — Q8 layer offload

Q8 weights ~11.78 GiB — full offload fights KV. Lab swept ngl:

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/gemma-4-12b-it-Q8_0.gguf \
  -ngl 40,45,48,50 \
  -p 4096 -n 256 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

At **ngl 40**, peak VRAM ~**11171 MB**:

| test | t/s |
|------|-----|
| pp4096 | 985.71 ± 15.83 |
| tg256 | **14.86 ± 0.08** |

(Q8 gen is much slower than Q5 on this card — prefer Q5 for interactive.)

---

## Server + vision + tools (Q5)

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q5_K_XL.gguf \
  --mmproj $MODEL_DIR/mmproj-F16.gguf \
  -c 64096 -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0 \
  --jinja --tools all
```

### llama-benchy (HTTP) at filled depths

Tool: [eugr/llama-benchy](https://github.com/eugr/llama-benchy)

```bash
llama-benchy \
  --base-url http://127.0.0.1:8080 \
  --model gemma-4-12b-it-UD-Q5_K_XL \
  --depth 0 4096 8192 16384 32768
```

Depth sweeps (pp2048 / tg32) stayed ~**29–32 tg** across depths in no-MTP runs; PP declines as depth grows (see raw vault for full tables).

### With MTP draft model

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-UD-Q5_K_XL.gguf \
  --mmproj $MODEL_DIR/mmproj-F16.gguf \
  --model-draft $MODEL_DIR/gemma-4-12B-it-MTP-Q8_0.gguf \
  -c 64096 -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0 \
  --jinja --spec-type draft-mtp --spec-draft-n-max 4
```

Benchy tg often landed **~33–43 t/s** depending on depth / sample (higher variance with speculation).

---

## Unsloth Q4 long context — non-QAT vs QAT, with/without MTP

Tool: [eugr/llama-benchy](https://github.com/eugr/llama-benchy) over `llama-server`.  
**KV (server):** `-ctk q8_0 -ctv q8_0`  
**Benchy:** `--pp 4096 --tg 256`, filled context depths (`d32768` = 32k already in context, etc.).  
**Threads in server log:** `-t 24` on these runs (not the usual `-t 12`).

### Server commands

Without MTP (base model only):

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-qat-UD-Q4_K_XL.gguf \
  --mmproj $MODEL_DIR/mmproj-F16.gguf \
  -c 131072 -b 512 -ub 512 -t 24 \
  -fa 1 -ctk q8_0 -ctv q8_0 \
  --jinja --reasoning-budget 1644
```

With MTP (draft model + draft-mtp):

```bash
# same server shape; lab used draft-mtp n-max 4 for the QAT+MTP table
# --model-draft $MODEL_DIR/gemma-4-12B-it-MTP-Q8_0.gguf
# --spec-type draft-mtp --spec-draft-n-max 4
```

```bash
llama-benchy \
  --base-url http://127.0.0.1:8080 \
  --model gemma-4-12b-it-qat-UD-Q4_K_XL \
  --pp 4096 --tg 256 \
  --depth 32768 65536 131072 \
  --no-warmup --runs 1
```

### A) Non-QAT `gemma-4-12b-it-UD-Q4_K_XL` — no MTP

Lab note: **~9980 MB VRAM**, **~7.5 GB RAM** (at 32k-class load).

| depth | pp4096 (t/s) | tg256 (t/s) | peak tg | VRAM note |
|------:|-------------:|------------:|--------:|-----------|
| 32k | 820.44 | **36.78** | 52 | |
| 64k | 686.97 | **27.86** | 37 | |
| 128k | 509.54 | **23.42** | 29 | |
| 256k | 343.91 | **17.43** | 22 | **~11376 MB** VRAM |

### B) QAT `gemma-4-12b-it-qat-UD-Q4_K_XL` — no MTP

Lab note: **~9380 MB VRAM**, **~7.8 GB RAM** (lighter than non-QAT at similar load).

Some intermediate rows in the raw log still printed the non-QAT model name in the benchy table; VRAM band matches QAT. Explicit QAT-named rows (2026-06-09):

| depth | pp4096 (t/s) | tg256 (t/s) | peak tg |
|------:|-------------:|------------:|--------:|
| 32k | 999.72 | **38.70** | 55 |
| 64k | 810.01 | **39.01** | 55 |

Additional depth sweep under the QAT section (model column in log may still say `UD-Q4_K_XL`):

| depth | pp4096 (t/s) | tg256 (t/s) | peak tg | VRAM note |
|------:|-------------:|------------:|--------:|-----------|
| 32k | 879.94 | **34.10** | 42 | ~9380 MB class |
| 64k | 722.33 | **30.60** | 38 | |
| 128k | 537.13 | **24.14** | 36 | |
| 256k | 356.40 | **18.19** | 22 | **~10784 MB** VRAM |

### C) QAT `gemma-4-12b-it-qat-UD-Q4_K_XL` — **with MTP** (`n-max` 4)

| depth | pp4096 (t/s) | tg256 (t/s) | peak tg |
|------:|-------------:|------------:|--------:|
| 32k | 1011.35 | **40.50** | 58 |
| 64k | 807.74 | **34.69** | 43 |

llama-benchy `0.3.8.dev2+gff162bcfc`, date **2026-06-09**.

### QAT / Q4 long-context comparison (same card)

| Setup | depth | pp4096 | tg256 | Notes |
|-------|------:|-------:|------:|-------|
| non-QAT Q4, no MTP | 32k | 820 | 37 | ~9980 MB VRAM |
| QAT Q4, no MTP | 32k | ~880–1000 | ~34–39 | ~9380 MB VRAM |
| **QAT Q4 + MTP n-max 4** | 32k | **1011** | **40.5** | best gen in this QAT set |
| non-QAT Q4, no MTP | 64k | 687 | 28 | |
| QAT Q4, no MTP | 64k | ~722–810 | ~31–39 | |
| **QAT Q4 + MTP n-max 4** | 64k | **808** | **34.7** | |

At deep context (128k–256k), prefill and gen both fall; QAT still tracks slightly better VRAM headroom than non-QAT at 256k in these logs.

---

## Summary table (this machine)

| Setup | KV | pp (typical) | tg (approx) | VRAM note |
|-------|-----|--------------|------------:|-----------|
| Q5_K_XL full GPU | q8_0 / q8_0 | pp4096 ~1070–1150 | ~30–33 | fits |
| Q6_K_XL full GPU | q8_0 / q8_0 | pp4096 ~1110 | ~26 | peak ~11.3 GB |
| Q8 ngl 40 | q8_0 / q8_0 | pp4096 ~986 | ~15 | peak ~11.2 GB |
| non-QAT Q4 @ 32k depth | q8_0 / q8_0 | pp4096 ~820 | ~37 | ~9980 MB + ~7.5 GB RAM |
| QAT Q4 @ 32k, no MTP | q8_0 / q8_0 | pp4096 ~880–1000 | ~34–39 | ~9380 MB + ~7.8 GB RAM |
| **QAT Q4 @ 32k + MTP n-max 4** | q8_0 / q8_0 | pp4096 ~**1011** | ~**40.5** | same QAT stack |
| Q5 + mmproj-F16 | q8_0 / q8_0 | — | — | vision server |
