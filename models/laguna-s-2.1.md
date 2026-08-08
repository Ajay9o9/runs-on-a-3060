# Laguna S-2.1 (118B-A8B MoE) on RTX 3060

**118B** total MoE, **~8B** active / token (256 routed top-10 + 1 shared). Weights **do not** fit in 12GB; all runs are **hybrid** (`-ngl 999` + `--n-cpu-moe`). Lab RAM: **64 GB** (experts mmap on host — Q4 is tight).

**Runtime:** [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) master (Laguna support PR #25165); lab binary under `krea2-model/llama.cpp-laguna`.  
**Bench tools:** [llama-benchy](https://github.com/eugr/llama-benchy) (IQ3_S) · `llama-server` slot timings (Q4_K_M).  
**Always log KV** (`-ctk` / `-ctv`). Catalog: [../techniques/kv-cache.md](../techniques/kv-cache.md) · offload: [../techniques/moe-offload.md](../techniques/moe-offload.md).

Lab dates: **2026-07-23** (IQ3_S + Q4 ncmoe 46) · **2026-07-24** (Q4 ncmoe 44 + q8).  
**Not** the 3090 daily recipe — 3090 uses lower `ncmoe` (more GPU experts).

## Quants tried

| File | Approx size | Notes |
|------|-------------|--------|
| `Laguna-S-2.1-UD-IQ3_S.gguf` | **~46 GB** | single file; best **speed** hybrid on this box |
| `Laguna-S-2.1-UD-Q4_K_M-0000x-of-00003.gguf` | **~73.1 GB** (3 shards) | pass **shard 1** only; 2–3 load from same dir |

### Q4_K_M shard layout (lab)

| Shard | Role | Approx size |
|------:|------|-------------|
| 00001-of-00003 | header / entry GGUF | ~3.7 MB |
| 00002-of-00003 | weights | ~49.93 GB |
| 00003-of-00003 | weights | ~23.18 GB |

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

### Why ncmoe 44 on 3060

12GB is tighter than the 3090 daily recipe (`ncmoe 32`). Higher `ncmoe` → more routed-expert layers on **CPU**. With **48** layers and **`--n-cpu-moe 44`** + **`-ngl 999`**:

| | Layers | Count |
|--|--------|------:|
| Dense lead | `blk.0` | 1 (GPU FFN, no routed exps) |
| **CPU routed experts** | first 44 layer indices (`blk.1`…`blk.43` area) | **~43** MoE layers |
| **GPU routed experts** | `blk.44`…`blk.47` | **~4** MoE layers |
| Attn + shared expert | all layers | **GPU** |

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

### llama-benchy command

Server may expose a local path — pass HF id for tokenizer + served name:

```bash
llama-benchy \
  --base-url http://127.0.0.1:8080 \
  --model poolside/Laguna-S-2.1 \
  --served-model-name "$MODEL_DIR/Laguna-S-2.1-UD-IQ3_S.gguf" \
  --tg 256 \
  --pp 2048 4096 8192 16384 28672 \
  --depth 0 \
  --no-warmup \
  --runs 3 \
  --no-cache
```

Avoid `pp + tg > 32536` (e.g. raw `pp 32536` + `tg 256`). Near ceiling use **`28672`** or **`32000`**.

### llama-benchy (depth 0, tg 256, runs 3)

| test | t/s | peak t/s | ttfr / est_ppt (ms) |
|------|----:|---------:|--------------------:|
| pp2048 | 186.22 ± 0.28 | | ~11.0k |
| tg256 | 24.95 ± 0.28 | 26.00 ± 0.00 | |
| pp4096 | 190.74 ± 7.95 | | ~21.5k |
| tg256 | 25.07 ± 0.29 | 25.67 ± 0.47 | |
| pp8192 | 203.06 ± 1.50 | | ~40.3k |
| tg256 | 24.89 ± 0.02 | 26.00 ± 0.00 | |
| pp16384 | 202.82 ± 0.93 | | ~80.8k |
| tg256 | 24.00 ± 0.20 | 25.00 ± 0.00 | |
| **pp28672** | **201.78 ± 0.73** | | ~142.1k |
| **tg256** | **23.16 ± 0.08** | 24.00 ± 0.00 | |

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

### Load log (excerpt)

```
load_model: loading model '.../Laguna-S-2.1-UD-Q4_K_M-00001-of-00003.gguf'
W: tensor overrides to CPU are used with mmap enabled — consider using --no-mmap
load_model: initializing, n_slots = 1, n_ctx_slot = 164352, kv_unified = 'false'
model loaded
listening on http://127.0.0.1:8080
```

No `-c` → auto fit reported **`n_ctx_slot = 164352`** (~164k) with **q4_0** KV.  
That is **what fits** (VRAM + KV + hybrid), not the model card max (1M). Pin later with e.g. `-c 65536` / `-c 131072`.  
**VRAM:** ~**10.3–10.7 GB**.  
**Host:** ~73 GB weights on ~64 GB RAM → mmap pressure; prefer mmap (full `--no-mmap` likely OOM).

Tokenizer warnings (`special_eos_id` / `special_eot_id` not in `special_eog_ids`) — config noise unless EOGs misbehave in chat.

### Offload sketch (`ncmoe 46`, 48 layers)

| | Layers | Count |
|--|--------|------:|
| Dense lead | `blk.0` | 1 |
| **CPU routed experts** | first 46 layer indices | most MoE bulk |
| **GPU routed experts** | last few (`blk.46`…`blk.47`) | **~2** MoE layers |
| Attn + shared | all | GPU |

### Context map (same card / RAM)

| ncmoe | KV | Auto context | Prefill | Decode | Notes |
|------:|-----|-------------:|--------:|-------:|-------|
| **46** | **q4_0** | **~164k** | **~51–57 t/s** | **~11–12 t/s** long | ~9–10.5 t/s after ~7k PP |
| **46** | **q8_0** | **~80k** claimed | — | — | **OOM under load** — no usable speed run |
| **44** | **q8_0** | **4k** | **~24–27 t/s** | **~13–14 t/s** | tiny TG bump; context dies |

Dropping ncmoe **46→44** with q8 puts more experts on GPU → **KV headroom collapses** (~80k → 4k). Horizontal offload is mainly a **context** knob at this size.

### Server timings — ncmoe 46, q4_0 (not benchy)

Source: `llama-server` `slot print_timing` (interactive).

**Prefill (task ~39067, LCP reuse):**

| n_tokens | time | PP t/s |
|---------:|-----:|-------:|
| 2048 | 38.29 s | **53.49** |
| 4096 | 80.60 s | **50.82** |
| 6144 | 110.18 s | **55.76** |
| 6911 | 121.88 s | **56.70** |
| 7413 | 131.40 s | **56.41** |
| 7423 | 132.95 s | **55.83** |

→ Prefill **~51–57 t/s** (much slower than IQ3_S benchy ~186–205 — heavier weights + RAM pressure).

**Long decode (task ~36894, warm / mid-long ctx):**

| n_decoded | tg t/s | tg_3s |
|----------:|-------:|------:|
| 1048 | 10.93 | 12.76 |
| 1206 | 11.16 | 13.51 |
| 1568 | 11.58 | 14.11 |
| 2009 | 11.90 | 13.18 |
| 2132 | 11.98 | 13.53 |

Steady **~11–12 t/s** cumulative; rolling **~13–14 t/s**. Stop sample: `n_tokens = 27350`.

**Decode right after ~7k prefill:**

| n_decoded | tg t/s | tg_3s |
|----------:|-------:|------:|
| 100 | 9.04 | 9.04 |
| 163 | 9.47 | 10.60 |
| 270 | 10.25 | 12.80 |
| 341 | 10.50 | 10.78 |
| 507 | 10.29 | 12.78 |

→ Early post-prefill decode **~9–10.5 t/s** (a bit below long warm TG).

### Server timings — ncmoe 44, q8_0 (4k only, 2026-07-24)

```
load_model: initializing, n_slots = 1, n_ctx_slot = 4096, kv_unified = 'false'
prompt eval: 2631 tok → 26.58 t/s
eval:        1465 tok → 14.33 t/s
total:       4096 tok (truncated=1)
```

| n_tokens | PP t/s |
|---------:|-------:|
| 2048 | 24.30 |
| 2115 | 24.16 |
| 2627 | 26.72 |
| final 2631 | **26.58** |

Decode climbed **~12.8 → ~14.3 t/s** (tg_3s often ~15).

### Lab takeaway (Q4_K_M)

| Knob | Verdict |
|------|---------|
| **ncmoe 46 + q4_0** | Best long-ctx daily on 3060 for this quant (~164k, TG ~11–12) |
| **ncmoe 46 + q8_0** | Fit may show **~80k**, but **OOM** under load — no reliable speeds |
| **ncmoe 44 + q8_0** | **4k** — avoid; TG bump useless vs context death |

### Tweet-ready block

```text
Same machine, same RAM, bigger model (Laguna S-2.1 UD-Q4_K_M ~73GB)

-ngl 999 --n-cpu-moe 46
VRAM ~10.7 GB on a 3060

q4_0 KV → ~164k context
  prefill ~51–57 t/s · decode ~11–12 t/s

q8_0 KV → ~80k claimed, but OOM under load — no clean PP/TG

--n-cpu-moe 44 + q8_0 → only 4k context
  prefill ~24–27 t/s · decode ~13–14 t/s
  tiny TG bump, context dies — not worth it
```

---

## Compare IQ3_S vs Q4_K_M (same GPU)

| | IQ3_S | Q4_K_M A | Q4_K_M B |
|--|------:|---------:|---------:|
| Weights | ~46 GB | ~73 GB (3 shards) | same |
| ncmoe | 44 | **46** | **44** |
| KV | f16 / q8_0 | **q4_0** | **q8_0** |
| Context | 32k f16; 64k q8 | **auto ~164k** | **auto 4k** |
| VRAM | ~9–10.3 GB | ~10.3–10.7 GB | q8 steals ctx |
| PP (order) | ~186–205 (benchy) | **~51–57** (server) | **~24–27** (server, 4k) |
| TG (order) | ~19–25 | **~11–12** long | **~13–14** @ 4k |

---

## Summary

| Quant | Best daily angle on 3060 |
|-------|--------------------------|
| **IQ3_S** | Faster hybrid (~20–25 tg @ short–mid ctx; ~14 tg after 64k PP) |
| **Q4_K_M** | Heavier / slower (~11–12 tg) but **q4_0 + ncmoe 46** can auto-fit **~164k** ctx |

System RAM holds most experts (~46 GB IQ3 / ~73 GB Q4). Hybrid MoE is required either way.

Snapshot: [../RESULTS.md](../RESULTS.md#laguna-s-21-ud-iq3_s--ud-q4_k_m).
