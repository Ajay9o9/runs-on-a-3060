# Runtime comparison — Qwen3.6 35B-A3B on RTX 3060 12GB

**Host:** Ryzen 9 9900X · **64 GB** RAM · Ubuntu 24.04  
**GPU:** RTX 3060 ~11887 MiB  

Same class of model (Qwen3.6 MoE 35B-A3B), different runtimes / quants / offload. Commands sanitized.

---

## 1) ik_llama.cpp — Unsloth Q5, experts on CPU

**Runtime:** [ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp)  
**Model:** `Qwen3.6-35B-A3B-UD-Q5_K_M.gguf` (~24.63 GiB weights)  
**Offload:** full layer count on GPU path + **all experts on CPU** (`-ot exps=CPU`)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q5_K_M.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ot exps=CPU \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv q4_0
```

| test | t/s |
|------|-----|
| pp4096 | **825.39 ± 4.79** |
| tg128 | **45.75 ± 0.07** |

> High PP because experts on CPU free the GPU for attention-heavy prefill in this fork’s path. Needs **lots of system RAM** (weights ~25 GiB on host).

---

## 2) Mainline-style llama.cpp — same Q5, horizontal MoE

**Runtime:** llama.cpp (mainline-style flags)  
**Model:** same Q5_K_M  
**Offload:** `-ngl 99 -ncmoe 32` (attention on GPU; 32 expert layers forced to CPU)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q5_K_M.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| test | t/s |
|------|-----|
| pp4096 | **501.23 ± 2.49** |
| tg128 | **47.93 ± 0.30** |

**Long context (128k prefill test):**

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q5_K_M.gguf \
  -p 131072 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 4094 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| test | t/s |
|------|-----|
| pp131072 | **425.35 ± 0.72** |
| tg128 | **47.98 ± 0.18** |

---

## 3) llama.cpp-tq3 — TQ3_4S weights (fastest gen in this lab)

**Runtime:** [llama.cpp-tq3](https://github.com/turbo-tan/llama.cpp-tq3)  
**Model:** `Qwen3.6-35B-A3B-TQ3_4S.gguf` (~12.38 GiB, ~4 bpw turbo four-scale)  
**HF:** [YTan2000/Qwen3.6-35B-A3B-TQ3_4S](https://huggingface.co/YTan2000/Qwen3.6-35B-A3B-TQ3_4S)  
**Offload:** `-ngl 99 -ncmoe 32` (or lower ncmoe for more experts on GPU)

### 4k context

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv tq3_0
```

| run | pp4096 | tg128 |
|-----|--------|-------|
| 1 | 592.43 ± 32.46 | **52.71 ± 3.99** |
| 2 | 618.79 ± 9.09 | **54.35 ± 1.12** |

### 128k context

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -p 131072 -n 128 \
  -ngl 99 -ncmoe 24,16 \
  -b 4096 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv tq3_0
```

| n_cpu_moe | pp131072 | tg128 |
|-----------|----------|-------|
| 24 | 501.32 ± 0.81 | **58.40 ± 0.29** |
| 16 | 531.64 ± 3.78 | **59.73 ± 0.14** |

### Power-limited (115 W, core 1550 MHz), ncmoe 20

```bash
sudo nvidia-smi -pl 115
sudo nvidia-smi -lgc 1550,1550

./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-TQ3_4S.gguf \
  -p 131072 -n 128 \
  -ngl 99 -ncmoe 20 \
  -b 4096 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv tq3_0
```

| state | pp131072 | tg128 | notes |
|-------|----------|-------|-------|
| stock (~142 W peak) | 507.64 ± 0.91 | **59.21 ± 0.66** | |
| 115 W / 1550 MHz | 477.90 ± 5.99 | **53.76 ± 0.76** | ~9% gen drop |

---

## 4) TurboQuant KV (TheTom-style) — Unsloth Q4, vertical vs horizontal

**Runtime:** TurboQuant-enabled llama.cpp branch  
**Model:** `Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` (~20.81 GiB)  
**KV:** `-ctk turbo4 -ctv turbo3`

### Vertical split (layer offload only)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -p 512 -n 128 \
  -ngl 16,18,20,22,24 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk turbo4 -ctv turbo3
```

| ngl | pp512 | tg128 |
|-----|-------|-------|
| 16 | 539.17 | 27.42 |
| 18 | 560.04 | 28.76 |
| 20 | 582.58 | **30.12** |

**4k vertical baseline (`-ngl 20`, no ncmoe):**

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -p 4096 -n 128 \
  -ngl 20 -ncmoe 0 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk turbo4 -ctv turbo3
```

| test | t/s |
|------|-----|
| pp4096 | 562.83 ± 3.24 |
| tg128 | **30.27 ± 0.29** |

### Horizontal split (attention on GPU, experts on CPU)

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ncmoe 64,56,48,40 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk turbo4 -ctv turbo3
```

| n_cpu_moe | pp4096 | tg128 | note |
|-----------|--------|-------|------|
| 64 | 481.90 | **42.96** | all experts CPU |
| 56 | 475.90 | 43.33 | |
| **48** | 483.19 | **43.45** | sweet spot in this sweep |
| 40 | 483.85 | 42.79 | slight regression (oversaturate / PCIe) |

**Takeaway:** horizontal MoE offload was **~43 t/s** vs vertical **~30 t/s** on the same Q4 weights — biggest free lunch on this 3060 + 64GB box.

### Server recipes (TurboQuant Q4)

Daily-driver style:

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -c 4096 -b 512 -ub 512 -t 12 \
  -ngl 999 --cpu-moe \
  -fa 1 -ctk turbo4 -ctv turbo3 \
  --host 0.0.0.0 --port 8080
```

Lab note: ~**35 t/s** interactive in early server runs with `--cpu-moe`.

Longer context:

```bash
./build/bin/llama-server \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -c 32768 -b 512 -ub 512 -t 12 \
  -ngl 99 -ncmoe 64 \
  -fa 1 -ctk turbo4 -ctv turbo3 \
  --host 0.0.0.0 --port 8080
```

---

## 5) ik_llama.cpp — Unsloth Q6

**Model:** `Qwen3.6-35B-A3B-UD-Q6_K.gguf` (~27.29 GiB)  
**Note:** Lab observed Q6 more reliable on **ik_llama** than on stock llama.cpp at the time of testing.

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q6_K.gguf \
  -p 4096 -n 128 \
  -ngl 99 -ot exps=CPU \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q4_0 -ctv q4_0
```

| test | t/s |
|------|-----|
| pp4096 | 721.19 ± 61.11 |
| tg128 | **43.27 ± 0.13** |

With `-n-cpu-moe` instead of tensor override:

```bash
./build/bin/llama-bench \
  -m $MODEL_DIR/Qwen3.6-35B-A3B-UD-Q6_K.gguf \
  -p 4096 -n 128 \
  -ngl 99 --n-cpu-moe 32 \
  -b 512 -ub 512 -t 12 \
  -fa 1 -ctk q8_0 -ctv q8_0
```

| n_cpu_moe | pp4096 | tg128 |
|-----------|--------|-------|
| 34 | 799.31 ± 10.14 | 45.32 ± 0.49 |
| **32** | 807.45 ± 12.82 | **46.76 ± 0.10** |

---

## Summary table (gen speed, same GPU)

| Runtime | Quant | Offload idea | Best tg (approx) |
|---------|-------|--------------|------------------|
| llama.cpp-tq3 | TQ3_4S | hybrid MoE | **~55–60 t/s** |
| ik_llama | Q5 / Q6 | exps CPU / ncmoe | **~45–48 t/s** |
| mainline llama.cpp | Q5 | ncmoe 32 | **~48 t/s** |
| TurboQuant branch | Q4_K_XL | horizontal ncmoe 48 | **~43 t/s** |
| TurboQuant branch | Q4_K_XL | vertical ngl 20 | **~30 t/s** |

**System RAM is part of the result.** Treat hybrid rows as “3060 + strong CPU + **64 GB RAM**,” not “3060 alone.”
