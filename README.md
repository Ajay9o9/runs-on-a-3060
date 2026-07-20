# Local LLM 3060

**Real local LLM (and image) results on an RTX 3060 12GB** — models, VRAM, system RAM offload, exact commands, and which `llama.cpp` forks we used.

> Not marketing synthetics. Lab notes cleaned from a working desktop: **3060 + Ryzen 9 9900X + 64 GB RAM**.

## Hardware (read this first)

| | |
|--|--|
| **GPU** | RTX **3060 12GB** (~11887 MiB) |
| **CPU** | Ryzen 9 **9900X** (12c/24t) |
| **RAM** | **64 GB** DDR5 @ 6000 MT/s |
| **OS** | Ubuntu 24.04 |

**Why RAM is in the title of every hybrid result:** Qwen3.6 35B Q4–Q6 weights are **20–30 GiB**. They do **not** fit in VRAM. Experts are offloaded to **system RAM**. If you have 16–32 GB RAM, do not expect the same hybrid speeds.

Full write-up: [HARDWARE.md](HARDWARE.md).

---

## What’s possible (cheat sheet)

### LLM — generation speed (approx, lab best)

| Model | Quant / path | Runtime | Offload | tg t/s | VRAM note |
|-------|--------------|---------|---------|-------:|-----------|
| Qwen3.6 35B-A3B | **TQ3_4S** | llama.cpp-**tq3** | ngl 99, ncmoe 16–32 | **~54–60** | hybrid; RAM holds experts |
| Qwen3.6 35B-A3B | Q5 / Q6 Unsloth | ik_llama / mainline | exps CPU / ncmoe 32 | **~46–48** | ~25–30 GiB weights in RAM |
| Qwen3.6 35B-A3B | Q4_K_XL TurboQuant | TQ branch | **horizontal** ncmoe 48 | **~43** | vs ~30 vertical |
| Qwen3.6 35B MTP | MTP Q6 | MTP branch | n-max 2–3 | **~31–36** | +~2–2.5 GB vs no MTP |
| Gemma 4 12B | Q5_K_XL | llama.cpp | full GPU | **~30–33** | fits 12GB |
| Gemma 4 12B | Q6_K_XL | llama.cpp | full GPU | **~26** | peak ~11.3 GB |
| Gemma 4 26B-A4B | Q6 | llama.cpp | ncmoe 20–32 | **~33–39** | 3.8–10.1 GB VRAM knob |
| Ternary Bonsai **27B LLM** | Q2_0 | llama-server | ngl 999 | **~23–25** | long PP 24k–49k |
| Diffusion Gemma 26B | Q4 (text diffusion) | diffusion-cli | ngl 15 | ~0.5 s/step | peak **10450 MB** |

### Image

| Model | Stack | Time (lab) | Peak VRAM |
|-------|-------|------------|-----------|
| **Bonsai Image 4B** ternary | PrismML studio / gemlite | **~9.7 s** / ~2.4 s/step | **~6585 MB** |

---

## Repo map

```text
local-llm-3060/
├── README.md                 ← you are here
├── HARDWARE.md               ← GPU + CPU + RAM + offload implications
├── models/                   ← LLM section (commands + results)
│   ├── qwen3.6-35b-a3b.md    ← deepest page
│   ├── gemma4-12b.md
│   ├── gemma4-26b-a4b.md
│   ├── diffusion-gemma.md    ← text diffusion, not image
│   └── bonsai-ternary-27b.md ← LLM, not image
├── image/                    ← image section
│   └── bonsai-image-4b.md
├── runtimes/                 ← which llama.cpp forks + side-by-side
│   ├── README.md
│   └── comparison.md
├── techniques/
│   ├── moe-offload.md        ← vertical vs horizontal
│   ├── mtp.md
│   └── power-underclock.md
└── recipes/                  ← short copy-paste starters
```

### Start here by goal

| I want… | Go to |
|---------|--------|
| Reproduce a command | [models/](models/) or [runtimes/comparison.md](runtimes/comparison.md) |
| Understand offload / RAM | [techniques/moe-offload.md](techniques/moe-offload.md) + [HARDWARE.md](HARDWARE.md) |
| Pick a fork | [runtimes/README.md](runtimes/README.md) |
| Image gen | [image/bonsai-image-4b.md](image/bonsai-image-4b.md) |
| Power limit tip | [techniques/power-underclock.md](techniques/power-underclock.md) |

---

## Runtimes tried

| Runtime | Why |
|---------|-----|
| **llama.cpp** (mainline) | Gemma, general GGUF, server |
| **ik_llama.cpp** | Strong Qwen Q5/Q6 hybrid |
| **llama.cpp-tq3** | TQ3_4S — fastest Qwen gen here |
| **TurboQuant branch** | turbo4/turbo3 KV on Q4 |
| **MTP / draft-mtp branches** | Speculative decoding |
| **llama-diffusion-gemma** | Diffusion text model |
| **llama-benchy** | Server-side long-context benches |

Details + full commands: [runtimes/comparison.md](runtimes/comparison.md).

---

## Command conventions

```bash
export MODEL_DIR=/path/to/ggufs
# run from the build dir of the runtime you chose
./build/bin/llama-bench ...
./build/bin/llama-server ...
```

Every serious result page tries to list:

1. Runtime  
2. Full flags (`-ngl`, `-ncmoe`, `-t`, KV types)  
3. pp t/s + tg t/s  
4. VRAM when known  
5. Reminder that hybrid MoE used **64 GB** host RAM  

---

## Name collisions (please don’t mix these)

| Name | What it is |
|------|------------|
| **Ternary Bonsai 27B** | Text **LLM** (GGUF) |
| **Bonsai Image 4B ternary** | **Image** gen (PrismML) |
| **Diffusion Gemma** | **Text** diffusion LLM |

---

## Contributing

Have a 3060 (or 12GB class) result? See [CONTRIBUTING.md](CONTRIBUTING.md). Always include **GPU + CPU + system RAM** and the **exact command**.

## License

Documentation: [CC-BY-4.0](LICENSE). Commands and numbers free to reuse with attribution.
