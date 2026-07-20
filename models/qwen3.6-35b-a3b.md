# Qwen3.6 35B-A3B (MoE) on RTX 3060

**Primary workhorse of this lab.** 34.66B params MoE (~3B active class). Weights usually **far larger than 12GB**, so almost everything is **hybrid GPU+CPU** using **64 GB system RAM**.

## Quants tried

| File | Approx size | Runtime that mattered |
|------|-------------|------------------------|
| `Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` | ~20.8 GiB | TurboQuant branch (`turbo4`/`turbo3` KV) |
| `Qwen3.6-35B-A3B-UD-Q5_K_M.gguf` | ~24.6 GiB | ik_llama + mainline |
| `Qwen3.6-35B-A3B-UD-Q6_K.gguf` | ~27.3 GiB | **ik_llama** preferred in lab |
| `Qwen3.6-35B-A3B-TQ3_4S.gguf` | ~12.4 GiB | **llama.cpp-tq3** |
| `Qwen3.6-35BA3B-MTP.gguf` | MTP | MTP speculative branch |
| `Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf` | ~29–30 GiB class | MTP + hybrid offload |

Vision projector used in some server runs: `mmproj-BF16.gguf`.

---

## Headline results (generation)

| Setup | tg t/s | pp note | VRAM / RAM story |
|-------|--------|---------|------------------|
| TQ3_4S, tq3 fork, ncmoe 16–24, 128k | **~58–60** | pp128k ~500–530 | Hybrid; RAM holds experts |
| TQ3_4S, 4k, ncmoe 32 | **~53–54** | pp4k ~600+ | Hybrid |
| Q5, mainline, ncmoe 32 | **~48** | pp4k ~500 | Experts on CPU → **many GB RAM** |
| Q6, ik_llama, ncmoe 32 | **~47** | pp4k ~800 | Same |
| Q4 TurboQuant, ncmoe 48 | **~43** | pp4k ~480 | highest tg in that ngl99 sweep |
| Q4 TurboQuant, ngl 20 vertical | **~30** | pp4k ~560 | Fewer layers on GPU |
| MTP on, n-max ~3 | **~31–36** vs ~21 off | — | +~2–2.5 GB VRAM for drafts |

Full fork tables: [../runtimes/comparison.md](../runtimes/comparison.md).

---

## Command recipes

### A) Fastest gen path — TQ3 (llama.cpp-tq3)

```bash
# Build: https://github.com/turbo-tan/llama.cpp-tq3
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv tq3_0
```

**Results (lab):** pp4096 ≈ **619 t/s**, tg128 ≈ **54 t/s**.

128k stress:

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -p 131072 -n 128 \
  -ngl 99 -ncmoe 16 \
  -b 4096 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv tq3_0
```

**Results:** pp131072 ≈ **532 t/s**, tg128 ≈ **60 t/s**.

Server-style (vision projector optional):

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  --mmproj $MODEL_DIR/mmproj-BF16.gguf \
  -ngl 99 -c 4096 -np 1 \
  -ctk q4_0 -ctv tq3_0 -fa on \
  --jinja --no-mmproj-offload \
  --reasoning off --reasoning-budget 0 --reasoning-format deepseek
```

### B) Unsloth Q4 + TurboQuant KV — horizontal MoE (best non-TQ3 daily gen)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 48 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk turbo4 -ctv turbo3
```

**Results:** pp4096 ≈ **483 t/s**, tg128 ≈ **43.5 t/s**.

Compare vertical-only:

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -p 4096 -n 128 \
  -ngl 20 -ncmoe 0 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk turbo4 -ctv turbo3
```

**Results:** tg128 ≈ **30.3 t/s** only.

Server (all experts CPU):

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -c 4096 -b 512 -ub 512 -t 12 \
  -ngl 999 --cpu-moe \
  -fa 1 -ctk turbo4 -ctv turbo3 \
  --host 0.0.0.0 --port 8080
```

Lab interactive note: ~**35 t/s**.

### C) Unsloth Q5 — ik_llama experts on CPU

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q5_K_M.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ot exps=CPU \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv q4_0
```

**Results (ik):** pp4096 ≈ **825 t/s**, tg128 ≈ **45.8 t/s**.

Same weights, mainline horizontal:

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q5_K_M.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

**Results:** pp4096 ≈ **501 t/s**, tg128 ≈ **48 t/s**.

### D) Unsloth Q6 — ik_llama

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q6_K.gguf \
  -p 4096 -n 128 \
  -ngl 99 --n-cpu-moe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

**Results:** pp4096 ≈ **807 t/s**, tg128 ≈ **46.8 t/s**.

Lab note: stock llama.cpp struggled with this Q6 at the time; **ik_llama worked**.

### E) MTP speculative (llama.cpp MTP branch)

Base command pattern (flags evolved across MTP PR / draft-mtp):

```bash
./build/bin/llama-cli \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf \
  -p "Create a robust Flutter state management solution using BLoC..." \
  -n 128 \
  -ngl 5 -ncmoe 32 \
  -fa 1 -t 12 \
  -ctk q8_0 -ctv q8_0 \
  --spec-type mtp --spec-draft-n-max 3
```

**Example memory breakdown (short run, RTX 3060):**

```text
CUDA0 (RTX 3060) | 11889 MiB total
  without MTP: model~3559 + context/compute … free ~6032; self ~4920
  with MTP n-max 3: free drops (draft overhead); gen ~30.8 t/s vs ~20.9 without
Host: ~30+ GiB self (model largely in system RAM)
```

**n-max sweep (ncmoe 32, ngl 5)** — gen t/s:

| | no mtp | n-max 1 | 2 | **3** | 4 | 5 | 6 |
|--|--------|---------|---|------|---|---|---|
| tg | 20.9 | 29.3 | 31.4 | **33.8** | 29.6 | 27.0 | 26.1 |
| VRAM MB | 5862 | 8023 | 8035 | 8040 | 8076 | 8106 | 8100 |

**Higher GPU expert budget (ncmoe 24, ngl 6):**

| | no mtp | n-max 2 | 3 | 4 |
|--|--------|---------|---|---|
| tg | 22.3 | **35.8** | 34.9 | 32.6 |
| VRAM MB | 8033 | 10511 | 10510 | 10594 |

Long server example (game / agent work):

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-MTP-UD-Q6_K_XL.gguf \
  -c 132768 \
  -ngl 99 --n-cpu-moe 36 \
  -fa 1 -t 12 --no-mmap \
  -ctk q8_0 -ctv q8_0 \
  --jinja --chat-template-kwargs '{"preserve_thinking":true}' \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --spec-stage mtp:n_max=4 --metrics
```

---

## Offload notes (this machine)

| Observation | Detail |
|-------------|--------|
| Q4–Q6 weight size | exceeds 12GB VRAM; host RAM used for experts |
| Vertical vs horizontal | see ngl-only vs ncmoe tables above |
| ncmoe values used often | ~48 (Q4 TurboQuant), ~16–32 (TQ3 / Q5 / Q6) |
| Highest Qwen tg in this lab | TQ3 fork + TQ3_4S weights |

## Related

- [../techniques/moe-offload.md](../techniques/moe-offload.md)
- [../techniques/mtp.md](../techniques/mtp.md)
- [../techniques/power-underclock.md](../techniques/power-underclock.md)
- [../runtimes/comparison.md](../runtimes/comparison.md)
