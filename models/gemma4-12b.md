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
| `gemma-4-12b-it-qat-UD-Q4_K_XL.gguf` | QAT Q4 | Long context via llama-benchy; ~**9980 MB** VRAM + ~**7.5 GB** RAM in one server run |
| `gemma-4-12B-it-MTP-Q8_0.gguf` | MTP **draft** | Used as `--model-draft` with Q5 base |

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

## QAT Q4_K_XL — long context (llama-benchy)

```bash
# server with QAT quant + mmproj
./build/bin/llama-server \
  -m $MODEL_DIR/gemma-4-12b-it-qat-UD-Q4_K_XL.gguf \
  --mmproj $MODEL_DIR/mmproj-F16.gguf \
  ...

llama-benchy \
  --base-url http://127.0.0.1:8080 \
  --model gemma-4-12b-it-qat-UD-Q4_K_XL \
  --pp 4096 --tg 256 \
  --depth 32768 65536 131072
```

Lab snapshot: **~9980 MB VRAM**, **~7.5 GB RAM**.  
Example: `pp4096 @ d32768` ≈ **1011 t/s**, `tg256` ≈ **40.5 t/s** (single-run style rows in notes).

---

## Summary table (this machine)

| Setup | KV | tg (approx) | VRAM note |
|-------|-----|------------:|-----------|
| Q5_K_XL full GPU | q8_0 / q8_0 | ~30–33 | fits |
| Q6_K_XL full GPU | q8_0 / q8_0 | ~26 | peak ~11.3 GB |
| Q8 ngl 40 | q8_0 / q8_0 | ~15 | peak ~11.2 GB |
| QAT Q4 long ctx (benchy) | q8_0 / q8_0 (server log) | ~35–40 (varies) | ~9980 MB VRAM, ~7.5 GB RAM in one server log |
| Q5 + mmproj-F16 | q8_0 / q8_0 | — | vision server config above |
