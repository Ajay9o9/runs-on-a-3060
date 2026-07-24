# Laguna S-2.1 (118B-A8B MoE) on RTX 3060

**118B** total MoE, **~8B** active / token (256 routed top-10 + 1 shared). Weights **do not** fit in 12GB; all runs are **hybrid** (`-ngl 999` + `--n-cpu-moe`). Lab RAM: **64 GB** (experts mmap on host — Q4 is tight).

**Runtime:** [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) master (Laguna support PR #25165).  
**Bench tools:** [llama-benchy](https://github.com/eugr/llama-benchy) (IQ3_S) · `llama-server` slot timings (Q4_K_M).  
**Always log KV** (`-ctk` / `-ctv`). Catalog: [../techniques/kv-cache.md](../techniques/kv-cache.md) · offload: [../techniques/moe-offload.md](../techniques/moe-offload.md).

## Quants tried

| File | Approx size | Notes |
|------|-------------|--------|
| `Laguna-S-2.1-UD-IQ3_S.gguf` | **~46 GB** | single file; best **speed** hybrid on this box |
| `Laguna-S-2.1-UD-Q4_K_M-0000x-of-00003.gguf` | **~73.1 GB** (3 shards) | pass **shard 1** only; 2–3 load from same dir |

HF: [unsloth/Laguna-S-2.1-GGUF](https://huggingface.co/unsloth/Laguna-S-2.1-GGUF) · base [poolside/Laguna-S-2.1](https://huggingface.co/poolside/Laguna-S-2.1)

---

## Headline results

| Quant | Offload | KV | Context (auto / set) | pp | tg | VRAM |
|-------|---------|-----|----------------------|----|----|------|
| **UD-IQ3_S** | ngl 999, **ncmoe 44** | **f16 / f16** | `-c 32536` (~32k max) | pp28672 **~202** | tg256 **~23** | **~10.1 GB** |
| **UD-IQ3_S** | same | **q8_0 / q8_0** | `-c 66536` | pp32768 **~197** · pp65536 **~183** | tg256 **~19.4** · **~14.3** | **~10.3 GB @ 64k** |
| **UD-Q4_K_M** | ngl 999, **ncmoe 46** | **q4_0 / q4_0** | auto **~164k** | server **~51–57** | long **~11–12** | **~10.7 GB** |
| **UD-Q4_K_M** | ncmoe **46** | **q8_0 / q8_0** | auto **~80k** claimed | — | — | **OOM under load** (no clean speed sample) |
| **UD-Q4_K_M** | ncmoe **44** | **q8_0 / q8_0** | auto **4k** | server **~24–27** | **~13–14** @ 4k | tiny TG bump; ctx dies |

Do not mix IQ3_S **benchy** rows with Q4 **server-log** rows as if they were the same methodology.

---

## A) UD-IQ3_S — f16 KV (~32k)

### Server

```bash
./build/bin/llama-server \
  --model $MODEL_DIR/Laguna-S-2.1-UD-IQ3_S.gguf \
  --jinja -fa on \
  -ngl 999 --n-cpu-moe 44 \
  -c 32536 \
  -b 2048 -ub 512 -t 12 \
  --host 127.0.0.1 --port 8080 \
  -np 1
```

**KV:** default **f16 / f16** (no `-ctk`/`-ctv`).  
**VRAM:** ~**10.1 GB**.

### llama-benchy (depth 0, tg 256, runs 3)

May need `--model poolside/Laguna-S-2.1` + `--served-model-name` path if auto-detect sees a local GGUF path; use `--skip-coherence` if the Paris check fails.

| test | t/s | peak t/s |
|------|----:|---------:|
| pp2048 | 186.22 ± 0.28 | |
| tg256 | 24.95 ± 0.28 | 26.00 ± 0.00 |
| pp4096 | 190.74 ± 7.95 | |
| tg256 | 25.07 ± 0.29 | 25.67 ± 0.47 |
| pp8192 | 203.06 ± 1.50 | |
| tg256 | 24.89 ± 0.02 | 26.00 ± 0.00 |
| pp16384 | 202.82 ± 0.93 | |
| tg256 | 24.00 ± 0.20 | 25.00 ± 0.00 |
| **pp28672** | **201.78 ± 0.73** | |
| **tg256** | **23.16 ± 0.08** | 24.00 ± 0.00 |

---

## B) UD-IQ3_S — q8_0 KV (up to 64k)

### Server

```bash
./build/bin/llama-server \
  --model $MODEL_DIR/Laguna-S-2.1-UD-IQ3_S.gguf \
  --jinja -fa on \
  -ngl 999 --n-cpu-moe 44 \
  -c 66536 -ctk q8_0 -ctv q8_0 \
  -b 2048 -ub 512 -t 12 \
  --host 127.0.0.1 --port 8080 \
  -np 1
```

**KV:** **q8_0 / q8_0**.  
**VRAM @ 64k:** ~**10.3 GB**.

### llama-benchy matrix (2k–28k, tg 256)

| test | t/s | peak t/s |
|------|----:|---------:|
| pp2048 | 186.51 ± 1.87 | |
| tg256 | 24.30 ± 0.09 | 25.00 ± 0.00 |
| pp4096 | 195.78 ± 2.76 | |
| tg256 | 24.05 ± 0.06 | 25.00 ± 0.00 |
| pp8192 | 202.42 ± 1.07 | |
| tg256 | 23.17 ± 0.13 | 24.00 ± 0.00 |
| pp16384 | 203.90 ± 1.04 | |
| tg256 | 21.87 ± 0.03 | 23.00 ± 0.00 |
| pp28672 | 205.49 ± 0.93 | |
| tg256 | 20.72 ± 0.02 | 21.00 ± 0.00 |

### Long-context points

| test | t/s | peak t/s |
|------|----:|---------:|
| **pp32768** | **197.01 ± 0.29** | |
| **tg256** | **19.38 ± 0.42** | 20.27 ± 0.39 |
| **pp65536** | **183.43 ± 0.00** | |
| **tg256** | **14.27 ± 0.00** | 15.00 ± 0.00 |

---

## C) UD-Q4_K_M — hybrid (3 shards)

Serve **shard 1**; place all three files in one directory.

```bash
./build/bin/llama-server \
  --model $MODEL_DIR/UD-Q4_K_M/Laguna-S-2.1-UD-Q4_K_M-00001-of-00003.gguf \
  --jinja -fa on \
  -ctk q4_0 -ctv q4_0 \
  -ngl 999 --n-cpu-moe 46 \
  -np 1
```

No `-c` → auto fit reported **`n_ctx_slot = 164352`** (~164k) with **q4_0** KV.  
**VRAM:** ~**10.7 GB**.  
**Host:** ~73 GB weights on ~64 GB RAM → mmap pressure; prefer mmap (full `--no-mmap` likely OOM).

### Context map (same card / RAM)

| ncmoe | KV | Auto context | Prefill | Decode | Notes |
|------:|-----|-------------:|--------:|-------:|-------|
| **46** | **q4_0** | **~164k** | **~51–57 t/s** | **~11–12 t/s** long | ~9–10.5 t/s after ~7k PP |
| **46** | **q8_0** | **~80k** claimed | — | — | **OOM under load** — no usable speed run |
| **44** | **q8_0** | **4k** | **~24–27 t/s** | **~13–14 t/s** | tiny TG bump; context dies |

### Server timings — ncmoe 46, q4_0 (not benchy)

Prefill sample:

| n_tokens | pp t/s |
|---------:|-------:|
| 2048 | 53.49 |
| 4096 | 50.82 |
| 6144 | 55.76 |
| 7423 | 55.83 |

Long decode sample (n_decoded ~1k–2k, slot ~27k tokens): **tg ~11–12 t/s** (tg_3s often ~13–14).

### Server timings — ncmoe 44, q8_0 (4k only)

```
n_ctx_slot = 4096
prompt eval: 2631 tok → 26.58 t/s
eval:        1465 tok → 14.33 t/s
truncated = 1
```

Prefill ~2k–2.6k: **~24–27 t/s**. Decode climbs **~12.8 → ~14.3 t/s**.

**Lab takeaway:** for Q4_K_M on 3060, prefer **ncmoe 46 + q4_0** if you want long context. **ncmoe 44 + q8** is not a useful trade.

---

## Summary

| Quant | Best daily angle on 3060 |
|-------|--------------------------|
| **IQ3_S** | Faster hybrid (~20–25 tg @ short–mid ctx; ~14 tg after 64k PP) |
| **Q4_K_M** | Heavier / slower (~11–12 tg) but **q4_0 + ncmoe 46** can auto-fit **~164k** ctx |

System RAM holds most experts (~46 GB IQ3 / ~73 GB Q4). Hybrid MoE is required either way.
